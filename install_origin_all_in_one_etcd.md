
# install_origin_all_in_one_etcd.md 

```
systemctl start iptables && systemctl enable iptables

mkdir -p /etc/systemd/system/docker.service.d

cat <<EOF >/etc/systemd/system/docker.service.d/custom.conf
[Unit]
Wants=iptables.service
After=iptables.service

# The following line is a work-around to ensure docker is restarted whenever
# iptables is restarted.  This ensures the proper iptables rules will be in
# place for docker.
# Note:  This will also cause docker to be stopped if iptables is stopped.
PartOf=iptables.service

EOF

# https://bugzilla.redhat.com/show_bug.cgi?id=1509880
mkdir -p /var/lib/containers
restorecon -R /var/lib/containers/

# `docker login` needed

docker run --rm openshift3/ose:v3.9 version

timedatectl set-ntp true

mkdir -p /etc/etcd/ca/certs && chmod 700 /etc/etcd/ca/certs
mkdir -p /etc/etcd/ca/crl && chmod 700 /etc/etcd/ca/crl
mkdir -p /etc/etcd/ca/fragments && chmod 700 /etc/etcd/ca/fragments

cp /etc/pki/tls/openssl.cnf /etc/etcd/ca/fragments/openssl.cnf

cat <<EOF  >/etc/etcd/ca/fragments/openssl_append.cnf
[ etcd_v3_req ]
basicConstraints = critical,CA:FALSE
keyUsage         = digitalSignature,keyEncipherment
subjectAltName   = ${ENV::SAN}

[ etcd_ca ]
dir             = /etc/etcd/ca
crl_dir         = /etc/etcd/ca/crl
database        = /etc/etcd/ca/index.txt
new_certs_dir   = /etc/etcd/ca/certs
certificate     = /etc/etcd/ca/ca.crt
serial          = /etc/etcd/ca/serial
private_key     = /etc/etcd/ca/ca.key
crl_number      = /etc/etcd/ca/crlnumber
x509_extensions = etcd_v3_ca_client
default_days    = 1825
default_md      = sha256
preserve        = no
name_opt        = ca_default
cert_opt        = ca_default
policy          = policy_anything
unique_subject  = no
copy_extensions = copy

[ etcd_v3_ca_self ]
authorityKeyIdentifier = keyid,issuer
basicConstraints       = critical,CA:TRUE,pathlen:0
keyUsage               = critical,digitalSignature,keyEncipherment,keyCertSign
subjectKeyIdentifier   = hash

[ etcd_v3_ca_peer ]
authorityKeyIdentifier = keyid,issuer:always
basicConstraints       = critical,CA:FALSE
extendedKeyUsage       = clientAuth,serverAuth
keyUsage               = digitalSignature,keyEncipherment
subjectKeyIdentifier   = hash

[ etcd_v3_ca_server ]
authorityKeyIdentifier = keyid,issuer:always
basicConstraints       = critical,CA:FALSE
extendedKeyUsage       = serverAuth
keyUsage               = digitalSignature,keyEncipherment
subjectKeyIdentifier   = hash

[ etcd_v3_ca_client ]
authorityKeyIdentifier = keyid,issuer:always
basicConstraints       = critical,CA:FALSE
extendedKeyUsage       = clientAuth
keyUsage               = digitalSignature,keyEncipherment
subjectKeyIdentifier   = hash

EOF

cat /etc/etcd/ca/fragments/openssl.cnf /etc/etcd/ca/fragments/openssl_append.cnf >> /etc/etcd/ca/openssl.cnf

touch /etc/etcd/ca/index.txt

cat <<EOF >/etc/etcd/ca/serial
01
EOF

openssl req -config /etc/etcd/ca/openssl.cnf -newkey rsa:4096 -keyout /etc/etcd/ca/ca.key -new -out /etc/etcd/ca/ca.crt -x509 -extensions etcd_v3_ca_self -batch -nodes -days 1825 -subj /CN=etcd-signer@1518242756


# Create etcd server certificates for etcd hosts

mkdir /etc/etcd/generated_certs/etcd-172.16.120.54 && chmod 700 /etc/etcd/generated_certs/etcd-172.16.120.54

cd /etc/etcd/generated_certs/etcd-172.16.120.54

# Create the server csr
openssl req -new -keyout server.key -config /etc/etcd/ca/openssl.cnf -out server.csr -reqexts etcd_v3_req -batch -nodes -subj /CN=172.16.120.54
SAN: "IP:{{ etcd_ip }},DNS:{{ etcd_hostname }}"

openssl ca -name etcd_ca -config /etc/etcd/ca/openssl.cnf -out server.crt -in server.csr -extensions etcd_v3_ca_server -batch
SAN: "IP:{{ etcd_ip }}"

# Create the peer csr

openssl req -new -keyout peer.key -config /etc/etcd/ca/openssl.cnf -out peer.csr -reqexts etcd_v3_req -batch -nodes -subj /CN=172.16.120.54
SAN: "IP:{{ etcd_ip }},DNS:{{ etcd_hostname }}"

openssl ca -name etcd_ca -config /etc/etcd/ca/openssl.cnf -out peer.crt -in peer.csr -extensions etcd_v3_ca_peer -batch
SAN: "IP:{{ etcd_ip }}"

ln /etc/etcd/ca/ca.crt /etc/etcd/generated_certs/etcd-172.16.120.54/ca.crt

tar -czvf /etc/etcd/generated_certs/etcd-172.16.120.54.tgz -C /etc/etcd/generated_certs/etcd-172.16.120.54 .

/usr/bin/gtar --extract -C /etc/etcd -z -f /etc/etcd/generated_certs/etcd-172.16.120.54.tgz

tar -czvf /etc/etcd/generated_certs/etcd_ca.tgz -C /etc/etcd/ca .

chmod 600 /etc/etcd/ca.crt
chmod 600 /etc/etcd/peer.crt
chmod 600 /etc/etcd/peer.key

chmod 700 /etc/etcd

# Create etcd client certificates for master hosts

mkdir -p /etc/etcd/generated_certs/openshift-master-172.16.120.54 && chmod 700 /etc/etcd/generated_certs/openshift-master-172.16.120.54

# Create the client csr
openssl req -new -keyout master.etcd-client.key -config /etc/etcd/ca/openssl.cnf -out master.etcd-client.csr -reqexts etcd_v3_req -batch -nodes -subj /CN=172.16.120.54
SAN: "IP:{{ etcd_ip }},DNS:{{ etcd_hostname }}"

# Sign and create the client crt
openssl ca -name etcd_ca -config /etc/etcd/ca/openssl.cnf -out master.etcd-client.crt -in master.etcd-client.csr -batch
SAN: "IP:{{ etcd_ip }}"

ln /etc/etcd/ca/ca.crt /etc/etcd/generated_certs/openshift-master-172.16.120.54/master.etcd-ca.crt

tar -czvf /etc/etcd/generated_certs/openshift-master-172.16.120.54.tgz -C /etc/etcd/generated_certs/openshift-master-172.16.120.54 .

mkdir -p /etc/origin/master && chmod 755 /etc/origin/master

/usr/bin/gtar --extract -C /etc/origin/master -z -f /etc/etcd/generated_certs/openshift-master-172.16.120.54.tgz

chmod 600 /etc/origin/master/master.etcd-client.crt /etc/origin/master/master.etcd-client.key /etc/origin/master/master.etcd-ca.crt

# Add iptables allow rules
iptables -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 2379 -j ACCEPT
iptables -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 2380 -j ACCEPT

cat <<EOF >/etc/profile.d/etcdctl.sh
#!/bin/bash
# Sets up handy aliases for etcd, need etcdctl2 and etcdctl3 because
# command flags are different between the two. Should work on stand
# alone etcd hosts and master + etcd hosts too because we use the peer keys.
etcdctl2() {
 /usr/bin/etcdctl --cert-file /etc/etcd/peer.crt --key-file /etc/etcd/peer.key --ca-file /etc/etcd/ca.crt -C https://`hostname`:2379 ${@}

}

etcdctl3() {
 ETCDCTL_API=3 /usr/bin/etcdctl --cert /etc/etcd/peer.crt --key /etc/etcd/peer.key --cacert /etc/etcd/ca.crt --endpoints https://`hostname`:2379 ${@}
}
EOF

chmod 755 /etc/profile.d/etcdctl.sh

docker pull registry.access.redhat.com/rhel7/etcd

cat <<EOF >/etc/systemd/system/etcd_container.service
[Unit]
Description=The Etcd Server container
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
EnvironmentFile=/etc/etcd/etcd.conf
ExecStartPre=-/usr/bin/docker rm -f etcd_container
ExecStart=/usr/bin/docker run --name etcd_container --rm -v /var/lib/etcd/:/var/lib/etcd/:z -v /etc/etcd:/etc/etcd:ro --env-file=/etc/etcd/etcd.conf --net=host --entrypoint=/usr/bin/etcd registry.access.redhat.com/rhel7/etcd
ExecStop=/usr/bin/docker stop etcd_container
SyslogIdentifier=etcd_container
Restart=always
RestartSec=5s

[Install]
WantedBy=docker.service
EOF

mkdir -p /var/lib/etcd/ && chmod 700 /var/lib/etcd/

systemctl stop etcd && systemctl disable etcd

chmod 644 /etc/systemd/system/etcd_container.service

cat <<EOF >/etc/etcd/etcd.conf
ETCD_NAME=172.16.120.51
ETCD_LISTEN_PEER_URLS=https://172.16.120.51:2380
ETCD_DATA_DIR=/var/lib/etcd/
#ETCD_WAL_DIR=""
#ETCD_SNAPSHOT_COUNT=10000
ETCD_HEARTBEAT_INTERVAL=500
ETCD_ELECTION_TIMEOUT=2500
ETCD_LISTEN_CLIENT_URLS=https://172.16.120.51:2379
#ETCD_MAX_SNAPSHOTS=5
#ETCD_MAX_WALS=5
#ETCD_CORS=


#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS=https://172.16.120.51:2380
ETCD_INITIAL_CLUSTER=172.16.120.51=https://172.16.120.51:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster-1
#ETCD_DISCOVERY=
#ETCD_DISCOVERY_SRV=
#ETCD_DISCOVERY_FALLBACK=proxy
#ETCD_DISCOVERY_PROXY=
ETCD_ADVERTISE_CLIENT_URLS=https://172.16.120.51:2379
#ETCD_STRICT_RECONFIG_CHECK="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"
#ETCD_ENABLE_V2="true"
ETCD_QUOTA_BACKEND_BYTES=4294967296

#[proxy]
#ETCD_PROXY=off
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"

#[security]
ETCD_TRUSTED_CA_FILE=/etc/etcd/ca.crt
ETCD_CLIENT_CERT_AUTH="true"
ETCD_CERT_FILE=/etc/etcd/server.crt
ETCD_KEY_FILE=/etc/etcd/server.key
#ETCD_AUTO_TLS="false"
ETCD_PEER_TRUSTED_CA_FILE=/etc/etcd/ca.crt
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_CERT_FILE=/etc/etcd/peer.crt
ETCD_PEER_KEY_FILE=/etc/etcd/peer.key
#ETCD_PEER_AUTO_TLS="false"

#[logging]
ETCD_DEBUG="False"

#[profiling]
#ETCD_ENABLE_PPROF="false"
#ETCD_METRICS="basic"
#
#[auth]
#ETCD_AUTH_TOKEN="simple"
EOF

systemctl restart etcd_container && systemctl enable etcd_container

```

