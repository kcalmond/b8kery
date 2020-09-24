# b8kery lab build PART 3: Metallb, certmanager, ...

* Configured local client (Macbook) with latest version of helm.  
* Installed/configured metallb, nginx-ingress, certmanager &* setup certmanager issuers.

References:
* Roughly following steps in parts of this HowTo: https://kauri.io/38-install-and-configure-a-kubernetes-cluster-with/418b3bc1e0544fbc955a4bbba6fff8a9/a
* https://opensource.com/article/20/7/homelab-metallb

### metallb
Installation by manifest using examples here: metallb.universe.tf/installation

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Applied a configmap specifying first half of 192.168.200.1/24. My homelab network is 192.168.0.1/16. This config based on sample provided in the opensource.com reference.

```bash
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: address-pool-1
      protocol: layer2
      addresses:
      - 192.168.200.1-192.168.200.127
EOF
```

### nginx-ingress
Used same cmd examples provided in the kauri.io reference:

```bash
$ helm install nginx-ingress stable/nginx-ingress --namespace kube-system \
    --set defaultBackend.enabled=false

     - check on status -

$ kubectl get pods -n kube-system -l app=nginx-ingress -o wide
$ kubectl get services  -n kube-system -l app=nginx-ingress -o wide
```
