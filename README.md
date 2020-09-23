# b8kery lab build notes
Following these blog guidelines to build a pi4 based *HA* k3s cluster for lab workloads:
1. https://thenewstack.io/tutorial-install-a-highly-available-k3s-cluster-at-the-edge/
1. https://thenewstack.io/tutorial-set-up-a-secure-and-highly-available-etcd-cluster/

## ClusterOS Setup
* used "ubuntu-18.04.5-preinstalled-server-arm64+raspi4.img.xz" image available here: https://cdimage.ubuntu.com/releases/18.04/release/
* nodes:
  * blueberry.almond.local 192.168.100.16
  * blackberry.almond.local 192.168.100.137
  * strawberry.almond.local 192.168.100.168

## ETCD Setup
*
