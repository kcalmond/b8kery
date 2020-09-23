# b8kery lab build notes
Following these blog guidelines to build a pi4 based *HA* k3s cluster for lab workloads:
1. https://thenewstack.io/tutorial-install-a-highly-available-k3s-cluster-at-the-edge/
1. https://thenewstack.io/tutorial-set-up-a-secure-and-highly-available-etcd-cluster/

## ClusterOS Setup
Used "ubuntu-18.04.5-preinstalled-server-arm64+raspi4.img.xz" image available here: https://cdimage.ubuntu.com/releases/18.04/release/
nodes:
  * blueberry.almond.local 192.168.100.16
  * blackberry.almond.local 192.168.100.137
  * strawberry.almond.local 192.168.100.168

## ETCD Setup
#### Following steps in #2 above - changed "amd" to "arm"...
```bash
ETCD_VER=v3.4.10

DOWNLOAD_URL=https://storage.googleapis.com/etcd

rm -f /tmp/etcd-${ETCD_VER}-linux-arm64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-arm64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-arm64.tar.gz

tar xzvf /tmp/etcd-${ETCD_VER}-linux-arm64.tar.gz -C /tmp/etcd-download-test --strip-components=1

rm -f /tmp/etcd-${ETCD_VER}-linux-arm64.tar.gz

chmod +x /tmp/etcd-download-test/etcd
chmod +x /tmp/etcd-download-test/etcdctl
```
#### `etcd --version` fails using commands below because you need to have ETC_UNSUPPORTED_ARCH=arm64 set first. Fixed this in later step setting up systemctl etc.service
```bash
#Verify the downloads
/tmp/etcd-download-test/etcd --version
/tmp/etcd-download-test/etcdctl version
```

#### move binaries to /usr/bin (instead of /usr/local/bin in blog)
```bash
#Move them to the bin folder
sudo mv /tmp/etcd-download-test/etcd /usr/bin
sudo mv /tmp/etcd-download-test/etcdctl /usr/bin
```

#### Ran cfssl commands just as described in blog:
```bash
brew install cfssl
mkdir ~/b8kerybuild;cd b8kerybuild  #...do everything in a staging directory
echo '{"CN":"CA","key":{"algo":"rsa","size":2048}}' | cfssl gencert -initca - | cfssljson -bare ca -
echo '{"signing":{"default":{"expiry":"43800h","usages":["signing","key encipherment","server auth","client auth"]}}}' > ca-config.json
```

```bash
ls -l ~/b8kerybuild
-rw-r--r--  1 calmond  staff   112 Sep 22 21:33 ca-config.json
-rw-------  1 calmond  staff  1675 Sep 22 21:31 ca-key.pem
-rw-r--r--  1 calmond  staff   883 Sep 22 21:31 ca.csr
-rw-r--r--  1 calmond  staff  1070 Sep 22 21:31 ca.pem
```

#### Generate certs and keys for each node
```bash
export NAME=blueberry.almond.local
export ADDRESS=192.168.100.16,$NAME
echo '{"CN":"'$NAME'","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -config=ca-config.json -ca=ca.pem -ca-key=ca-key.pem -hostname="$ADDRESS" - | cfssljson -bare $NAME
export NAME=blackberry.almond.local
export ADDRESS=192.168.100.137,$NAME
echo '{"CN":"'$NAME'","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -config=ca-config.json -ca=ca.pem -ca-key=ca-key.pem -hostname="$ADDRESS" - | cfssljson -bare $NAME
export NAME=strawberry.almond.local
export ADDRESS=192.168.100.168,$NAME
echo '{"CN":"'$NAME'","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -config=ca-config.json -ca=ca.pem -ca-key=ca-key.pem -hostname="$ADDRESS" - | cfssljson -bare $NAME
```

```bash
ls -l ~/b8kerybuild
-rw-------  1 calmond  staff  1675 Sep 22 21:37 blackberry.almond.local-key.pem
-rw-r--r--  1 calmond  staff   989 Sep 22 21:37 blackberry.almond.local.csr
-rw-r--r--  1 calmond  staff  1241 Sep 22 21:37 blackberry.almond.local.pem
-rw-------  1 calmond  staff  1675 Sep 22 21:37 blueberry.almond.local-key.pem
-rw-r--r--  1 calmond  staff   989 Sep 22 21:37 blueberry.almond.local.csr
-rw-r--r--  1 calmond  staff  1241 Sep 22 21:37 blueberry.almond.local.pem
-rw-r--r--  1 calmond  staff   112 Sep 22 21:33 ca-config.json
-rw-------  1 calmond  staff  1675 Sep 22 21:31 ca-key.pem
-rw-r--r--  1 calmond  staff   883 Sep 22 21:31 ca.csr
-rw-r--r--  1 calmond  staff  1070 Sep 22 21:31 ca.pem
-rw-------  1 calmond  staff  1675 Sep 22 21:38 strawberry.almond.local-key.pem
-rw-r--r--  1 calmond  staff   989 Sep 22 21:38 strawberry.almond.local.csr
-rw-r--r--  1 calmond  staff  1241 Sep 22 21:38 strawberry.almond.local.pem
```

