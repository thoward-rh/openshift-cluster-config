# Openshift Cluster Config

This repo sets up OpenShift with Day 2 things via Argo CD. It also uses keycloak for Authentication with OpenShift.


## Installing ArgoCD

> :warning: This is based on OpenShift 4.8 deploying Argo CD v2.x using OpenShift GitOps v1.1.x.

First apply the `Subscription` manifest.

```shell
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Wait for the pods to appear

```shell
echo "Still waiting" ; until [[ $(oc get pods -n openshift-gitops -o name  | wc -l) -ne 0 ]]; do echo -n "."; sleep 3; done; echo ""
```

Give the serviceAccount permission to admin the cluster.

```shell
oc adm policy add-cluster-role-to-user cluster-admin -z openshift-gitops-argocd-application-controller -n openshift-gitops
```

Patch the manifest in order to use OpenShift SSO.

```shell
oc patch argocd openshift-gitops -n openshift-gitops --type=merge -p='{"spec": {"sso": {"provider": "keycloak"}}}'
```

Wait for the deployment rollout

```shell
oc rollout status deploy/openshift-gitops-server -n openshift-gitops
```

To view the policies set up by the patch

```shell
oc get argocd openshift-gitops -n openshift-gitops -o yaml | grep -B 1 -A 6 -w 'defaultPolicy: ""'
```

To get the route

```shell
oc get routes openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}{"\n"}'
```

To get the admin password, ( but you can also login using your openshift account )

```shell
oc  get secret openshift-gitops-cluster -n openshift-gitops -ojsonpath='{.data.admin\.password}' | base64 -d ; echo
```

## Deploying this Repo

To configure your cluster to this repo run

```
oc apply -k https://github.com/thoward-rh/openshift-cluster-config/cluster-config/config/overlays/default
```

This will configure your server with the following.

Cluster Configurations:
* HTPassword Authentication
  * Three users: `ocp-admin`,`ocp-developer`, and `ocp-marketing`
* Three Groups created
  * `admins`
    * `ocp-admin` is part of `admins`
  * `developer`
    * `ocp-developer` is part of `developer`
  * `marketing`
    * `ocp-marketing` is part of `marketing`
* ClusterRole/Role Bindings setup
  * `admins` group has `cluster-admin` on OpenShift
  * The `developer` group has `edit` on the `pricelist` namespace on OpenShift
* Container Security Operator installed
* Pipelines Operator installed
* Sealed Secrets installed

Application Deployments:
* Deploy Pricelist in an ArgoCD project called `pricelist`
  * One `application` Consisting of...
    * Frontend Web Application
    * Backend Database store
    * Job that creates database tables and the such
  * The manifests for this app lives in my [gitops example repo](https://github.com/thoward-rh/gitops-examples)

ArgoCD Configurations
* ArgoCD is integrated with the OpenShift oAuth
<!-- * RBAC Policy
  * The `admins` OpenShift group is set up as ArgoCD admins
  * The `developer` OpenShift group is set up as ArgoCD users
  * ArgoCD admins can see and sync all ArgoCD Applications -->
* The `cluster-config` ArgoCD project has all "cluster wide" configurations
  * Can only be seen/synced by ArgoCD admins
* The `pricelist` ArgoCD project has all appliaction components to run the [Pricelist](https://github.com/thoward-rh/openshift-cluster-config) application
  * Can be seen/synced by ArgoCD admins or ArgoCD users
* Autosync is turned on

Admin password is myadminpassword
developer password is mydeveloperpassword
marketing password is mymarketingpassword
# How do I make changes

You don't, it's GitOps!

Jokes aside, the idea is to manage your cluster by pull request to the right repo. In a lot of instances, that means many PRs to many repos!
