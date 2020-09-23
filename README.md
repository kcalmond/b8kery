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
#### etcd --version fails using commands below because you need to have ETC_UNSUPPORTED_ARCH-arm64 set first. Fixed this in later step setting up systemctl etc.service
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

#### Configure & start etcd Cluster
