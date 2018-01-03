Instructions to install OpenShift Origin on Atomic Host ( > 7.4)

# Why Atomic Host

It's designed to run linux containers specifically by leveraging the core
technologies of RHEL. Relatively easier to install OpenShift without 
considering any other dependeces and edge cases.

# Prerequisites

## DNS

- All the hosts should have a reslovable hostname that will be mainly used 
to generate the certficates

# Install etcd

```
# Create etcd ca
mkdir -P /etc/etcd/ca/certs
mkdir -P /etc/etcd/ca/crl
mkdir -P /etc/etcd/ca/fragments
cp /etc/pki/tls/openssl.cnf /etc/etcd/ca/fragments/
/etc/etcd/ca/openssl.cnf
touch /etc/etcd/ca/index.txt
cat /etc/etcd/ca/serial
01

openssl req -config /etc/etcd/ca/openssl.cnf \
            -newkey rsa:4096 \
            -keyout /etc/etcd/ca/ca.key \
            -new \
            -out /etc/etcd/ca/ca.crt \
            -x509 \
            -extensions etcd_v3_ca_self \
            -batch \
            -nodes \
            -days 1825 \
            -subj /CN=etcd-signer@1514980505

# Create the server csr
mkdir -P /etc/etcd/generated_certs/etcd-172.16.120.125

openssl req -new -keyout server.key -config /etc/etcd/ca/openssl.cnf -out server.csr -reqexts etcd_v3_req -batch -nodes -subj /CN=172.16.120.125

# Sign and create the server crt
openssl ca -name etcd_ca -config /etc/etcd/ca/openssl.cnf -out server.crt -in server.csr -extensions etcd_v3_ca_server -batch

# Create the peer csr
openssl req -new -keyout peer.key -config /etc/etcd/ca/openssl.cnf -out peer.csr -reqexts etcd_v3_req -batch -nodes -subj /CN=172.16.120.125

# Sign and create the peer crt
openssl ca -name etcd_ca -config /etc/etcd/ca/openssl.cnf -out peer.crt -in peer.csr -extensions etcd_v3_ca_peer -batch

cp /etc/etcd/ca/ca.crt /etc/etcd/generated_certs/etcd-172.16.120.125/ca.crt

tar -czvf /etc/etcd/generated_certs/etcd-172.16.120.125.tgz -C /etc/etcd/generated_certs/etcd-172.16.120.125 .

/usr/bin/gtar --extract -C /etc/etcd -z -f "/root/.ansible/tmp/ansible-tmp-1514980524.18-98625907039920/source


tar -czvf /etc/etcd/generated_certs/etcd_ca.tgz -C /etc/etcd/ca .

# 0600
/etc/etcd/ca.crt
/etc/etcd/server.crt
/etc/etcd/server.key

/etc/etcd/peer.crt
/etc/etcd/peer.key


# etcd master ca
mkdir -P /etc/etcd/generated_certs/openshift-master-172.16.120.125

# Create the client csr
openssl req -new -keyout master.etcd-client.key -config /etc/etcd/ca/openssl.cnf -out master.etcd-client.csr -reqexts etcd_v3_req -batch -nodes -subj /CN=172.16.120.125

#  Sign and create the client crt
openssl ca -name etcd_ca -config /etc/etcd/ca/openssl.cnf -out master.etcd-client.crt -in master.etcd-client.csr -batch

cp /etc/etcd/ca/ca.crt /etc/etcd/generated_certs/openshift-master-172.16.120.125/master.etcd-ca.crt

tar -czvf /etc/etcd/generated_certs/openshift-master-172.16.120.125.tgz -C /etc/etcd/generated_certs/openshift-master-172.16.120.125 .

mkdir -P /etc/origin/master

/usr/bin/gtar --extract -C /etc/origin/master -z -f /root/.ansible/tmp/ansible-tmp-1514980535.23-272233579863628/source

# 0600
/etc/origin/master/master.etcd-client.crt
/etc/origin/master/master.etcd-client.key
/etc/origin/master/master.etcd-ca.crt


# iptables

2379
2380 etcd peering

# install etcd

/etc/profile.d/etcdctl.sh

docker", "pull", "registry.access.redhat.com/rhel7/etcd

/etc/systemd/system/etcd_container.service

mkdir -P /var/lib/etcd/

/etc/etcd/etcd.conf
```


