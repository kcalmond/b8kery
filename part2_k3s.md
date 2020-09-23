# b8kery lab build PART 2
Following the steps in the blog below to complete second phase (k3s config) here:

https://thenewstack.io/tutorial-install-a-highly-available-k3s-cluster-at-the-edge/

NOTE: this is an *experimental* HA config since etcd is considered "experimental" right now on arm64.

## Add `cgroup_memory=1 cgroup_enable=memory` to /boot/firmware/nobtcmd.txt
K3s startup failed until I troubleshooted and found that for my install of Ubuntu 18.04 on RPi4b/arm64 nodes you need to first add the last two parameters to nobtcmd.txt:
```bash
> cat /boot/firmware/nobtcmd.txt
net.ifnames=0 dwc_otg.lpm_enable=0 console=ttyAMA0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc cgroup_memory=1 cgroup_enable=memory
```

Reboot, check the health of your fresh etcd cluster post-reboot, and proceed.

#### Install k3s

I'm running a 3-node cluster (no extra "agent" node per the source blog above):

Host | IP | K3s role
---- | -- | --------
blueberry.almond.lan | 192.658.100.16 | etc,server
blackberry.almond.lan | 192.658.100.137 | etc,server
strawberry.almond.lan | 192.658.100.168 | etc,server
