# b8kery lab build

**Using a 3-node pi4B (4GB) cluster to configure a "HA" k3s environment.** *HA* in quotes because running etcd on arm64 is still considered experimental.

The build process is based on the two excellent blogs posted here:

1. https://thenewstack.io/tutorial-set-up-a-secure-and-highly-available-etcd-cluster/
2. https://thenewstack.io/tutorial-install-a-highly-available-k3s-cluster-at-the-edge/

Annotations/fixes to the process are included.

* **[Part 1](https://github.com/kcalmond/b8kery/blob/master/part1_OS_etcd.md)**
* **[Part 2](https://github.com/kcalmond/b8kery/blob/master/part2_k3s.md)**

**Key Differences** between this config and thenewstack.io blog:
* Using a RaspPi4b based cluster - ARM64 - therefore the etcd implementation is experimental status
* 3 nodes instead of 4 - no need to run k3s agent setup part
