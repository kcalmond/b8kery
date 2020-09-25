# b8kery lab build PART 3: Metallb, certmanager, ...

* Configured local client (Macbook) with latest version of helm.  
* Installed/configured metallb, nginx-ingress, certmanager &* setup certmanager issuers.

References:
* Roughly followed steps here: https://kauri.io/38-install-and-configure-a-kubernetes-cluster-with/418b3bc1e0544fbc955a4bbba6fff8a9/a
* cherry picked some guidance from here: https://opensource.com/article/20/7/homelab-metallb

### metallb
Installation by manifest using examples here: metallb.universe.tf/installation

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Applied a configmap specifying first half of 192.168.200.1/24. My homelab network is 192.168.0.1/16. This config based on sample provided in the opensource.com reference. Deploying using layer2 mode. Details: https://metallb.universe.tf/concepts/layer2/

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
$ helm install nginx-ingress stable/nginx-ingress --namespace kube-system --set defaultBackend.enabled=false

Checked deployment status:

$ kubectl get pods -n kube-system -l app=nginx-ingress -o wide

$ kubectl get services  -n kube-system -l app=nginx-ingress -o wide
NAME                       TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)                      AGE
nginx-ingress-controller   LoadBalancer   10.43.87.32   192.168.200.1   80:30351/TCP,443:30783/TCP   58s
```

###cert-manager
Used helm chart to install. Using commands from kauri.io reference, updated to v1.0.2:

```bash
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.0.2/cert-manager.crds.yaml
$ helm repo add jetstack https://charts.jetstack.io && helm repo update
$ helm install cert-manager jetstack/cert-manager --namespace kube-system  --version v1.0.2
$ kubectl get pods -n kube-system -l app.kubernetes.io/instance=cert-manager -o wide

$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: <EMAIL>
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

$ cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: <EMAIL>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
EOF
```
