# acm-lab

```
$ git config --global credential.helper 'cache --timeout 72000'

$ git commit -a -m "update READDME"; git push -u origin main

$ oc get clusterversion
NAME      VERSION   AVAILABLE   PROGRESSING   SINCE   STATUS
version   4.10.3    True        False         154d    Cluster version is 4.10.3

```
#  Installing the RHACM Operator and importing clusters

### Create a project named open-cluster-management
```
$ oc login -u admin -p redhat \
  https://api.ocp4.example.com:6443

$ oc new-project open-cluster-management
```

### Create an operator group
```
$ cat operator-group.yaml 
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: acm-operator-group
spec:
  targetNamespaces:
  - open-cluster-management

$ oc create -f operator-group.yaml

```

### Create a subscription to the Advanced Cluster Management for Kubernetes operator
```
$ cat subscription.yaml 
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: acm-operator-subscription
spec:
  sourceNamespace: openshift-marketplace
  source: do480-catalog
  channel: release-2.4
  installPlanApproval: Manual
  name: advanced-cluster-management

$ oc create -f subscription.yaml

```

## Approve the installation of the operator

```
$ oc get installplan
NAME            CSV                                  APPROVAL   APPROVED
install-xztxr   advanced-cluster-management.v2.4.2   Manual     false

$ oc patch installplan install-xztxr --type merge --patch '{"spec":{"approved":true}}'

```

### Watch the status of the available ClusterServiceVersion objects to verify the installation status

```
$ oc get csv
NAME                                 DISPLAY                                      VERSION   REPLACES                             PHASE
advanced-cluster-management.v2.4.2   Advanced Cluster Management for Kubernetes   2.4.2     advanced-cluster-management.v2.4.1   Succeeded


```

### Create the RHACM MultiClusterHub object

```
$ cat mch.yaml
apiVersion: operator.open-cluster-management.io/v1
kind: MultiClusterHub
metadata:
  name: multiclusterhub
  namespace: open-cluster-management
spec: {}

$ oc create -f mch.yaml

$ oc get multiclusterhub
NAME              STATUS       AGE
multiclusterhub   Installing   31s

$ oc get multiclusterhub
NAME              STATUS    AGE
multiclusterhub   Running   4m26s

```

## Prepare RHACM to import a cluster using the name managed-cluster

### In the hub cluster, create the managed-cluster namespace
```
$ oc new-project managed-cluster

```

### Label the namespace with the cluster name used by RHACM

```
$ oc label namespace managed-cluster cluster.open-cluster-management.io/managedCluster=managed-cluster

```

### Create the ManagedCluster object 

```
$ cat mngcluster.yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: managed-cluster
spec:
  hubAcceptsClient: true

$ oc create -f mngcluster.yaml

```

### Create a klusterlet add-on configuration

```
$ cat klusterlet.yaml
apiVersion: agent.open-cluster-management.io/v1
kind: KlusterletAddonConfig
metadata:
  name: managed-cluster
  namespace: managed-cluster
spec:
  clusterName: managed-cluster
  clusterNamespace: managed-cluster
  applicationManager:
    enabled: true
  certPolicyController:
    enabled: true
  clusterLabels:
    cloud: auto-detect
    vendor: auto-detect
  iamPolicyController:
    enabled: true
  policyController:
    enabled: true
  searchCollector:
    enabled: true
  version: 2.4.2

$ oc create -f klusterlet.yaml

```

**Next, the ManagedCluster-Import-Controller generates a secret named managed-cluster-import. This secret contains the import.yaml file that contains the secret used in the imported cluster.

## Generate the files to import the managed-cluster into RHACM

### Obtain the necessary files to import the managed-cluster cluster
```
$ oc get secret managed-cluster-import \
   -n managed-cluster -o jsonpath={.data.crds\\.yaml} | base64 \
   --decode > klusterlet-crd.yaml

$ oc get secret managed-cluster-import \
   -n managed-cluster -o jsonpath={.data.import\\.yaml} | base64 \
   --decode > import.yaml

```

## Import the RHCOP cluster available in the lab environment, ocp4-mng.example.com, into RHACM. Use managed-cluster as the name of the imported cluster

### Use the terminal to log in to the ocp4-mng cluster as the admin user. The API server address is https://api.ocp4-mng.example.com:6443
```
$ oc login -u admin -p redhat \
  https://api.ocp4-mng.example.com:6443
```

### Use the klusterlet-crd.yaml file to create the klusterlet custom resource definition.
```
$ oc create -f klusterlet-crd.yaml

```

