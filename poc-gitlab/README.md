# Proof of concept gitlab

K8s cluster is installed using *[k0sctl](https://github.com/k0sproject/k0sctl)*

## Instruction

### MetalLB

1. Deploy MetalLB

```shell
$ alias k='kubectl'
$ k apply -f manifests/metallb/namespace.yaml
$ k apply -f manifests/metallb/metallb.yaml
```

2. Wait for pods to be in Running state
3. Edit *manifests/metallb/metallb-configmap.yaml* according to your needs.
   See [example](https://metallb.org/usage/example/)
4. Apply config

```shell
$ k apply -f manifests/metallb/metallb-configmap.yaml
```

### Preparations

1. Create gitlab namespace

```shell
$ k apply -f manifests/namespace.yaml
```

2. If you haven't deployed k8s with *k0s-config.yaml* then on every worker node prepare volumes

```shell
$ rm -r /mnt/data/*
$ mkdir -p /mnt/data/postgres-pv
$ mkdir -p /mnt/data/minio-pv
$ mkdir -p /mnt/data/prometheus-pv
$ mkdir -p /mnt/data/redis-pv
$ mkdir -p /mnt/data/repository-pv
$ chmod 777 /mnt/data/*
```

NOTE: Do that every time you are redeploying gitlab to remove garbage

4. Add persistent volumes

```shell
$ k apply -f manifests/volumes/
```

### Install gitlab using helm

1. Add gitlab repo

```shell
$ helm repo add gitlab https://charts.gitlab.io/
$ helm repo update
```

2. Use *--kubeconfig* to point to your config. Choose your version with *--version*,
   see [version mappings](https://docs.gitlab.com/charts/installation/version_mappings.html)

```shell
$ helm upgrade --install gitlab gitlab/gitlab \
	--kubeconfig=~/.kube/config \
	--set global.hosts.domain=<yourdomain.com> \
	--set global.hosts.externalIP=<firewall-ip> \
	--set certmanager-issuer.email=<your-email> \
	--version=5.8.2 -n gitlab
```

3. Wait for installation to complete
4. Add LoadBalancer

```shell
$ k apply -f manifests/net/ingress-lb.yaml
```

5. LoadBalancer should get and *external-ip*

```shell
$ k get svc -n gitlab
```

6. Configure your firewall to access *nodeports* that LoadBalancer got. It should be like:

```
<firewall-ip>:80  -> <node-ip>:<http-nodeport>
<firewall-ip>:443 -> <node-ip>:<https-nodeport>
<firewall-ip>:22  -> <node-ip>:<ssh-nodeport>
```

7. You should be able to access your gitlab with your domain. Get *root* password

```shell
$ k get secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}' -n gitlab | base64 --decode ; echo
```

### Adding SSO to Gitlab

Links: \
[Preparation](https://docs.gitlab.com/ee/integration/omniauth.html) \
[Adding omniauth to your release](https://docs.gitlab.com/charts/charts/globals#omniauth)

1. Get values

```shell
$ helm show values gitlab/gitlab > values.yaml
```

2. Change omniauth config in *values.yaml*. Config example:

```yaml
 omniauth:
   enabled: true # Default: false
   autoSignInWithProvider:
   syncProfileFromProvider: [ ]
   syncProfileAttributes: [ 'email' ]
   allowSingleSignOn: [ 'saml', 'google_oauth2' ] # Default: ['saml']
   blockAutoCreatedUsers: false # Default: true
   autoLinkLdapUser: true # Default: false
   autoLinkSamlUser: false
   autoLinkUser: [ ]
   externalProviders: [ ]
   allowBypassTwoFactor: [ ]
   providers: # Default: []
     - secret: gitlab-google-oauth2
```

3. Add *gitlab-google-oauth2* secret,
   see [example](https://github.com/borodun/metal-gitlab/blob/main/poc-gitlab/manifests/google-provider-secret.yaml)

```shell
$ k create secret generic -n gitlab gitlab-google-oauth2 --from-file=provider=manifests/google-provider-secret.yaml  
```

4. Apply changes

```shell
$ helm upgrade --install gitlab gitlab/gitlab \
	--kubeconfig=~/.kube/config \
	--set global.hosts.domain=<yourdomain.com> \
	--set global.hosts.externalIP=<firewall-ip> \
	--set certmanager-issuer.email=<your-email> \
	-f values.yaml \
	--version=5.8.2 -n gitlab
```