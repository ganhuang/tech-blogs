```
# generate a default wildcard router certificate

/usr/local/bin/oc adm ca create-server-cert --cert=/etc/origin/master/openshift-router.crt --hostnames=apps.0210-jlr.qe.rhcloud.com,*.apps.0210-jlr.qe.rhcloud.com --key=/etc/origin/master/openshift-router.key --overwrite=True --signer-cert=/etc/origin/master/ca.crt --signer-key=/etc/origin/master/ca.key --signer-serial=/etc/origin/master/ca.serial.txt

# Create the router service account(s)

# Grant the router service account(s) access to the appropriate scc

# Set additional permissions for router service account

#  Create OpenShift router

# Create the registry service account

usr/local/bin/oc get sa registry -o json -n default

/usr/local/bin/oc adm policy add-scc-to-user hostnetwork system:serviceaccount:default:registry -n default"

/usr/local/bin/oc adm policy add-cluster-role-to-user system:registry system:serviceaccount:default:registry -n default

/usr/local/bin/oc get service docker-registry -o json -n default

/usr/local/bin/oc get route docker-registry -o json -n default

# Generate self-signed docker-registry certificates
/usr/local/bin/oc adm ca create-server-cert --cert=/etc/origin/master/registry.crt --expire-days=730 --hostnames=172.30.172.41,docker-registry-default.apps.0210-jlr.qe.rhcloud.com,docker-registry.default.svc,docker-registry.default.svc.cluster.local,__omit_place_holder__fb0b8b649ac230f55f40f4ac18e7fbdf4cfbc34d --key=/etc/origin/master/registry.key --overwrite=True --signer-cert=/etc/origin/master/ca.crt --signer-key=/etc/origin/master/ca.key --signer-serial=/etc/origin/master/ca.serial.txt

#  Generate certificate bundle

usr/local/bin/oc secrets new registry-certificates registry.crt=/etc/origin/master/named_certificates/docker-registry.pem registry.key=/etc/origin/master/registry.key -n default

# Add the secret to the registry's pod service accounts
/usr/local/bin/oc get sa registry -o json -n default

/usr/local/bin/oc get sa registry -o json -n default

/usr/local/bin/oc create -f /tmp/deploymentconfigr9fiCI -n default

/usr/local/bin/oc", "rollout", "status", "deploymentconfig", "router", "--namespace", "default", "--config", "/etc/origin/master/admin.kubeconfig

/usr/local/bin/oc", "get", "deploymentconfig", "router", "--namespace", "default", "--config", "/etc/origin/master/admin.kubeconfig", "-o", "jsonpath={ .status.latestVersion }"

usr/local/bin/oc", "rollout", "status", "deploymentconfig", "docker-registry", "--namespace", "default", "--config", "/etc/origin/master/admin.kubeconfig

/usr/local/bin/oc", "get", "deploymentconfig", "docker-registry", "--namespace", "default", "--config", "/etc/origin/master/admin.kubeconfig", "-o", "jsonpath={ .status.latestVersion }

# cockpit-ui : Create passthrough route for registry-console
/usr/local/bin/oc get route registry-console -o json -n default", "results

/usr/local/bin/oc", "new-app", "--template=registry-console", "-p", "IMAGE_PREFIX=registry.reg-aws.openshift.com:443/openshift3/", "-p", "OPENSHIFT_OAUTH_PROVIDER_URL=https://host-8-245-133.host.centralci.eng.rdu2.redhat.com:8443", "-p", "REGISTRY_HOST=docker-registry-default.apps.0210-jlr.qe.rhcloud.com", "-p", "COCKPIT_KUBE_URL=https://registry-console-default.apps.0210-jlr.qe.rhcloud.com", "--config=/tmp/openshift-ansible-Usju0f/admin.kubeconfig", "-n", "default

# web console
/usr/local/bin/oc get namespace openshift-web-console -o json

# Reconcile with the web console RBAC file
/usr/local/bin/oc process -f \"/tmp/console-ansible-yJ8L3y/console-rbac-template.yaml\" --config=/tmp/console-ansible-yJ8L3y/admin.kubeconfig

/usr/local/bin/oc process -f \"/tmp/console-ansible-yJ8L3y/console-template.yaml\

curl", "-k", "https://webconsole.openshift-web-console.svc/healthz

# service catalog
/usr/local/bin/oc get namespace kube-service-catalog -o json

/usr/local/bin/oc", "adm", "--config=/etc/origin/master/admin.kubeconfig", "ca", "create-signer-cert", "--key=/etc/origin/service-catalog/ca.key", "--cert=/etc/origin/service-catalog/ca.crt", "--serial=/etc/origin/service-catalog/apiserver.serial.txt", "--name=service-catalog-signer"

/usr/local/bin/oc adm ca create-server-cert --cert=/etc/origin/service-catalog/apiserver.crt --hostnames=apiserver.kube-service-catalog.svc,apiserver.kube-service-catalog.svc.cluster.local,apiserver.kube-service-catalog --key=/etc/origin/service-catalog/apiserver.key --overwrite=True --signer-cert=/etc/origin/service-catalog/ca.crt --signer-key=/etc/origin/service-catalog/ca.key --signer-serial=/etc/origin/service-catalog/apiserver.serial.txt

/usr/local/bin/oc secrets new apiserver-ssl tls.crt=/etc/origin/service-catalog/apiserver.crt tls.key=/etc/origin/service-catalog/apiserver.key -n kube-service-catalog

/usr/local/bin/oc secrets new service-catalog-ssl tls.crt=/etc/origin/service-catalog/apiserver.crt -n kube-service-catalog"

/usr/local/bin/oc get apiservices.apiregistration.k8s.io v1beta1.servicecatalog.k8s.io -o json -n kube-service-catalog

/usr/local/bin/oc get template service-catalog-role-bindings -o json -n kube-service-catalog

/usr/local/bin/oc create -f /tmp/servicecatalog-serviceclass-viewer-Dv8mTe -n kube-service-catalog"

/usr/local/bin/oc auth reconcile --config=/etc/origin/master/admin.kubeconfig -f /tmp/openshift-service-catalog-ansible-VeclxP

/usr/local/bin/oc adm policy add-scc-to-user hostmount-anyuid system:serviceaccount:kube-service-catalog:service-catalog-apiserver -n kube-service-catalog

/usr/local/bin/oc adm policy add-cluster-role-to-user admin system:serviceaccount:kube-service-catalog:default -n kube-service-catalog

/usr/local/bin/oc get daemonset apiserver -o json -n kube-service-catalog

/usr/local/bin/oc get service apiserver -o json -n kube-service-catalog
/usr/local/bin/oc get route apiserver -o json -n kube-service-catalog

/usr/local/bin/oc get daemonset controller-manager -o json -n kube-service-catalog

curl", "-k", "https://apiserver.kube-service-catalog.svc/healthz


