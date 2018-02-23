# install_origin_all_in_one_master.md

```
mkdir -p /etc/origin/master/named_certificates/
chmod 600 /etc/origin/master/named_certificates/

docker pull openshift3/ose:v3.9.0

# Copy client binaries/symlinks out of CLI image for use on the host

# Create the master certificates
/usr/local/bin/oc adm ca create-master-certs --hostnames=kubernetes.default,kubernetes.default.svc.cluster.local,kubernetes,10.8.245.133,openshift.default,openshift.default.svc,172.16.120.54,172.30.0.1,host-8-245-133.host.centralci.eng.rdu2.redhat.com,openshift.default.svc.cluster.local,kubernetes.default.svc,openshift --master=https://172.16.120.54:8443 --public-master=https://host-8-245-133.host.centralci.eng.rdu2.redhat.com:8443 --cert-dir=/etc/origin/master --expire-days=730 --signer-expire-days=1825 --overwrite=false

cat /etc/origin/master/ca.crt >>/etc/origin/master/client-ca-bundle.crt

# Test local loopback context
/usr/local/bin/oc config view --config=/etc/origin/master/openshift-master.kubeconfig

rm /etc/origin/generated-configs/master-172.16.120.54/master.etcd-client.crt
rm /etc/origin/generated-configs/master-172.16.120.54/master.etcd-client.key

mkdir -p /root/.kube
chmod 700 /root/.kube
cp /etc/origin/master/admin.kubeconfig /root/.kube/config
chmod 700 /root/.kube/config

iptables -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 8443 -j ACCEPT
iptables -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 8444 -j ACCEPT
iptables -A OS_FIREWALL_ALLOW -p tcp -m state --state NEW -m tcp --dport 8053 -j ACCEPT
iptables -A OS_FIREWALL_ALLOW -p udp -m state --state NEW -m udp --dport 8053 -j ACCEPT

mkdir -p /var/lib/origin
systemctl daemon-reload

/usr/local/bin/oc adm create-bootstrap-policy-file --filename=/etc/origin/master/policy.json

cat <<EOF >/etc/origin/master/scheduler.json
{
    "apiVersion": "v1", 
    "kind": "Policy", 
    "predicates": [
        {
            "name": "NoVolumeZoneConflict"
        }, 
        {
            "name": "MaxEBSVolumeCount"
        }, 
        {
            "name": "MaxGCEPDVolumeCount"
        }, 
        {
            "name": "MaxAzureDiskVolumeCount"
        }, 
        {
            "name": "MatchInterPodAffinity"
        }, 
        {
            "name": "NoDiskConflict"
        }, 
        {
            "name": "GeneralPredicates"
        }, 
        {
            "name": "PodToleratesNodeTaints"
        }, 
        {
            "name": "CheckNodeMemoryPressure"
        }, 
        {
            "name": "CheckNodeDiskPressure"
        }, 
        {
            "name": "NoVolumeNodeConflict"
        }, 
        {
            "argument": {
                "serviceAffinity": {
                    "labels": [
                        "region"
                    ]
                }
            }, 
            "name": "Region"
        }
    ], 
    "priorities": [
        {
            "name": "SelectorSpreadPriority", 
            "weight": 1
        }, 
        {
            "name": "InterPodAffinityPriority", 
            "weight": 1
        }, 
        {
            "name": "LeastRequestedPriority", 
            "weight": 1
        }, 
        {
            "name": "BalancedResourceAllocation", 
            "weight": 1
        }, 
        {
            "name": "NodePreferAvoidPodsPriority", 
            "weight": 10000
        }, 
        {
            "name": "NodeAffinityPriority", 
            "weight": 1
        }, 
        {
            "name": "TaintTolerationPriority", 
            "weight": 1
        }, 
        {
            "argument": {
                "serviceAntiAffinity": {
                    "label": "zone"
                }
            }, 
            "name": "Zone", 
            "weight": 2
        }
    ]
}
EOF

cat <<EOF >/etc/systemd/journald.conf
#  This file is part of systemd.
#
#  systemd is free software; you can redistribute it and/or modify it
#  under the terms of the GNU Lesser General Public License as published by
#  the Free Software Foundation; either version 2.1 of the License, or
#  (at your option) any later version.
#
# Entries in this file show the compile time defaults.
# You can change settings by editing this file.
# Defaults can be restored by simply deleting this file.
#
# See journald.conf(5) for details.

[Journal]
 Storage=persistent
 Compress=True
#Seal=yes
#SplitMode=uid
 SyncIntervalSec=1s
 RateLimitInterval=1s
 RateLimitBurst=10000
 SystemMaxUse=8G
 SystemMaxFileSize=10M
#RuntimeKeepFree=
#RuntimeMaxFileSize=
 MaxRetentionSec=1month
 ForwardToSyslog=False
#ForwardToKMsg=no
#ForwardToConsole=no
 ForwardToWall=False
#TTYPath=/dev/console
#MaxLevelStore=debug
#MaxLevelSyslog=debug
#MaxLevelKMsg=notice
#MaxLevelConsole=info
#MaxLevelWall=emerg
 Storage=persistent
EOF

systemctl restart systemd-journald

rm /etc/systemd/system/atomic-openshift-master.service

docker pull openshift3/ose:v3.9.0

cat <<EOF >/etc/systemd/system/atomic-openshift-master-api.service
[Unit]
Description=Atomic OpenShift Master API
Documentation=https://github.com/openshift/origin
After=etcd_container.service
Wants=etcd_container.service
Before=atomic-openshift-node.service
After=docker.service
PartOf=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/etc/sysconfig/atomic-openshift-master-api
Environment=GOTRACEBACK=crash
ExecStartPre=-/usr/bin/docker rm -f atomic-openshift-master-api
ExecStart=/usr/bin/docker run --rm --privileged --net=host \
  --name atomic-openshift-master-api \
  --env-file=/etc/sysconfig/atomic-openshift-master-api \
  -v /var/lib/origin:/var/lib/origin \
  -v /var/log:/var/log -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc/origin:/etc/origin \
  \
  -v /etc/pki:/etc/pki:ro \
   -v /var/lib/origin/.docker:/root/.docker:ro\
  openshift3/ose:${IMAGE_VERSION} start master api \
  --config=${CONFIG_FILE} $OPTIONS
ExecStartPost=/usr/bin/sleep 10
ExecStop=/usr/bin/docker stop atomic-openshift-master-api
LimitNOFILE=131072
LimitCORE=infinity
WorkingDirectory=/var/lib/origin
SyslogIdentifier=atomic-openshift-master-api
Restart=always
RestartSec=5s

[Install]
WantedBy=docker.service
WantedBy=atomic-openshift-node.service
EOF

cat <<EOF >/etc/systemd/system/atomic-openshift-master-controllers.service
[Unit]
Description=Atomic OpenShift Master Controllers
Documentation=https://github.com/openshift/origin
Wants=atomic-openshift-master-api.service
After=atomic-openshift-master-api.service
After=docker.service
Requires=docker.service
PartOf=docker.service

[Service]
EnvironmentFile=/etc/sysconfig/atomic-openshift-master-controllers
Environment=GOTRACEBACK=crash
ExecStartPre=-/usr/bin/docker rm -f atomic-openshift-master-controllers
ExecStart=/usr/bin/docker run --rm --privileged --net=host \
  --name atomic-openshift-master-controllers \
  --env-file=/etc/sysconfig/atomic-openshift-master-controllers \
  -v /var/lib/origin:/var/lib/origin \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /etc/origin:/etc/origin \
  \
  -v /etc/pki:/etc/pki:ro \
   -v /var/lib/origin/.docker:/root/.docker:ro\
  openshift3/ose:${IMAGE_VERSION} start master controllers \
  --config=${CONFIG_FILE} $OPTIONS
ExecStartPost=/usr/bin/sleep 10
ExecStop=/usr/bin/docker stop atomic-openshift-master-controllers
LimitNOFILE=131072
LimitCORE=infinity
WorkingDirectory=/var/lib/origin
SyslogIdentifier=atomic-openshift-master-controllers
Restart=always
RestartSec=5s

[Install]
WantedBy=docker.service
EOF

systemctl daemon-reload

sytemctl enable atomic-openshift-master-api atomic-openshift-master-controllers

cat <<EOF >/etc/sysconfig/atomic-openshift-master-api
OPTIONS=--loglevel=5 --listen=https://0.0.0.0:8443 --master=https://172.16.120.51:8443
CONFIG_FILE=/etc/origin/master/master-config.yaml
OPENSHIFT_DEFAULT_REGISTRY=docker-registry.default.svc:5000
IMAGE_VERSION=v3.9.0


# Proxy configuration
# See https://docs.openshift.com/enterprise/latest/install_config/install/advanced_install.html#configuring-global-proxy
EOF

cat <<EOF >/etc/sysconfig/atomic-openshift-master-controllers
OPTIONS=--loglevel=5 --listen=https://0.0.0.0:8444
CONFIG_FILE=/etc/origin/master/master-config.yaml
OPENSHIFT_DEFAULT_REGISTRY=docker-registry.default.svc:5000
IMAGE_VERSION=v3.9.0


# Proxy configuration
# See https://docs.openshift.com/enterprise/latest/install_config/install/advanced_install.html#configuring-global-proxy
EOF

cat <<EOF >/etc/origin/master/session-secrets.yaml
apiVersion: v1
kind: SessionSecrets
secrets:
- authentication: "baLoOq5SfDHlCpK2zR9xsNYCBT67+84c"
  encryption: "baLoOq5SfDHlCpK2zR9xsNYCBT67+84c"
EOF
```

