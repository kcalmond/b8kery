# b8kery lab build PART 2
Following the steps in the blog below to complete second phase (k3s config) here:

https://thenewstack.io/tutorial-install-a-highly-available-k3s-cluster-at-the-edge/

NOTE: this is an *experimental* HA config since etcd is considered "experimental" right now on arm64.

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

(no need to export K3S_TOKEN since I don't have more "agent" nodes to add...)

Ran the k3s install on each node:
`curl -sfL https://get.k3s.io | sh -`

After a few seconds:
```bash
ubuntu@blueberry:~$ sudo chown ubuntu:ubuntu /etc/rancher/k3s/k3s.yaml
ubuntu@blueberry:~$ export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
ubuntu@blueberry:~$ kubectl get nodes
NAME                      STATUS   ROLES    AGE   VERSION
blackberry.almond.local   Ready    master   17h   v1.18.8+k3s1
blueberry.almond.local    Ready    master   17h   v1.18.8+k3s1
strawberry.almond.local   Ready    master   17h   v1.18.8+k3s1
```

```bash
ubuntu@blueberry:~$ sudo systemctl status k3s.service
● k3s.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enabled)
   Active: active (running) since Wed 2020-09-23 06:45:14 UTC; 16m ago
     Docs: https://k3s.io
  Process: 3616 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
  Process: 3615 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
 Main PID: 3617 (k3s-server)
    Tasks: 65
   CGroup: /system.slice/k3s.service...
```

If you have worker nodes to add follow the additional steps to install the K3s agent per the source blog.
Also - run the steps to check proper usage of the etcd service included in the source blog.
