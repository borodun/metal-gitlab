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

2. Wait for pods to be in Ready state
3. Edit *metallb-configmap.yaml* according to your needs. See [example](https://metallb.org/usage/example/)

```shell
$ k apply -f manifests/metallb/metallb-configmap.yaml
```

### Preparations

1. Create gitlab namespace

```shell
$ k apply -f manifests/namespace.yaml
```

2. Add persistent volumes

```shell
$ k apply -f manifests/volumes/
```

## Install gitlab using helm

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