### use the import.yaml file to create the rest of the resources necessary to import the cluster into RHACM.

```
$ oc create -f import.yaml
```

### Verify the status of the pods running in the open-cluster-management-agent namespace.

```
$ oc get pod -n open-cluster-management-agent
NAME                                             READY   STATUS    RESTARTS      AGE
klusterlet-5984dd44bc-lm6zr                      1/1     Running   0             68s
klusterlet-registration-agent-867fc6d5f8-65hrd   1/1     Running   0             52s
klusterlet-registration-agent-867fc6d5f8-v4692   1/1     Running   0             52s
klusterlet-registration-agent-867fc6d5f8-vdtq5   1/1     Running   0             52s
klusterlet-work-agent-66f9556c87-dxkwx           1/1     Running   0             51s
klusterlet-work-agent-66f9556c87-hr7fb           1/1     Running   0             51s
klusterlet-work-agent-66f9556c87-kzxqc           1/1     Running   1 (42s ago)   51s

```

### Finally, use the watch command to validate the status of the agent pods running in the open-cluster-management-agent-addon namespace.

```
$ oc get pod -n open-cluster-management-agent-addon
NAME                                                         READY   STATUS    RESTARTS   AGE
klusterlet-addon-appmgr-58c6cd799c-68qql                     1/1     Running   0          3m42s
klusterlet-addon-certpolicyctrl-6874c9cb65-77l9f             1/1     Running   0          3m42s
klusterlet-addon-iampolicyctrl-74946dc857-pbclm              1/1     Running   0          3m42s
klusterlet-addon-operator-c5559f597-jztjh                    1/1     Running   0          4m11s
klusterlet-addon-policyctrl-config-policy-58c58f97d9-ghc2n   1/1     Running   0          3m42s
klusterlet-addon-policyctrl-framework-76f7469d7f-h4twp       3/3     Running   0          3m42s
klusterlet-addon-search-56db9757c4-m7dvd                     1/1     Running   0          3m42s
klusterlet-addon-workmgr-b855b98bb-4df8w                     1/1     Running   0          3m42s

```

**It takes around two minutes before the pods in the open-cluster-management-agent-addon start.

## Log back into the hub cluster and verify the managed-cluster status

### Log back into the hub cluster

```
$ oc login -u admin -p redhat \
  https://api.ocp4.example.com:6443
```

### Verify the managed-cluster status

```
$ oc get managedcluster
NAME              HUB ACCEPTED   MANAGED CLUSTER URLS                    JOINED   AVAILABLE   AGE
local-cluster     true           https://api.ocp4.example.com:6443       True     True        40m
managed-cluster   true           https://api.ocp4-mng.example.com:6443   True     True        28m

```
**The JOINED and AVAILABLE must have a True status.

# Deploying and Managing Policies with RHACM

**Log in to the Hub OpenShift cluster and create a policy-governance project.
```
$ oc login -u admin -p redhat \
  https://api.ocp4.example.com:6443

$ oc new-project policy-governance

```

**Log in to the RHACM web console and create the certificate policy.

open Firefox and access https://multicloud-console.apps.ocp4.example.com.

Create the certificate policy with the following parameters for openshift-console and openshift-ingress namespace:

Field name	Value
Name	policy-certificatepolicy
Namespace	policy-governance
Specifications	CertificatePolicy - Certificate management expiration
Remediation	Inform

**Click `Governance` on the left pane to navigate to the governance dashboard. Then, click `Create policy`. The Create policy page displays.

**On the right side of the Create policy page, edit the YAML code as follows:
```
spec:
  remediationAction: inform
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: CertificatePolicy
        metadata:
          name: policy-certificatepolicy-cert-expiration
        spec:
          namespaceSelector:
            include:
              - default
              - openshift-console
              - openshift-ingress
            exclude:
              - kube-*
          remediationAction: inform
          severity: low
          minimumDuration: 300h

```


```
oc login -u admin -p redhat \
  https://api.ocp4-mng.example.com:6443

$ ./renew_wildcard.sh

```

# Deploying and Configuring the Compliance Operator for Multiple Clusters Using RHACM

- Install the compliance operator from the RHACM governance dashboard.
- Deploy the E8-scan policy.
- Check the compliant scan result from E8 scan.

```
$ oc login -u admin -p redhat \
  https://api.ocp4.example.com:6443

$ oc new-project policy-compliance

```

** In the left pane, click Governance to display the governance dashboard, and then click Create policy to create the policy. The Create policy page is displayed.