```
cat <<EOF >/etc/origin/master/master-config.yaml
apiVersion: v1
kind: SessionSecrets
secrets:
- authentication: "baLoOq5SfDHlCpK2zR9xsNYCBT67+84c"
  encryption: "baLoOq5SfDHlCpK2zR9xsNYCBT67+84c"
[root@host-172-16-120-51 ~]# cat /etc/origin/master/master-config.yaml
admissionConfig:
  pluginConfig:
    BuildDefaults:
      configuration:
        apiVersion: v1
        env: []
        kind: BuildDefaultsConfig
        resources:
          limits: {}
          requests: {}
    BuildOverrides:
      configuration:
        apiVersion: v1
        kind: BuildOverridesConfig
    PodPreset:
      configuration:
        apiVersion: v1
        disable: false
        kind: DefaultAdmissionConfig
    openshift.io/ImagePolicy:
      configuration:
        apiVersion: v1
        executionRules:
        - matchImageAnnotations:
          - key: images.openshift.io/deny-execution
            value: 'true'
          name: execution-denied
          onResources:
          - resource: pods
          - resource: builds
          reject: true
          skipOnResolutionFailure: true
        kind: ImagePolicyConfig
aggregatorConfig:
  proxyClientInfo:
    certFile: aggregator-front-proxy.crt
    keyFile: aggregator-front-proxy.key
apiLevels:
- v1
apiVersion: v1
authConfig:
  requestHeader:
    clientCA: front-proxy-ca.crt
    clientCommonNames:
    - aggregator-front-proxy
    extraHeaderPrefixes:
    - X-Remote-Extra-
    groupHeaders:
    - X-Remote-Group
    usernameHeaders:
    - X-Remote-User
controllerConfig:
  election:
    lockName: openshift-master-controllers
  serviceServingCert:
    signer:
      certFile: service-signer.crt
      keyFile: service-signer.key
controllers: '*'
corsAllowedOrigins:
- (?i)//127\.0\.0\.1(:|\z)
- (?i)//localhost(:|\z)
- (?i)//172\.16\.120\.51(:|\z)
- (?i)//10\.8\.245\.133(:|\z)
- (?i)//kubernetes\.default(:|\z)
- (?i)//kubernetes\.default\.svc\.cluster\.local(:|\z)
- (?i)//kubernetes(:|\z)
- (?i)//openshift\.default(:|\z)
- (?i)//openshift\.default\.svc(:|\z)
- (?i)//172\.30\.0\.1(:|\z)
- (?i)//host\-8\-245\-133\.host\.centralci\.eng\.rdu2\.redhat\.com(:|\z)
- (?i)//openshift\.default\.svc\.cluster\.local(:|\z)
- (?i)//kubernetes\.default\.svc(:|\z)
- (?i)//openshift(:|\z)
dnsConfig:
  bindAddress: 0.0.0.0:8053
  bindNetwork: tcp4
etcdClientInfo:
  ca: master.etcd-ca.crt
  certFile: master.etcd-client.crt
  keyFile: master.etcd-client.key
  urls:
  - https://172.16.120.51:2379
etcdStorageConfig:
  kubernetesStoragePrefix: kubernetes.io
  kubernetesStorageVersion: v1
  openShiftStoragePrefix: openshift.io
  openShiftStorageVersion: v1
imageConfig:
  format: registry.reg-aws.openshift.com:443/openshift3/ose-${component}:${version}
  latest: false
kind: MasterConfig
kubeletClientInfo:
  ca: ca-bundle.crt
  certFile: master.kubelet-client.crt
  keyFile: master.kubelet-client.key
  port: 10250
kubernetesMasterConfig:
  apiServerArguments:
    runtime-config:
    - apis/settings.k8s.io/v1alpha1=true
    storage-backend:
    - etcd3
    storage-media-type:
    - application/vnd.kubernetes.protobuf
  controllerArguments: null
  masterCount: 1
  masterIP: 172.16.120.51
  podEvictionTimeout: null
  proxyClientInfo:
    certFile: master.proxy-client.crt
    keyFile: master.proxy-client.key
  schedulerArguments: null
  schedulerConfigFile: /etc/origin/master/scheduler.json
  servicesNodePortRange: ''
  servicesSubnet: 172.30.0.0/16
  staticNodeNames: []
masterClients:
  externalKubernetesClientConnectionOverrides:
    acceptContentTypes: application/vnd.kubernetes.protobuf,application/json
    burst: 400
    contentType: application/vnd.kubernetes.protobuf
    qps: 200
  externalKubernetesKubeConfig: ''
  openshiftLoopbackClientConnectionOverrides:
    acceptContentTypes: application/vnd.kubernetes.protobuf,application/json
    burst: 600
    contentType: application/vnd.kubernetes.protobuf
    qps: 300
  openshiftLoopbackKubeConfig: openshift-master.kubeconfig
masterPublicURL: https://host-8-245-133.host.centralci.eng.rdu2.redhat.com:8443
networkConfig:
  clusterNetworkCIDR: 10.128.0.0/14
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostSubnetLength: 9
  externalIPNetworkCIDRs:
  - 0.0.0.0/0
  hostSubnetLength: 9
  networkPluginName: redhat/openshift-ovs-subnet
  serviceNetworkCIDR: 172.30.0.0/16
oauthConfig:
  assetPublicURL: https://host-8-245-133.host.centralci.eng.rdu2.redhat.com:8443/console/
  grantConfig:
    method: auto
  identityProviders:
  - challenge: true
    login: true
    mappingMethod: claim
    name: allow_all
    provider:
      apiVersion: v1
      kind: AllowAllPasswordIdentityProvider
  masterCA: ca-bundle.crt
  masterPublicURL: https://host-8-245-133.host.centralci.eng.rdu2.redhat.com:8443
  masterURL: https://172.16.120.51:8443
  sessionConfig:
    sessionMaxAgeSeconds: 3600
    sessionName: ssn
    sessionSecretsFile: /etc/origin/master/session-secrets.yaml
  tokenConfig:
    accessTokenMaxAgeSeconds: 86400
    authorizeTokenMaxAgeSeconds: 500
pauseControllers: false
policyConfig:
  bootstrapPolicyFile: /etc/origin/master/policy.json
  openshiftInfrastructureNamespace: openshift-infra
  openshiftSharedResourcesNamespace: openshift
projectConfig:
  defaultNodeSelector: ''
  projectRequestMessage: ''
  projectRequestTemplate: ''
  securityAllocator:
    mcsAllocatorRange: s0:/2
    mcsLabelsPerProject: 5
    uidAllocatorRange: 1000000000-1999999999/10000
routingConfig:
  subdomain: apps.0210-jlr.qe.rhcloud.com
serviceAccountConfig:
  limitSecretReferences: false
  managedNames:
  - default
  - builder
  - deployer
  masterCA: ca-bundle.crt
  privateKeyFile: serviceaccounts.private.key
  publicKeyFiles:
  - serviceaccounts.public.key
servingInfo:
  bindAddress: 0.0.0.0:8443
  bindNetwork: tcp4
  certFile: master.server.crt
  clientCA: ca.crt
  keyFile: master.server.key
  maxRequestsInFlight: 500
  requestTimeoutSeconds: 3600
volumeConfig:
  dynamicProvisioningEnabled: true
EOF
```
systemctl start atomic-openshift-master-api

