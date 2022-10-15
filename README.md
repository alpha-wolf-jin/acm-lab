# acm-lab

```
$ git config --global credential.helper 'cache --timeout 72000'

$ git commit -a -m "update READDME"; git push -u origin main

```
#  Installing the RHACM Operator and importing clusters

## Create a project named open-cluster-management
```
$ oc new-project open-cluster-management
```

## Create an operator group
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

## Create a subscription to the Advanced Cluster Management for Kubernetes operator
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

## Watch the status of the available ClusterServiceVersion objects to verify the installation status

```
$ oc get csv
NAME                                 DISPLAY                                      VERSION   REPLACES                             PHASE
advanced-cluster-management.v2.4.2   Advanced Cluster Management for Kubernetes   2.4.2     advanced-cluster-management.v2.4.1   Succeeded


```

## Create the RHACM MultiClusterHub object

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
