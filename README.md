# Google OIDC for Kubernetes Dashboard & Kubectl

This will show you how to setup your Kubernetes cluster so that you can access it from the dashboard and `kubectl` with a Google OIDC token.

Most of the document assumes that you're using kops on AWS, but you should be able to extract the relevant information to run this anywhere.

## Versions

- kubectl: 1.8.6
- kubernetes: 1.8.6 **w/ RBAC enabled**
- kops: 1.8

The RBAC part is important.

## Tools

We're using a few tools I found around the web to set this all up easily.

- [kubernetes/ingress-nginx](https://github.com/kubernetes/ingress-nginx) (for exposing the dashboard)
- [myob/openresty-oidc](https://github.com/MYOB-Technology/docker-images/tree/master/openresty-oidc-adfs) (for dashboard auth)
- [negz/kuberos](https://github.com/negz/kuberos) (for kubectl auth)
- [neilpang/acme.sh](https://github.com/Neilpang/acme.sh) (for generating LE certs for the exposed dashboard)
- [tazjin/kontemplate](https://github.com/tazjin/kontemplate) (for simple manifest templating)

Just letting you know beforehand..

## Steps

### 1. Setup a OIDC application in Google

a) https://console.cloud.google.com/apis/credentials

b) Create credentials > OAuth client ID > Web Application

c) Name it Kubernetes and add the callbacks URLS
`https://kuber.example/.org/oauth2/callback`
`https://kuberos.example/.org/ui`

### 2. Configure the Kubernetes API Server

We need to configure the API Server with OIDC client information so that it can validate claims with Google.

If you're using kops, you can do a `kops edit cluster` and add the following.

```
kubeAPIServer:
  oidcClientID: REDACTED.apps.googleusercontent.com
  oidcIssuerURL: "https://accounts.google.com"
  oidcUsernameClaim: email
```

And then do the usual `kops update` and `kops rolling-update` to update your cluster. This will take your master node down for a couple minutes.

### 3. Create TLS Certs using LetsEncrypt

Now, we'll create TLS certs for the subdomain that our kubernetes dashboard will be exposed (but authenticated).

```
./acme.sh --issue -d "kuber.example.org" --dns dns_aws --keylength ec-256
```

### 4. Create Nginx Ingress

Next, we need to create an nginx-ingress so that we can serve our OIDC proxy and access our dashboard from a a domain. This part will just setup the base

````
# Basics
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml

# RBAC
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml

# AWS ELB
kubectl patch deployment -n ingress-nginx nginx-ingress-controller --type='json' \
  --patch="$(curl https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/publish-service-patch.yaml)"
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/aws/service-l4.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/aws/patch-configmap-l4.yaml
````
### 5. Setup your route53 for new ELB

Kops should have setup everything you needed as far as Kubernetes API server records, but we need to create an additional record that will wildcard match all subdomains to route them through our Nginx Ingress. If you would like, you can use kuber.example.org instead of a wildcard to limit scope.

![r53](assets/r53-aws-nginx-ingress.png)

### 6. Deploy OIDC Proxy & Dashboard

Here comes the fun part! We're going to deploy a OIDC proxy that will authenticate our users against Google and then redirect them to the Kubernetes dashboard with a header that authenticates them (`Authorization : Bearer xxxx`).

Well now need to configure the file `cluster.yaml` with the appropriate values so that we can template our Kubernetes manifests. Some TLS certs for your ingress, some OIDC configuration for the OIDC proxy, and a developer/admin list for access control to cluster resources. Please feel free to tweak this last one to your liking at `oidc-proxy-dashboard/rbac.yaml`.

Now we need to create a secret with your OIDC information:

```
kubectl create secret \
  -n kube-system \
  generic \
  kube-dashboard-secrets \
  --from-literal=client_id=REDACTED.apps.googleusercontent.com  \
  --from-literal=client_secret=REDACTED \
  --from-literal=session=enGCuITaBPHQtpZSxhcivw==
```

Then create the TLS certs

```
kubectl create secret tls kuberos-tls-secret \
  --key '/Users/noqcks/.acme.sh/*.example.org_ecc/*.example.org.key' \
  --cert '/Users/noqcks/.acme.sh/*.example.org_ecc/*.example.org.cer' \
  -n kube-system
```

Now back to the manifests, you can use kontemplate to see the output in YAML.

```
kontemplate template cluster.yaml -i oidc-proxy-dashboard
```

Once it's to your liking, you can deploy to your cluster.

```
kontemplate apply cluster.yaml -i oidc-proxy-dashboard
```

You should now be able to access your Kubernetes dashboard at `https://kuber.example.org`! If you've run into issues, file an issue above.

### 7. Setup Proxy for Kubectl

So, we've setup our Kubernetes dashboard, but unfortunately our developers have no easy way to authenticate with the Kubernetes cluster via `kubectl`.

We need to create a secret with your OIDC client secret to be used by kuberos.

```
kubectl create secret \
  -n kube-system \
  generic \
  kuberos-secret \
  --from-literal=secret={{ OIDC client secret here }}
```

Then we can deploy Kuberos to Kubernetes.

```
kontemplate apply cluster.yaml -i kuberos
```

The service should be running at `https://kuberos.example.org`. If we access it and login to our
google account, it will generate us a id_token to use for authenticating with our

```
kubectl config set-credentials "500IQDevOpsGenius@example.org" \
  --auth-provider=oidc \
  --auth-provider-arg=client-id="REDACTED.apps.googleusercontent.com" \
  --auth-provider-arg=client-secret="REDACTED" \
  --auth-provider-arg=id-token="REDACTED" \
  --auth-provider-arg=refresh-token="REDACTED" \
  --auth-provider-arg=idp-issuer-url="https://accounts.google.com"
```

![user-restricted](assets/restricted-kubernetes-user-access.png)