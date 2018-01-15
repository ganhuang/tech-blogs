# How to create LbaaS v2

Reference: https://docs.openstack.org/mitaka/networking-guide/config-lbaas.html

```
neutron lbaas-loadbalancer-create --name test-lb private_subnet
neutron lbaas-listener-create   --name test-lb-https   --loadbalancer test-lb   --protocol HTTPS   --protocol-port 443
neutron lbaas-pool-create   --name test-lb-pool-https   --lb-algorithm LEAST_CONNECTIONS   --listener test-lb-https   --protocol HTTPS
neutron lbaas-healthmonitor-create   --delay 5   --max-retries 2   --timeout 10   --type HTTPS   --pool test-lb-pool-https
neutron port-list
neutron floatingip-list
neutron floatingip-associate 10.14.89.130 427982af-5026-40fe-9fb2-a0800c120769



neutron lbaas-member-create \
  --subnet private_subnet \
  --address 192.168.1.16 \
  --protocol-port 443 \
  test-lb-pool-https

```