curl --silent --tlsv1.2 --cacert /etc/origin/master/ca-bundle.crt https://172.16.120.51:8443/healthz/ready

systemctl start atomic-openshift-master-controllers

cat <<EOF >/etc/tuned/openshift/tuned.conf 
#
# tuned configuration
#

[main]
summary=Optimize systems running OpenShift (parent profile)
include=${f:virt_check:atomic-guest:throughput-performance}

[selinux]
avc_cache_threshold=65536

[net]
nf_conntrack_hashsize=131072

[sysctl]
kernel.pid_max=131072
net.netfilter.nf_conntrack_max=1048576
fs.inotify.max_user_watches=65536
net.ipv4.neigh.default.gc_thresh1=8192
net.ipv4.neigh.default.gc_thresh2=32768
net.ipv4.neigh.default.gc_thresh3=65536
net.ipv6.neigh.default.gc_thresh1=8192
net.ipv6.neigh.default.gc_thresh2=32768
net.ipv6.neigh.default.gc_thresh3=65536
EOF

cat <<EOF >/etc/tuned/openshift-control-plane/tuned.conf 
#
# tuned configuration
#

[main]
summary=Optimize systems running OpenShift control plane
include=openshift

[sysctl]
# ktune sysctl settings, maximizing i/o throughput
#
# Minimal preemption granularity for CPU-bound tasks:
# (default: 1 msec#  (1 + ilog(ncpus)), units: nanoseconds)
kernel.sched_min_granularity_ns=10000000

