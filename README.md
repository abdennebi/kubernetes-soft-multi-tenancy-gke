# Kubernetes Soft Multi-tenancy on Google Kubernetes Engine (GKE)

## GKE Setup

- A GKE cluster with **Network Policy** (calico) enabled.

## Create a Custom Role to limit permissions for a user

- The less permissive IAM Role to access a cluster is ``roles/container.viewer`` but it remains too permissive (a user can still read the other namespaces content). Run the following command to view all the permissions tied to the *viewer* role :
    ``gcloud iam roles describe roles/container.viewer``.

- To overcome this, we will create an **IAM Custome Role** to allow a user to just having access to a cluster then we delegate all fine-grained permissions to the Kubernetes RBAC system (via Roles and RoleBindings).

````bash
gcloud iam roles create custom.container.minimal \
--title "Kubernetes Engine Minimal (Custom)" \
--permissions "container.apiServices.get,container.apiServices.list,container.clusters.get" \
--description "Minimum access to a cluster" \
--project ${PROJECT}
````

To verify,

````bash
gcloud iam roles describe  custom.container.minimal --project $PROJECT
````

Click [here](https://cloud.google.com/iam/docs/creating-custom-roles#iam-custom-roles-testable-permissions-gcloud) to get detailed informations about custom roles.

## Create a tenant

By tenant, we mean a namespace on wich we autorized users to use and for which we prohibited communications with the other namespaces. Although, it is quite possible to relax the rule and allow communications betwwen certain namespaces.

*The following commands use ``envsubst`` which is not installed by defaut in some environements. If this commmand is not installed on your system follow the installation guide in the end of the document.*

1- Create a namespace

````bash
cat namespace.yaml | NAMESPACE=${NAMESPACE} envsubst | kubectl apply -f -
````

````yaml
kind: Namespace
apiVersion: v1
metadata:
  name: $NAMESPACE
````

2- Deny ingress and egress communications with other namespaces

````bash
cat deny-from-other-namespaces.yaml | NAMESPACE=${NAMESPACE} envsubst | kubectl apply -f -
````

````yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  namespace: $NAMESPACE
  name: deny-from-other-namespaces
spec:
  podSelector:
    matchLabels:
  ingress:
  - from:
    - podSelector: {}
````

3- Make an user as the admin of the namespace

````bash
cat allow-user-on-namespace.yaml | NAMESPACE=${NAMESPACE} USERNAME=${USERNAME} EMAIL=${EMAIL} envsubst | kubectl apply -f -
````

````yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: $NAMESPACE-admin
  namespace: $NAMESPACE
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["*"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources:
  - jobs
  - cronjobs
  verbs: ["*"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: $USERNAME-$NAMESPACE-admin
  namespace: $NAMESPACE
subjects:
- kind: User
  name: $EMAIL
  namespace: $NAMESPACE
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: $NAMESPACE-admin
````

## Access to the cluster

### Project Owner: Give the user access to the cluster

- As project ``Owner`` give the user the newly crafted role:

    ````bash
    gcloud projects add-iam-policy-binding ${PROJECT_ID} --member="user:${EMAIL}" --role="projects/${PROJECT_ID}/roles/custom.container.minimal"
    ````

- Send these informations to the user : The GCP project id ``${PROJECT_ID}`` , the cluster name ``${CLUSTER_NAME}`` and the cluster zone ${ZONE} (or region if you use multi-zonal cluster).

### User: Get credentials

*Make sure you are using the account of the user for whom you want to give an access to a specified namespace.*

````bash
gcloud config set account ${EMAIL}
gcloud container clusters get-credentials ${CLUSTER_NAME} --project ${PROJECT_ID} --zone ${ZONE}

Fetching cluster endpoint and auth data.
kubeconfig entry generated for ${CLUSTER_NAME}.
````

### User: Test the access

- Try to list the project's kubernetes clusters.

    ````bash
    gcloud container clusters list  --project ${PROJECT_ID}
    ERROR: (gcloud.container.clusters.list) ResponseError: code=403, message=Required "container.clusters.list" permission(s) for "projects/${PROJECT_ID}".
    ````
    To let the user see the clusters, add the permission ``container.clusters.list`` to the custom role.

- Get the description of the cluster

    ````bash
    gcloud container clusters describe ${CLUSTER_NAME}  --project ${PROJECT_ID} --zone ${ZONE}

    addonsConfig:
    httpLoadBalancing: {}
    kubernetesDashboard: {}
    networkPolicyConfig: {}
    clusterIpv4Cidr: 10.0.0.0/16
    createTime: '2018-10-13T17:17:06+00:00'
    currentMasterVersion: 1.10.7-gke.6
    currentNodeCount: 3
    currentNodeVersion: 1.10.7-gke.6
    ...
    ````

- Try to list the namespaces

    ````bash
    kubectl get namespaces

    Error from server (Forbidden): namespaces is forbidden: User "${EMAIL}" cannot list namespaces at the cluster scope: Required "container.namespaces.list" permission.
    ````

- Try to get informations on ``kube-system`` namespace:

    ````bash
    kubectl describe namespace kube-system
    Error from server (Forbidden): namespaces "kube-system" is forbidden: User "${EMAIL}" cannot get namespaces in the namespace "kube-system": Required "container.namespaces.get" permission.
    ````

    Even ``default`` namespace is not accessible.

- Finaly, get information on authorized namespace

    ````bash
    kubectl describe namespace ${NAMESPACE}
    Name:         ${NAMESPACE}
    Labels:       <none>
    Annotations:  <none>
    Status:       Active

    No resource quota.

    No resource limits.
    ````
- From now, the user can set the defaut namespace to the assigned one:

    ````bash
    kubectl config set-context $(kubectl config current-context) --namespace=${NAMESPACE}
    ````

## envsubst installation

To verify if ``envsubst`` is installed : ``which envsubst``.

To install :

Debian:

````bash
sudo apt install gettext
````

MacOs:

````bash
brew install gettext
brew link --force gettext
````
