# Running k3s cluster on Raspberry Pi 4B/4gb nodes consuming "external" etcd for k8s HA

The build process is based on an adaptation of the two excellent HowTos posted here:

1. https://thenewstack.io/tutorial-set-up-a-secure-and-highly-available-etcd-cluster/
2. https://thenewstack.io/tutorial-install-a-highly-available-k3s-cluster-at-the-edge/
3. (for alt part 2...) https://kauri.io/38-install-and-configure-a-kubernetes-cluster-with/418b3bc1e0544fbc955a4bbba6fff8a9/a

Read #2 to understand the target architecture for etcd and k3s.

**Key Differences** between this config and thenewstack.io blog:
* Using a RaspPi4b based cluster, which is ARM64 based. **The etcd implementation on arm64 is still unsupport/experimental status as of this writing.** ref: https://github.com/etcd-io/etcd/issues/9077
* 3 nodes instead of 4 - no need to run k3s agent setup part

Annotations/fixes to the process are included.

* **[part1_OS_etcd](https://github.com/kcalmond/b8kery/blob/master/part1_OS_etcd.md)**
* **[part2_k3s](https://github.com/kcalmond/b8kery/blob/master/part2_k3s.md)**
* **[part2_metallb_certmgr_nginx](https://github.com/kcalmond/b8kery/blob/master/part2_metallb_certmgr_nginx.md)**


## TBDoc - stuff completed that needs to be retro-doc'd into this repo:
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
