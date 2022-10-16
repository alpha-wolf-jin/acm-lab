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

Field name	Value
Name	policy-complianceoperator
Namespace	policy-compliance
Specifications	ComplianceOperator - Install the Compliance operator
Cluster selector	location: "APAC"
Remediation	Inform

```
$ git clone   https://github.com/stolostron/policy-collection.gi

$ cd policy-collection

$ cat policy-compliance-operator-e8-scan.yaml
# This policy assumes that the Compliance Operator is installed and running on
# the managed clusters. It deploys a scan that will check the master and worker
# nodes and verifies for compliance with the E8 (Essential 8) security profile.
apiVersion: policy.open-cluster-management.io/v1
kind: Policy
metadata:
  name: policy-e8-scan
  annotations:
    policy.open-cluster-management.io/standards: NIST SP 800-53
    policy.open-cluster-management.io/categories: CM Configuration Management
    policy.open-cluster-management.io/controls: CM-6 Configuration Settings
spec:
  remediationAction:  enfore
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: compliance-e8-scan
        spec:
          remediationAction: inform
          severity: high
          object-templates:
            - complianceType: musthave # this template creates ScanSettingBinding:e8
              objectDefinition:
                apiVersion: compliance.openshift.io/v1alpha1
                kind: ScanSettingBinding
                metadata:
                  name: e8 
                  namespace: openshift-compliance
                profiles:
                - apiGroup: compliance.openshift.io/v1alpha1
                  kind: Profile
                  name: ocp4-e8
                - apiGroup: compliance.openshift.io/v1alpha1
                  kind: Profile
                  name: rhcos4-e8
                settingsRef:
                  apiGroup: compliance.openshift.io/v1alpha1
                  kind: ScanSetting
                  name: default
    - objectDefinition: 
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: compliance-suite-e8
        spec:
          remediationAction: inform
          severity: high
          object-templates:
            - complianceType: musthave # this template checks if scan has completed by checking the status field
              objectDefinition:
                apiVersion: compliance.openshift.io/v1alpha1
                kind: ComplianceSuite
                metadata:
                  name: e8
                  namespace: openshift-compliance
                status:
                  phase: DONE
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: compliance-suite-e8-results
        spec:
          remediationAction: inform
          severity: high
          object-templates:
            - complianceType: mustnothave # this template reports the results for scan suite: e8 by looking at ComplianceCheckResult CRs
              objectDefinition:
                apiVersion: compliance.openshift.io/v1alpha1
                kind: ComplianceCheckResult
                metadata:
                  namespace: openshift-compliance
                  labels:
                    compliance.openshift.io/check-status: FAIL
                    compliance.openshift.io/suite: e8
---
apiVersion: policy.open-cluster-management.io/v1
kind: PlacementBinding
metadata:
  name: binding-policy-e8-scan
placementRef:
  name: placement-policy-e8-scan
  kind: PlacementRule
  apiGroup: apps.open-cluster-management.io
subjects:
- name: policy-e8-scan
  kind: Policy
  apiGroup: policy.open-cluster-management.io
---
apiVersion: apps.open-cluster-management.io/v1
kind: PlacementRule
metadata:
  name: placement-policy-e8-scan
spec:
  clusterConditions:
  - status: "True"
    type: ManagedClusterConditionAvailable
  clusterSelector:
    matchExpressions:
      - {key: location, operator: In, values: ["APAC"]}

$ oc login -u admin -p redhat \
  https://api.ocp4.example.com:6443

$ oc project policy-compliance

$ oc create -f \
policy-compliance-operator-e8-scan.yaml

 
```

In the left pane, click Governance. The dashboard shows the policy-e8-scan policy. Click policy-e8-scan and then click Clusters to check the status.

The clusters tab shows three resources: compliance-e8-scan, compliance-suite-e8, and compliance-suite-e8-result. Compliance-suite-e8-result shows Not compliant.

# Deploying and Managing Policies with RHACM

In this lab, you create a policy to ensure that the test namespace does not exist in the production environment. 

You also create an Identity and Access Management (IAM) policy to limit the number of cluster administrators to two for all clusters.

```
$ oc login -u admin -p redhat \
  https://api.ocp4.example.com:6443

$ oc new-project policy-review
```

In the left pane, click Governance to display the Governance dashboard, and then click Create policy to create the policy. The Create policy page is displayed.

Complete the page with the following details, leaving the other fields unchanged. Do not click Create yet.

Field name	Value
Name	policy-namespace
Namespace	policy-review
Specifications	Namespace - Must have namespace 'prod'
Cluster selector	environment: "production"
Remediation	Enforce