# The total time the scheduler will consider a migrated process
# "cache hot" and thus less likely to be re-migrated
# (system default is 500000, i.e. 0.5 ms)
kernel.sched_migration_cost_ns=5000000

# SCHED_OTHER wake-up granularity.
#
# Preemption granularity when tasks wake up.  Lower the value to improve 
# wake-up latency and throughput for latency critical tasks.
kernel.sched_wakeup_granularity_ns = 4000000
EOF

cat <<EOF >/etc/tuned/openshift-node/tuned.conf 
#
# tuned configuration
#

[main]
summary=Optimize systems running OpenShift nodes
include=openshift

[sysctl]
net.ipv4.tcp_fastopen=3
EOF

cat <<EOF > /etc/tuned/active_profile 
openshift-control-plane
EOF

systemctl restart tuned

# Creating First Master Aggregator signer certs
/usr/local/bin/oc adm ca create-signer-cert --cert=/etc/origin/master/front-proxy-ca.crt --key=/etc/origin/master/front-proxy-ca.key --serial=/etc/origin/master/ca.serial.txt

# Create first master api-client config for Aggregator
/usr/local/bin/oc adm create-api-client-config --certificate-authority=/etc/origin/master/front-proxy-ca.crt --signer-cert=/etc/origin/master/front-proxy-ca.crt --signer-key=/etc/origin/master/front-proxy-ca.key --user aggregator-front-proxy --client-dir=/tmp/openshift-service-catalog-ansible-XBR0XM --signer-serial=/etc/origin/master/ca.serial.txt

cp /tmp/openshift-service-catalog-ansible-XBR0XM/aggregator-front-proxy.crt /etc/origin/master/aggregator-front-proxy.crt
cp /tmp/openshift-service-catalog-ansible-XBR0XM/aggregator-front-proxy.key /etc/origin/master/aggregator-front-proxy.key
cp /tmp/openshift-service-catalog-ansible-XBR0XM/aggregator-front-proxy.kubeconfig /etc/origin/master/aggregator-front-proxy.kubeconfig
systemctl restart atomic-openshift-master-api
systemctl restart atomic-openshift-master-controllers
curl --silent --tlsv1.2 --cacert /etc/origin/master/ca-bundle.crt https://172.16.120.51:8443/healthz/ready
```
