# b8kery lab build PART 2
Following the steps in the blog below to complete second phase (k3s config) here:

https://thenewstack.io/tutorial-install-a-highly-available-k3s-cluster-at-the-edge/

NOTES:
* This is an *experimental* HA config since etcd is considered "experimental" right now on arm64.
* If you need to uninstall/reinstall k3s then you must blow away k8s data written into the etcd db (because etcd is external in this config and from experience I found a very confused k3s env starting up if I didn't blow away etcd data before running the k3s installer again). To reset/restart etcd (after running k3s-uninstall.sh), on each node do this:
  `  ubuntu@blueberry:~$ sudo systemctl stop etcd`
  `  ubuntu@blueberry:~$ sudo sudo rm -rf /var/lib/etcd/*`
  `  ubuntu@blueberry:~$ sudo systemctl start etcd`
  `  ubuntu@blueberry:~$ etcdctl --endpoints https://192.168.100.1:2379 --cert /etc/etcd/server.crt --cacert /etc/etcd/etcd-ca.crt --key /etc/etcd/server.key member list`



### Add `cgroup_memory=1 cgroup_enable=memory` to /boot/firmware/nobtcmd.txt
K3s startup failed until I troubleshooted and found that for my install of Ubuntu 18.04 on RPi4b/arm64 nodes you need to first add the last two parameters to nobtcmd.txt:
```bash
> cat /boot/firmware/nobtcmd.txt
net.ifnames=0 dwc_otg.lpm_enable=0 console=ttyAMA0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc cgroup_memory=1 cgroup_enable=memory
```

Reboot, check the health of your fresh etcd cluster post-reboot, and proceed.

### Install k3s

I'm running a 3-node cluster (no extra "agent" node per the source blog above):

Host | IP | K3s role
---- | -- | --------
blueberry.almond.lan | 192.658.100.16 | etc,server
blackberry.almond.lan | 192.658.100.137 | etc,server
strawberry.almond.lan | 192.658.100.168 | etc,server

Exported the following on each cluster node:
```bash
export K3S_DATASTORE_ENDPOINT='https://192.168.100.16:2379,https://192.168.100.137:2379,https://192.168.100.168:2379'
export K3S_DATASTORE_CAFILE='/etc/etcd/etcd-ca.crt'
export K3S_DATASTORE_CERTFILE='/etc/etcd/server.crt'
export K3S_DATASTORE_KEYFILE='/etc/etcd/server.key'
```

*No need to export K3S_TOKEN since I don't have more "agent" nodes to add at this time.*

Ran the k3s install on each node:

`curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=servicelb --disable=traefik" sh -s - --write-kubeconfig-mode 644`

After a few seconds check status of k3s on one of the cluster nodes:
```bash
ubuntu@blueberry:~$ export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
ubuntu@blueberry:~$ kubectl get nodes
NAME                      STATUS   ROLES    AGE   VERSION
blackberry.almond.local   Ready    master   4m16s   v1.18.8+k3s1
blueberry.almond.local    Ready    master   4m14s   v1.18.8+k3s1
strawberry.almond.local   Ready    master   4m14s   v1.18.8+k3s1
```

```bash
ubuntu@blueberry:~$ sudo systemctl status k3s.service
● k3s.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-09-24 06:09:39 UTC; 4min 49s ago
     Docs: https://k3s.io
  Process: 13239 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
  Process: 13238 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
 Main PID: 13240 (k3s-server)
    Tasks: 45
   CGroup: /system.slice/k3s.service...
```

If you have worker nodes to add follow the additional steps to install the K3s agent per the newstack.io source blog.
Also - good idea to run the steps to check proper usage of the external etcd service included in the source blog.
