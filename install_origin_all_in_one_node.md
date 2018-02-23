```
mkdir -p /etc/origin/generated-configs

# Generate the node client config
/usr/local/bin/oc", "adm", "create-api-client-config", "--certificate-authority=/etc/origin/master/ca.crt", "--client-dir=/etc/origin/generated-configs/node-172.16.120.51", "--groups=system:nodes", "--master=https://172.16.120.51:8443", "--signer-cert=/etc/origin/master/ca.crt", "--signer-key=/etc/origin/master/ca.key", "--signer-serial=/etc/origin/master/ca.serial.txt", "--user=system:node:172.16.120.51", "--expire-days=730"

# Generate the node server certificate
/usr/local/bin/oc", "adm", "ca", "create-server-cert", "--cert=/etc/origin/generated-configs/node-172.16.120.51/server.crt", "--key=/etc/origin/generated-configs/node-172.16.120.51/server.key", "--expire-days=730", "--overwrite=true", "--hostnames=172.16.120.51,172.16.120.51,host-8-245-133.host.centralci.eng.rdu2.redhat.com,host-8-245-133.host.centralci.eng.rdu2.redhat.com,172.16.120.51,10.8.245.133", "--signer-cert=/etc/origin/master/ca.crt", "--signer-key=/etc/origin/master/ca.key", "--signer-serial=/etc/origin/master/ca.serial.txt

tar", "-czvf", "/etc/origin/generated-configs/node-172.16.120.51.tgz", "--transform", "s|system:node-172.16.120.51|node|", "-C", "/etc/origin/generated-configs/node-172.16.120.51", ".

mkdir -p /etc/origin/node
/usr/bin/gtar", "--extract", "-C", "/etc/origin/node", "-z", "-f", "/root/.ansible/tmp/ansible-tmp-1518242981.66-202754867838319/source

#Copy OpenShift CA to system CA trust
cp /etc/origin/node/ca.crt /etc/pki/ca-trust/source/anchors/openshift-ca.crt
update-ca-trust
systemctl restart docker

mkdir -p /etc/origin
mkdir -p /etc/origin/node

cat <<EOF >/etc/origin/node/node-dnsmasq.conf
server=/in-addr.arpa/127.0.0.1
server=/cluster.local/127.0.0.1
EOF

cat <<EOF >/etc/dnsmasq.d/origin-dns.conf
no-resolv
domain-needed
no-negcache
max-cache-ttl=1
enable-dbus
dns-forward-max=5000
cache-size=5000
bind-dynamic
except-interface=lo
# End of config
EOF

systemctl enable dnsmasq

cat <<EOF >/etc/NetworkManager/dispatcher.d/99-origin-dns.sh
#!/bin/bash -x
# -*- mode: sh; sh-indentation: 2 -*-

# This NetworkManager dispatcher script replicates the functionality of
# NetworkManager's dns=dnsmasq  however, rather than hardcoding the listening
# address and /etc/resolv.conf to 127.0.0.1 it pulls the IP address from the
# interface that owns the default route. This enables us to then configure pods
# to use this IP address as their only resolver, where as using 127.0.0.1 inside
# a pod would fail.
#
# To use this,
# - If this host is also a master, reconfigure master dnsConfig to listen on
#   8053 to avoid conflicts on port 53 and open port 8053 in the firewall
# - Drop this script in /etc/NetworkManager/dispatcher.d/
# - systemctl restart NetworkManager
# - Configure node-config.yaml to set dnsIP: to the ip address of this
#   node
#
# Test it:
# host kubernetes.default.svc.cluster.local
# host google.com
#
# TODO: I think this would be easy to add as a config option in NetworkManager
# natively, look at hacking that up

cd /etc/sysconfig/network-scripts
. ./network-functions

[ -f ../network ] && . ../network

if [[ $2 =~ ^(up|dhcp4-change|dhcp6-change)$ ]]; then
  # If the origin-upstream-dns config file changed we need to restart
  NEEDS_RESTART=0
  UPSTREAM_DNS='/etc/dnsmasq.d/origin-upstream-dns.conf'
  # We'll regenerate the dnsmasq origin config in a temp file first
  UPSTREAM_DNS_TMP=`mktemp`
  UPSTREAM_DNS_TMP_SORTED=`mktemp`
  CURRENT_UPSTREAM_DNS_SORTED=`mktemp`
  NEW_RESOLV_CONF=`mktemp`
  NEW_NODE_RESOLV_CONF=`mktemp`


  ######################################################################
  # couldn't find an existing method to determine if the interface owns the
  # default route
  def_route=$(/sbin/ip route list match 0.0.0.0/0 | awk '{print $3 }')
  def_route_int=$(/sbin/ip route get to ${def_route} | awk '{print $3}')
  def_route_ip=$(/sbin/ip route get to ${def_route} | awk '{print $5}')
  if [[ ${DEVICE_IFACE} == ${def_route_int} ]]; then
    if [ ! -f /etc/dnsmasq.d/origin-dns.conf ]; then
      cat << EOF > /etc/dnsmasq.d/origin-dns.conf
no-resolv
domain-needed
server=/cluster.local/172.30.0.1
server=/30.172.in-addr.arpa/172.30.0.1
enable-dbus
dns-forward-max=5000
cache-size=5000
EOF
      # New config file, must restart
      NEEDS_RESTART=1
    fi

    # If network manager doesn't know about the nameservers then the best
    # we can do is grab them from /etc/resolv.conf but only if we've got no
    # watermark
    if ! grep -q '99-origin-dns.sh' /etc/resolv.conf; then
      if [[ -z "${IP4_NAMESERVERS}" || "${IP4_NAMESERVERS}" == "${def_route_ip}" ]]; then
            IP4_NAMESERVERS=`grep '^nameserver ' /etc/resolv.conf | awk '{ print $2 }'`
      fi
      ######################################################################
      # Write out default nameservers for /etc/dnsmasq.d/origin-upstream-dns.conf
      # and /etc/origin/node/resolv.conf in their respective formats
      for ns in ${IP4_NAMESERVERS}; do
        if [[ ! -z $ns ]]; then
          echo "server=${ns}" >> $UPSTREAM_DNS_TMP
          echo "nameserver ${ns}" >> $NEW_NODE_RESOLV_CONF
        fi
      done
      # Sort it in case DNS servers arrived in a different order
      sort $UPSTREAM_DNS_TMP > $UPSTREAM_DNS_TMP_SORTED
      sort $UPSTREAM_DNS > $CURRENT_UPSTREAM_DNS_SORTED
      # Compare to the current config file (sorted)
      NEW_DNS_SUM=`md5sum ${UPSTREAM_DNS_TMP_SORTED} | awk '{print $1}'`
      CURRENT_DNS_SUM=`md5sum ${CURRENT_UPSTREAM_DNS_SORTED} | awk '{print $1}'`
      if [ "${NEW_DNS_SUM}" != "${CURRENT_DNS_SUM}" ]; then
        # DNS has changed, copy the temp file to the proper location (-Z
        # sets default selinux context) and set the restart flag
        cp -Z $UPSTREAM_DNS_TMP $UPSTREAM_DNS
        NEEDS_RESTART=1
      fi
      # compare /etc/origin/node/resolv.conf checksum and replace it if different
      NEW_NODE_RESOLV_CONF_MD5=`md5sum ${NEW_NODE_RESOLV_CONF}`
      OLD_NODE_RESOLV_CONF_MD5=`md5sum /etc/origin/node/resolv.conf`
      if [ "${NEW_NODE_RESOLV_CONF_MD5}" != "${OLD_NODE_RESOLV_CONF_MD5}" ]; then
        cp -Z $NEW_NODE_RESOLV_CONF /etc/origin/node/resolv.conf
      fi
    fi

    if ! `systemctl -q is-active dnsmasq.service`; then
      NEEDS_RESTART=1
    fi

    ######################################################################
    if [ "${NEEDS_RESTART}" -eq "1" ]; then
      systemctl restart dnsmasq
    fi

    # Only if dnsmasq is running properly make it our only nameserver and place
    # a watermark on /etc/resolv.conf
    if `systemctl -q is-active dnsmasq.service`; then
      if ! grep -q '99-origin-dns.sh' /etc/resolv.conf; then
          echo "# nameserver updated by /etc/NetworkManager/dispatcher.d/99-origin-dns.sh" >> ${NEW_RESOLV_CONF}
      fi
      sed -e '/^nameserver.*$/d' /etc/resolv.conf >> ${NEW_RESOLV_CONF}
      echo "nameserver "${def_route_ip}"" >> ${NEW_RESOLV_CONF}
      if ! grep -qw search ${NEW_RESOLV_CONF}; then
        echo 'search cluster.local' >> ${NEW_RESOLV_CONF}
      elif ! grep -q 'search.*cluster.local' ${NEW_RESOLV_CONF}; then
        sed -i '/^search/ s/$/ cluster.local/' ${NEW_RESOLV_CONF}
      fi
      cp -Z ${NEW_RESOLV_CONF} /etc/resolv.conf
    fi
  fi

  # Clean up after yourself
  rm -f $UPSTREAM_DNS_TMP $UPSTREAM_DNS_TMP_SORTED $CURRENT_UPSTREAM_DNS_SORTED $NEW_RESOLV_CONF
fi
EOF

systemctl restart NetworkManager
systemctl restar dnsmasq

# kubelet
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 10250 -j ACCEPT
# http
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
# https
-A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 443 -j ACCEPT
# sdn
-A OS_FIREWALL_ALLOW -p udp -m state --state NEW -m udp --dport 4789 -j ACCEPT

docker pull openshift3/node:v3.9.0 
cat <<EOF >/etc/sysctl.d/99-openshift.conf
net.ipv4.ip_forward=1
EOF
sysctl -p


cat <<EOF >/etc/systemd/system/atomic-openshift-node.service
[Unit]
After=atomic-openshift-master.service
After=docker.service
After=openvswitch.service
PartOf=docker.service
Requires=docker.service
Wants=openvswitch.service
PartOf=openvswitch.service
After=ovsdb-server.service
After=ovs-vswitchd.service
Wants=atomic-openshift-master.service
Requires=atomic-openshift-node-dep.service
After=atomic-openshift-node-dep.service
Wants=dnsmasq.service
After=dnsmasq.service

[Service]
EnvironmentFile=/etc/sysconfig/atomic-openshift-node
EnvironmentFile=/etc/sysconfig/atomic-openshift-node-dep
ExecStartPre=-/usr/bin/docker rm -f atomic-openshift-node
ExecStartPre=/usr/bin/cp /etc/origin/node/node-dnsmasq.conf /etc/dnsmasq.d/
ExecStartPre=/usr/bin/dbus-send --system --dest=uk.org.thekelleys.dnsmasq /uk/org/thekelleys/dnsmasq uk.org.thekelleys.SetDomainServers array:string:/in-addr.arpa/127.0.0.1,/cluster.local/127.0.0.1
ExecStart=/usr/bin/docker run --name atomic-openshift-node \
  --rm --privileged --net=host --pid=host --env-file=/etc/sysconfig/atomic-openshift-node \
  -v /:/rootfs:ro,rslave -e CONFIG_FILE=${CONFIG_FILE} -e OPTIONS=${OPTIONS} \
  -e HOST=/rootfs -e HOST_ETC=/host-etc \
  -v /var/lib/origin:/var/lib/origin:rslave \
  -v /etc/origin/node:/etc/origin/node \
  \
  -v /etc/localtime:/etc/localtime:ro -v /etc/machine-id:/etc/machine-id:ro \
  -v /run:/run -v /sys:/sys:rw -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  -v /usr/bin/docker:/usr/bin/docker:ro -v /var/lib/docker:/var/lib/docker \
  -v /lib/modules:/lib/modules -v /etc/origin/openvswitch:/etc/openvswitch \
  -v /etc/origin/sdn:/etc/openshift-sdn -v /var/lib/cni:/var/lib/cni \
  -v /etc/systemd/system:/host-etc/systemd/system -v /var/log:/var/log \
  \
  -v /dev:/dev $DOCKER_ADDTL_BIND_MOUNTS -v /etc/pki:/etc/pki:ro \
   -v /var/lib/origin/.docker:/root/.docker:ro\
  openshift3/node:${IMAGE_VERSION}
ExecStartPost=/usr/bin/sleep 10
ExecStop=/usr/bin/docker stop atomic-openshift-node
ExecStopPost=/usr/bin/rm /etc/dnsmasq.d/node-dnsmasq.conf
ExecStopPost=/usr/bin/dbus-send --system --dest=uk.org.thekelleys.dnsmasq /uk/org/thekelleys/dnsmasq uk.org.thekelleys.SetDomainServers array:string:
SyslogIdentifier=atomic-openshift-node
Restart=always
RestartSec=5s

[Install]
WantedBy=docker.service
EOF

cat <<EOF >/etc/systemd/system/atomic-openshift-node-dep.service

[Unit]
Requires=docker.service
After=docker.service
PartOf=atomic-openshift-node.service
Before=atomic-openshift-node.service

[Service]
ExecStart=/bin/bash -c 'if [[ -f /usr/bin/docker-current ]]; \
 then echo DOCKER_ADDTL_BIND_MOUNTS=\"--volume=/usr/bin/docker-current:/usr/bin/docker-current:ro \
 --volume=/etc/sysconfig/docker:/etc/sysconfig/docker:ro \
 --volume=/etc/containers/registries:/etc/containers/registries:ro \
  --volume=/var/lib/origin/.docker:/root/.docker:ro\" > \
 /etc/sysconfig/atomic-openshift-node-dep; \
 else echo "#DOCKER_ADDTL_BIND_MOUNTS=" > /etc/sysconfig/atomic-openshift-node-dep; fi'
ExecStop=
SyslogIdentifier=atomic-openshift-node-dep

EOF

cat <<EOF >/etc/sysconfig/openvswitch
IMAGE_VERSION=v3.9.0
EOF

cat <<EOF >/etc/systemd/system/openvswitch.service
[Unit]
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
EnvironmentFile=/etc/sysconfig/openvswitch
ExecStartPre=-/usr/bin/docker rm -f openvswitch
ExecStart=/usr/bin/docker run --name openvswitch --rm --privileged --net=host --pid=host -v /lib/modules:/lib/modules -v /run:/run -v /sys:/sys:ro -v /etc/origin/openvswitch:/etc/openvswitch openshift3/openvswitch:${IMAGE_VERSION}
ExecStartPost=/usr/bin/sleep 5
ExecStop=/usr/bin/docker stop openvswitch
SyslogIdentifier=openvswitch
Restart=always
RestartSec=5s

[Install]
WantedBy=docker.service
EOF

cat <<EOF > /etc/sysconfig/atomic-openshift-node
OPTIONS=--loglevel=5
CONFIG_FILE=/etc/origin/node/node-config.yaml
IMAGE_VERSION=v3.9.0
EOF

docker pull openshift3/openvswitch:v3.9.0

systemctl restart openvswitch
systemctl enable openvswitch

cat <<EOF >/etc/origin/node/node-config.yaml
OPTIONS=--loglevel=5
CONFIG_FILE=/etc/origin/node/node-config.yaml
IMAGE_VERSION=v3.9.0
[root@host-172-16-120-51 ~]# caat /etc/origin/node/node-config.yaml
bash: caat: command not found
[root@host-172-16-120-51 ~]# cat /etc/origin/node/node-config.yaml
allowDisabledDocker: false
apiVersion: v1
dnsBindAddress: 127.0.0.1:53
dnsRecursiveResolvConf: /etc/origin/node/resolv.conf
dnsDomain: cluster.local
dnsIP: 172.16.120.51
dockerConfig:
  execHandlerName: ""
iptablesSyncPeriod: "30s"
imageConfig:
  format: registry.reg-aws.openshift.com:443/openshift3/ose-${component}:${version}
  latest: False
kind: NodeConfig
kubeletArguments: 
  enable-controller-attach-detach:
  - 'true'
  image-gc-high-threshold:
  - '80'
  image-gc-low-threshold:
  - '70'
  maximum-dead-containers:
  - '20'
  maximum-dead-containers-per-container:
  - '1'
  minimum-container-ttl-duration:
  - 10s
  node-labels:
  - router=enabled
  - role=node
  - registry=enabled
masterClientConnectionOverrides:
  acceptContentTypes: application/vnd.kubernetes.protobuf,application/json
  contentType: application/vnd.kubernetes.protobuf
  burst: 200
  qps: 100
masterKubeConfig: system:node:172.16.120.51.kubeconfig
networkPluginName: redhat/openshift-ovs-subnet
# networkConfig struct introduced in origin 1.0.6 and OSE 3.0.2 which
# deprecates networkPluginName above. The two should match.
networkConfig:
   mtu: 1350
   networkPluginName: redhat/openshift-ovs-subnet
nodeName: 172.16.120.51
podManifestConfig:
servingInfo:
  bindAddress: 0.0.0.0:10250
  certFile: server.crt
  clientCA: ca.crt
  keyFile: server.key
volumeDirectory: /var/lib/origin/openshift.local.volumes
proxyArguments:
  proxy-mode:
     - iptables
volumeConfig:
  localQuota:
    perFSGroup: 
EOF

curl", "--silent", "--tlsv1.2", "--cacert", "/etc/origin/node/ca.crt", "https://172.16.120.51:8443/healthz/ready

systemctl restart openshift-node-dep
systemctl enable openshift-node-dep

systemctl restart atomic-openshift-node
systemctl enable atomic-openshift-node

# Set seboolean to allow nfs storage plugin access from containers

getsebool', u'virt_use_nfs

#Set seboolean to allow gluster storage plugin access from containers
getsebool', u'virt_use_fusefs

getsebool', u'virt_sandbox_use_fusefs

mkdir -p /etc/systemd/system/openvswitch.service.d/

cat <<EOF >/etc/systemd/system/openvswitch.service.d/01-avoid-oom.conf
# Avoid the OOM killer for openvswitch and it's children:
[Service]
OOMScoreAdjust=-1000
EOF

systemctl daemon-reload

/usr/local/bin/oc label node 172.16.120.51 node-role.kubernetes.io/master=true router=enabled role=node registry=enabled --overwrite -n default








```