Use the YAML editor on the right of the Create policy page to make the following edits to the YAML file:

```
spec:
  remediationAction: enforce
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: ConfigurationPolicy
        metadata:
          name: policy-namespace-prod-ns
        spec:
          remediationAction: inform
          severity: low
          namespaceSelector:
            exclude:
              - kube-*
            include:
              - default
          object-templates:
            - complianceType: mustnothave
              objectDefinition:
                kind: Namespace
                apiVersion: v1
                metadata:
                  name: test
```

Click Create.

**You also create an Identity and Access Management (IAM) policy to limit the number of cluster administrators to two for all clusters.

In the left pane, click Governance to display the Governance dashboard, and then click Create policy to create the policy. The Create policy page is displayed.

Complete the page with the following details, leaving the other fields unchanged. Do not click Create yet.

Field name	Value
Name	policy-iampolicy
Namespace`	policy-review
Specifications	IamPolicy - Limit clusteradmin roles

Use the YAML editor on the right of the Create policy page to make the following edit to the YAML file:

```
spec:
  remediationAction: inform
  disabled: false
  policy-templates:
    - objectDefinition:
        apiVersion: policy.open-cluster-management.io/v1
        kind: IamPolicy
        metadata:
          name: policy-iampolicy-limit-clusteradmin
        spec:
          severity: medium
          namespaceSelector:
            include:
              - '*'
            exclude:
              - kube-*
              - openshift-*
          remediationAction: inform
          maxClusterRoleBindingUsers: 2
```

Click Create.

# Installing and Configuring Red Hat Quay

In this lab, you use the command line to install Red Quay and push a container image by using a robot account.

1. Log in to the ocp4 cluster as the admin user with the redhat password. The API server address is https://api.ocp4.example.com:6443.

```
$ oc login -u admin -p redhat \
  https://api.ocp4.example.com:6443
```

2. Install the Quay operator cluster-wide. Use the do480-catalog offline catalog. The lab script includes a DO480/solutions/quay-review/subscription.yaml solution file.

```
$ cat subscription.yaml 
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: quay-operator
  namespace: openshift-operators
spec:
  sourceNamespace: openshift-marketplace
  source: do480-catalog
  channel: stable-3.6
  installPlanApproval: Automatic
  name: quay-operator

$ oc create -f subscription.yaml

$ oc get deployment -n openshift-operators
NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
quay-operator.v3.6.4   1/1     1            1           24s

```

3. Create a registry namespace.


```
$ oc create namespace registry
```
4. Create an init-config-bundle-secret object in the registry namespace with a config.yaml key containing the Quay configuration.

Use the suggested options for automation.

- Enable the /api/v1/user/initialize Quay API endpoint to create the first user.
- Allow access to the Quay registry API without the use of an X-Requested-With header.
- Define a quayadmin super user.

```
$ cat config.yaml 
FEATURE_USER_INITIALIZE: true
BROWSER_API_CALLS_XHR_ONLY: false
SUPER_USERS:
- quayadmin
FEATURE_USER_CREATION: false

$ oc create secret generic \
  --from-file config.yaml=./config.yaml init-config-bundle-secret -n registry

```

5. Create a QuayRegistry object named central in the registry namespace. The QuayRegistry object references the init-config-bundle-secret object containing the Quay configuration.

Do not include Clair, the horizontal pod autoscaler, or the mirror feature.

The lab script includes a DO480/solutions/quay-review/quay-registry.yaml solution file.

```
$ cat quay-registry.yaml 
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: central
  namespace: registry
spec:
  configBundleSecret: init-config-bundle-secret
  components:
    - kind: clair
      managed: false
    - kind: horizontalpodautoscaler
      managed: false
    - kind: mirror
      managed: false

$ oc create -f quay-registry.yaml

$ oc get deployment -n registry
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
central-quay-app             2/2     2            2           2m52s
central-quay-config-editor   1/1     1            1           2m52s
central-quay-database        1/1     1            1           2m52s
central-quay-redis           1/1     1            1           2m52s
```

6. Create the quayadmin user and obtain an access token. The password must be at least 8 characters and contain no whitespace. The lab script includes a DO480/solutions/quay-review/quayadmin-user.json solution file.

Store the token in a /home/student/quayadmin_token file. This step is necessary so that the lab script can use the API to grade your work.
```
$ cat quayadmin-user.json 
{
    "username": "quayadmin",
    "password":"redhat123",
    "email": "quayadmin@example.com",
    "access_token": true
}


