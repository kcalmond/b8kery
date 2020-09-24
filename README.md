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
