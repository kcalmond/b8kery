# RaspberryPi b8kery

## k3s version here:
**[README_k3s](https://github.com/kcalmond/b8kery/blob/master/k3s/README_k3s.md)**

___

## Raspbernetes version here:
**[README_raspbernetes](https://github.com/kcalmond/b8kery/blob/master/raspbernetes/README_raspbernetes.md)**


___
## TBD - stuff completed that needs to be retro-doc'd into this repo:
* static & dynamic PV setup. Ref guidelines:
  * static first part of this: https://medium.com/@myte/kubernetes-nfs-and-dynamic-nfs-provisioning-97e2afb8b4a9
  * dynamic: https://opensource.com/article/20/6/kubernetes-nfs-client-provisioning

* Monitoring Setup:
  * Refs:
    * https://github.com/carlosedp/cluster-monitoring
    * https://kauri.io/deploy-prometheus-and-grafana-to-monitor-a-kube/186a71b189864b9ebc4ef7c8a9f0a6b5/a
    * https://opensource.com/article/20/7/homelab-metallb  <--for LB setup examples
      * modified cluster-monitoring/manifests/grafana-service.yaml for using metallb provided external l4 LB:
      ```
      rancher@hackberry:~/GH/cluster-monitoring/manifests> cat grafana-service.yaml
      apiVersion: v1
      kind: Service
      metadata:
        labels:
          app: grafana
        name: grafana
        namespace: monitoring
      spec:
        ports:
        - name: http
          protocol: TCP
          port: 80
          targetPort: 3000
        selector:
          app: grafana
        type: LoadBalancer
        ```
  * TBD
    * setup persistence for Prometheus & Grafana: https://github.com/carlosedp/cluster-monitoring
    * setup TLS
      * https://cert-manager.io/next-docs/tutorials/acme/ingress/
      * re using default self-signed cert section here: https://github.com/carlosedp/cluster-monitoring


* Clean up
(DONE...)
```
  * do a delete on these:
    ```
    kubectl apply -f manifests/ingress-alertmanager.yaml
    kubectl apply -f manifests/ingress-prometheus.yaml
    kubectl apply -f manifests/ingress-grafana.yaml
    kubectl apply -f ./grafana-service.yaml
    ```
  * Do upgrade process here to migration from stable/nginx-ingress to nginx-ingress/nginx-ingress
    * helm uninstall (check that svc grabbing 200.1 IP also deleted...?
    * helm install of nginx-ingress/nginx-ingress
      * use form of install with defaultBackend.enabled=true
      * check ip allocation for default svc
    * redo monitoring yamls above
```
