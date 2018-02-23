```
tar", "-C", "/home/slave5/workspace/Launch-Environment-Flexy/private-openshift-ansible/roles/openshift_examples/files/examples/v3.9/", "-cvf", "/tmp/openshift-ansible-t0KHj2n/openshift-examples.tar

mkdir -p /etc/origin/examples
/usr/bin/gtar", "--extract", "-C", "/etc/origin/examples/", "-f", "/root/.ansible/tmp/ansible-tmp-1518242943.03-53755578098376/source

/usr/local/bin/oc", "create", "--config=/etc/origin/master/admin.kubeconfig", "-n", "openshift", "-f", "/etc/origin/examples/image-streams/image-streams-rhel7.json
/usr/local/bin/oc", "create", "--config=/etc/origin/master/admin.kubeconfig", "-n", "openshift", "-f", "/etc/origin/examples/image-streams/dotnet_imagestreams.json
/usr/local/bin/oc", "create", "--config=/etc/origin/master/admin.kubeconfig", "-n", "openshift", "-f", "/etc/origin/examples/db-templates
rm /etc/origin/examples/quickstart-templates/nodejs.json /etc/origin/examples/quickstart-templates/cakephp.json /etc/origin/examples/quickstart-templates/dancer.json /etc/origin/examples/quickstart-templates/django.json

/usr/local/bin/oc", "create", "--config=/etc/origin/master/admin.kubeconfig", "-n", "openshift", "-f", "/etc/origin/examples/quickstart-templates
rm /etc/origin/examples/xpaas-templates/sso70-basic.json
/usr/local/bin/oc", "create", "--config=/etc/origin/master/admin.kubeconfig", "-n", "openshift", "-f", "/etc/origin/examples/xpaas-streams/
/usr/local/bin/oc", "create", "--config=/etc/origin/master/admin.kubeconfig", "-n", "openshift", "-f", "/etc/origin/examples/xpaas-templates

/usr/local/bin/oc", "create", "-f", "/etc/origin/hosted", "--config=/tmp/openshift-ansible-TvrXZs/admin.kubeconfig", "-n", "openshift
/usr/local/bin/oc get namespace management-infra -o json
/usr/local/bin/oc get sa management-admin -o json -n management-infra
/usr/local/bin/oc get sa inspector-admin -o json -n management-infra
/usr/local/bin/oc get clusterrole management-infra-admin -o json
/usr/local/bin/oc get clusterrole hawkular-metrics-admin -o json

/usr/local/bin/oc adm policy add-role-to-user admin management-admin -n management-infra
/usr/local/bin/oc adm policy add-role-to-user admin system:serviceaccount:management-infra:management-admin -n management-infra
/usr/local/bin/oc adm policy add-cluster-role-to-user cluster-reader system:serviceaccount:management-infra:management-admin -n management-infra
/usr/local/bin/oc adm policy add-scc-to-user privileged system:serviceaccount:management-infra:management-admin -n management-infra
/usr/local/bin/oc adm policy add-cluster-role-to-user system:image-puller system:serviceaccount:management-infra:inspector-admin -n management-infra

/usr/local/bin/oc adm policy add-cluster-role-to-user system:image-auditor system:serviceaccount:management-infra:management-admin -n management-infra

```