```

Use the curl command to post the user definition in JSON format to the initialize user endpoint. Note the returned access token for a later step.

```
$ curl -X POST -k \
>   https://central-quay-registry.apps.ocp4.example.com/api/v1/user/initialize \
>   --header "Content-Type: application/json" \
>   --data @quayadmin-user.json
{"access_token":"PQNZDWG9PC4KRB5C5JPETJKDELY132BGU7OMTQP9","email":"quayadmin@example.com","encrypted_password":"3eIgOiJZeGzkZByboAYzaicIriMfQNCEqBIe0G9ZLb7rc42f/qdOTmFrXcCIz6qW","username":"quayadmin"}

$ echo PQNZDWG9PC4KRB5C5JPETJKDELY132BGU7OMTQP9 >quayadmin_token

$ QUAYADMIN_TOKEN=PQNZDWG9PC4KRB5C5JPETJKDELY132BGU7OMTQP9

```

7. Create the finance organization by using the API. The lab script includes a DO480/solutions/quay-review/finance-organization.json solution file.

```
$ cat finance-organization.json 
{
    "name": "finance",
    "email": "finance@example.com"
}

$ curl -X POST -k \
  https://central-quay-registry.apps.ocp4.example.com/api/v1/organization/ \
  --header "Content-Type: application/json" \
  --header "Authorization: Bearer PQNZDWG9PC4KRB5C5JPETJKDELY132BGU7OMTQP9" \
  --data @finance-organization.json
```

8. Create the budget-app-dev image repository by using the API. The lab script includes a DO480/solutions/quay-review/budget-app-dev-repository.json solution file.

```
$ cat budget-app-dev-repository.json 
{
    "namespace": "finance",
    "repository": "budget-app-dev",
    "description": "development",
    "visibility": "private"
}

$ curl -X POST -k \
  https://central-quay-registry.apps.ocp4.example.com/api/v1/repository \
  --header "Content-Type: application/json" \
  --header "Authorization: Bearer PQNZDWG9PC4KRB5C5JPETJKDELY132BGU7OMTQP9" \
  --data @budget-app-dev-repository.json
```

9. Create the deployer robot account by using the API. The lab script includes a DO480/solutions/quay-review/deployer-robot.json solution file.

```
$ cat deployer-robot.json 
{
    "description": "deployer"

$ curl -X PUT -k \
  https://central-quay-registry.apps.ocp4.example.com/api/v1/organization/finance/robots/deployer \
  --header "Content-Type: application/json" \
  --header "Authorization: Bearer PQNZDWG9PC4KRB5C5JPETJKDELY132BGU7OMTQP9" \
  --data @deployer-robot.json
{"name": "finance+deployer", "created": "Sun, 16 Oct 2022 05:03:47 -0000", "last_accessed": null, "description": "deployer", "token": "DOWMJQMRUFMUT3PF8N7B6RII7Y6Q5CTFX5VCL3MH4HSLR2NOYDGZCLV2F7U1VFAF", "unstructured_metadata": null}

```

10. Grant permissions to the deployer robot account over the budget-app-dev image repository.

Use the curl command to grant the deployer robot account the write role on the budget-app-dev repository in the finance organization. Replace the token for the quayadmin user with the token that you obtained in a previous step.

```
$ curl -X PUT -k \
  https://central-quay-registry.apps.ocp4.example.com/api/v1/repository/finance/budget-app-dev/permissions/user/finance+deployer \
  --header "Content-Type: application/json" \
  --header "Authorization: Bearer PQNZDWG9PC4KRB5C5JPETJKDELY132BGU7OMTQP9" \
  --data '{"role": "write"}'
{"role": "write", "name": "finance+deployer", "is_robot": true, "avatar": {"name": "finance+deployer", "hash": "9a16cc387a65db286096651cbd5f36a29a3513b2da13c7d5bb82553769c5f574", "color": "#b5cf6b", "kind": "robot"}, "is_org_member": true}

```

11. Create a budget-app:development image by using the podman command. You can create an image based on the quay.io/podman/hello image.

Create a Containerfile file containing a minimal image definition. Your file should match the following example.

```
$ mkdir tmp
$ cd tmp
$ vim Containerfile
$ cat Containerfile
FROM quay.io/podman/hello
$ podman build . -t budget-app:development
```

12. As the deployer robot account, push the image to the budget-app-dev image repository in the finance organization.

```
$ podman login -u=finance+deployer \
  -p=DOWMJQMRUFMUT3PF8N7B6RII7Y6Q5CTFX5VCL3MH4HSLR2NOYDGZCLV2F7U1VFAF \
  central-quay-registry.apps.ocp4.example.com

$ podman push localhost/budget-app:development \
  central-quay-registry.apps.ocp4.example.com/finance/budget-app-dev
```
