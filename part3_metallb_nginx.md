# b8kery lab build PART 3: Metallb, nginx

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

(migrated from stable/nginx-ingress to nginx-ingress/nginx-ingress 21 Oct 20, ref: https://github.com/kubernetes/ingress-nginx/tree/master/charts/ingress-nginx ...)

Used same cmd examples provided in the kauri.io reference:

```bash
❯ helm install ingress-nginx ingress-nginx/ingress-nginx
NAME: ingress-nginx
LAST DEPLOYED: Wed Oct 21 16:17:11 2020
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The ingress-nginx controller has been installed.
It may take a few minutes for the LoadBalancer IP to be available.
You can watch the status by running 'kubectl --namespace kube-system get services -o wide -w ingress-nginx-controller'

An example Ingress that makes use of the controller:

  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: example
    namespace: foo
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: exampleService
                servicePort: 80
              path: /
    # This section is only required if TLS is to be enabled for the Ingress
    tls:
        - hosts:
            - www.example.com
          secretName: example-tls

If TLS is enabled for the Ingress, a Secret containing the certificate and key must also be provided:

  apiVersion: v1
  kind: Secret
  metadata:
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
❯ kubectl --namespace kube-system get services -o wide -w ingress-nginx-controller
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                      AGE   SELECTOR
ingress-nginx-controller   LoadBalancer   10.43.21.124   192.168.200.1   80:30819/TCP,443:31115/TCP   39s   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
```
