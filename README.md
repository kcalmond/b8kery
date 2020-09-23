# b8kery lab build

The build process is based on an adaptation of the two excellent HowTos posted here:

1. https://thenewstack.io/tutorial-set-up-a-secure-and-highly-available-etcd-cluster/
2. https://thenewstack.io/tutorial-install-a-highly-available-k3s-cluster-at-the-edge/

**Key Differences** between this config and thenewstack.io blog:
* Using a RaspPi4b based cluster - ARM64 - *therefore the etcd implementation is experimental status°
* 3 nodes instead of 4 - no need to run k3s agent setup part

Annotations/fixes to the process are included.

* **[Part 1](https://github.com/kcalmond/b8kery/blob/master/part1_OS_etcd.md)**
* **[Part 2](https://github.com/kcalmond/b8kery/blob/master/part2_k3s.md)**