#### Distribute certs to cluster nodes. Passwordless ssh not setup between source (my mac client) and destination cluster nodes so ran each of these commands one at a time:
```
scp ./ca.pem ubuntu@192.168.100.16:etcd-ca.crt
scp blueberry.almond.local.pem ubuntu@192.168.100.16:server.crt
scp blueberry.almond.local-key.pem ubuntu@192.168.100.16:server.key
scp ./ca.pem ubuntu@192.168.100.137:etcd-ca.crt
scp blackberry.almond.local.pem ubuntu@192.168.100.137:server.crt
scp blackberry.almond.local-key.pem ubuntu@192.168.100.137:server.key
scp ./ca.pem ubuntu@192.168.100.168:etcd-ca.crt
scp strawberry.almond.local.pem ubuntu@192.168.100.168:server.crt
scp strawberry.almond.local-key.pem ubuntu@192.168.100.168:server.key
 ```

#### Configure `etcd.conf` on each node

Created `/etc/etcd/etcd.conf` on each cluster node.
***Note the addition of `ETCD_UNSUPPORTED_ARCH="arm64"` at end of each file.***
This is necessary to be able to start etcd on arm64. [[ref](https://github.com/etcd-io/etcd/issues/10677#issuecomment-558058616)]
```
ubuntu@strawberry:~$ cat /etc/etcd/etcd.conf
ETCD_NAME=strawberry.almond.lan
ETCD_LISTEN_PEER_URLS="https://192.168.100.168:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.100.168:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="blueberry.almond.lan=https://192.168.100.16:2380,blackberry.almond.lan=https://192.168.100.137:2380,strawberry.almond.lan=https://192.168.100.168:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.100.168:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.100.168:2379"
ETCD_TRUSTED_CA_FILE="/etc/etcd/etcd-ca.crt"
ETCD_CERT_FILE="/etc/etcd/server.crt"
ETCD_KEY_FILE="/etc/etcd/server.key"
ETCD_PEER_CLIENT_CERT_AUTH=true
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/etcd-ca.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/server.key"
ETCD_PEER_CERT_FILE="/etc/etcd/server.crt"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_UNSUPPORTED_ARCH="arm64"

ubuntu@blueberry:~$ cat /etc/etcd/etcd.conf
ETCD_NAME=blueberry.almond.lan
ETCD_LISTEN_PEER_URLS="https://192.168.100.16:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.100.16:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="blueberry.almond.lan=https://192.168.100.16:2380,blackberry.almond.lan=https://192.168.100.137:2380,strawberry.almond.lan=https://192.168.100.168:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.100.16:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.100.16:2379"
ETCD_TRUSTED_CA_FILE="/etc/etcd/etcd-ca.crt"
ETCD_CERT_FILE="/etc/etcd/server.crt"
ETCD_KEY_FILE="/etc/etcd/server.key"
ETCD_PEER_CLIENT_CERT_AUTH=true
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/etcd-ca.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/server.key"
ETCD_PEER_CERT_FILE="/etc/etcd/server.crt"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_UNSUPPORTED_ARCH="arm64"

ubuntu@blackberry:~$ cat /etc/etcd/etcd.conf
ETCD_NAME=blackberry.almond.lan
ETCD_LISTEN_PEER_URLS="https://192.168.100.137:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.100.137:2379"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER="blueberry.almond.lan=https://192.168.100.16:2380,blackberry.almond.lan=https://192.168.100.137:2380,strawberry.almond.lan=https://192.168.100.168:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.100.137:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.100.137:2379"
ETCD_TRUSTED_CA_FILE="/etc/etcd/etcd-ca.crt"
ETCD_CERT_FILE="/etc/etcd/server.crt"
ETCD_KEY_FILE="/etc/etcd/server.key"
ETCD_PEER_CLIENT_CERT_AUTH=true
ETCD_PEER_TRUSTED_CA_FILE="/etc/etcd/etcd-ca.crt"
ETCD_PEER_KEY_FILE="/etc/etcd/server.key"
ETCD_PEER_CERT_FILE="/etc/etcd/server.crt"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_UNSUPPORTED_ARCH="arm64"
```

#### Configure etcd.service
Added identical `/etc/systemd/system/etc.service` to each cluster host.
NOTE: the blog specifies `/lib/systemd/system/etc.service`. This is not correct file hierarchy. Local configurations should use /etc/systemd. (see `man system.unit`)
```bash
ubuntu@blueberry:~$ cat /etc/systemd/system/etcd.service
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
Type=notify
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd
Restart=always
RestartSec=10sj
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```

#### Start up & test etcd Cluster
```bash
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```

```bash
etcdctl --endpoints https://10.0.0.60:2379 --cert /etc/etcd/server.crt --cacert /etc/etcd/etcd-ca.crt --key /etc/etcd/server.key put foo bar

> etcdctl --endpoints https://192.168.100.168:2379 --cert /etc/etcd/server.crt --cacert /etc/etcd/etcd-ca.crt --key /etc/etcd/server.key put foo bar
OK

> etcdctl --endpoints https://192.168.100.168:2379 --cert /etc/etcd/server.crt --cacert /etc/etcd/etcd-ca.crt --key /etc/etcd/server.key get foo
foo
bar

> curl --cacert /etc/etcd/etcd-ca.crt --cert /etc/etcd/server.crt --key /etc/etcd/server.key https://192.168.100.168:2379/health
{"health":"true"}

> etcdctl --endpoints https://192.168.100.168:2379 --cert /etc/etcd/server.crt --cacert /etc/etcd/etcd-ca.crt --key /etc/etcd/server.key member list
31f4822107f1a1d3, started, blueberry.almond.lan, https://192.168.100.16:2380, https://192.168.100.16:2379, false
5246b58fd5117ab1, started, strawberry.almond.lan, https://192.168.100.168:2380, https://192.168.100.168:2379, false
5700b9ecd6ca26d0, started, blackberry.almond.lan, https://192.168.100.137:2380, https://192.168.100.137:2379, false

```
