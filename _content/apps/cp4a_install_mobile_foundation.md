---
title: Install Mobile Foundation
weight: 475
---

## Introduction

IBM Mobile Foundation is an integrated platform that helps you extend your business to mobile devices.

Mobile Foundation includes a comprehensive development environment, mobile-optimized runtime middleware, a private enterprise application store, and an integrated management and analytics console, all supported by various security mechanisms.

For more information about Mobile Foundation please visit the [Product Overview](http://mobilefirstplatform.ibmcloud.com/tutorials/en/foundation/8.0/product-overview/) page.

## Steps Overview

0. Create db2 instance with database to host data for Mobile foundation. (Prerequisites)

1. Set up images and secrets

2. Create openshift project

3. Prepare to install Mobile foundation components

4. Deploy Mobile foundation components

5. Uninstall Mobile Foundation

### Prerequisites

The [prerequisites](http://mobilefirstplatform.ibmcloud.com/tutorials/en/foundation/8.0/product-overview/requirements/) include a configured database. The reccomended database is DB2, the setup guide for this can be found [here](https://www.youtube.com/watch?time_continue=28&v=k1Wj2Sc5Ing&feature=emb_logo).

Other databases such as MySQL can also be used but must be configured using the [guide](https://mobilefirstplatform.ibmcloud.com/tutorials/ru/foundation/8.0/installation-configuration/production/prod-env/databases/).

IBM Cloud CLI, [Installation guide](https://cloud.ibm.com/docs/cli?topic=cloud-cli-install-ibmcloud-cli).

OpenShift CLI, [Installation guide](https://docs.openshift.com/container-platform/4.2/cli_reference/openshift_cli/getting-started-cli.html).

Download Mobile foundation installation files (part number - CC3FDEN)

### Installation

## Step 1 - Setup images and secrets

Login to IBM Cloud

```
ibmcloud login --sso
```

Login to the Openshift cluster using the OC cli

```
oc login
```

Login into OpenShift’s internal docker registry

```
oc create route reencrypt --service=docker-registry -n default
oc get route docker-registry -n default

docker login -u $(oc whoami) -p $(oc whoami -t) <docker-registry-url>
```

Navigate to folder containing the mobile foundation installation file and unpack into a new directory

```
mkdir mfospkg

tar xzvf MPF_V8.0_MPS_ON_ROSCP_EN.tar.gz -C mfospkg/
```

Load the IBM Mobile Foundation images locally

```
cd mfospkg/images

ls * | xargs -I{} docker load --input {}
```

Store namespace and container registry url in variables for easier use

```
export MFOS_PROJECT=<my_namespace>
export CONTAINER_REGISTRY_URL=<docker-registry-url>
```
* Note down `<my_namespace>`, as you will need to name your project using this namespace.

Push the images to the internal docker registry

```
for file in * ; do
docker tag ${file/.tar.gz/} $CONTAINER_REGISTRY_URL/$MFOS_PROJECT/${file/.tar.gz/}
docker push $CONTAINER_REGISTRY_URL/$MFOS_PROJECT/${file/.tar.gz/}
done
```

* You might encounter an issue while pushing the images where the docker push is stuck at retrying eventually resulting in 504 gateway time out [more info](https://www.ibm.com/support/knowledgecenter/en/SSCSJL_3.x/troubleshoot.html).
To fix this try scaling down the registry to 1 pod, in our case using network with lower upload speed has fixed the issue.


## Step 2 - Create openshift project

Use the namespace you defined earlier to create a new project in OpenShift

```
oc new-project <my_namespace>
```

## Step 3 - Prepare for component installation

Make sure you are in project where you want mobile foundation to be deployed

```
oc project <my_namespace>
```

Set docker image url for MF operator in deploy/operator.yaml by replacing REPO_URL

```
###############################################################################
# Licensed Materials - Property of IBM.
# Copyright IBM Corporation 2019. All Rights Reserved.
# U.S. Government Users Restricted Rights - Use, duplication or disclosure
# restricted by GSA ADP Schedule Contract with IBM Corp.
#
# Contributors:
# IBM Corporation - initial API and implementation
###############################################################################
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mf-operator
  labels:
    app.kubernetes.io/name: mf-operator
    app.kubernetes.io/instance: mf-instance
    app.kubernetes.io/managed-by: helm
    helm.sh/chart: mf-operator-1.0.1
    release: mf-operator-1.0.1
spec:
  replicas: 1
  selector:
    matchLabels:
      name: mf-operator
  template:
    metadata:
      labels:
        name: mf-operator
        app.kubernetes.io/name: mf-operator
        app.kubernetes.io/instance: mf-instance
        app.kubernetes.io/managed-by: helm
        helm.sh/chart: mf-operator-1.0.1
        release: mf-release
      annotations:
        productName: "IBM MobileFirst Platform Foundation"
        productID: "9380ea99ddde4f5f953cf773ce8e57fc"
        productVersion: "8.0"
    spec:
      serviceAccountName: mf-operator
      containers:
        - name: mf-operator
          image: docker-registry.default.svc.cluster.local:5000/<my_namespace>/mf-operator:1.0.1 # Replace this with the built image name
          imagePullPolicy: Always
          env:
            - name: WATCH_NAMESPACE
```

Set the OpenShift Project name for the cluster role binding definition in deploy/cluster_role_binding.yaml by replacing the placeholder REPLACE_NAMESPACE

```
###############################################################################
# Licensed Materials - Property of IBM.
# Copyright IBM Corporation 2019. All Rights Reserved.
# U.S. Government Users Restricted Rights - Use, duplication or disclosure
# restricted by GSA ADP Schedule Contract with IBM Corp.
#
# Contributors:
# IBM Corporation - initial API and implementation
###############################################################################
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: mf-operator
  labels:
    app.kubernetes.io/name: mf-operator
    app.kubernetes.io/instance: mf-instance
    app.kubernetes.io/managed-by: helm
    helm.sh/chart: mf-operator-1.0.1
    release: mf-operator-1.0.1
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: mf-operator
  namespace: <my_namespace>
```

Deploy the operator

```
oc create -f deploy/crds/charts_v1_mfoperator_crd.yaml
oc create -f deploy/
```

Set SCC

```
oc adm policy add-scc-to-group mf-operator system:serviceaccounts:<my_namespace>
```

Check that the MF operator was deployed
```
oc get pods

NAME                           READY     STATUS    RESTARTS   AGE
mf-operator-5db7bb7w5d-b29j7   1/1       Running   0          1m
```

Obtain credentials to access the database.

- Go to IBM cloud dashboard in your web browser
- Go to the list of resources
- Click on the db2 service under the services tab
- On the left side navigate to "Service credentials"
- Click on "new credential"
- Select a name or leave the default name and click add
- New credential appears in the list of credentials, click on view credentials to see the username and password

{%
    include figure.html
    src="/assets/img/cp4a/db2-credentials.png"
    alt="Example credentials on DB2"
    caption="Example credentials on DB2"
%}

Convert the username and password into base64 format

```
echo -n <username> | base64
echo -n <password> | base64
```

Create secret with access details for the DB2 database

```
#make new secret
vim scrt.yaml

#copy the following into scrt.yaml
apiVersion: v1
data:
    MFPF_ADMIN_DB_USERNAME: <username in base64 format>
    MFPF_ADMIN_DB_PASSWORD: <password in base 64 format>
    MFPF_RUNTIME_DB_USERNAME: <username in base64 format>
    MFPF_RUNTIME_DB_PASSWORD: <password in base 64 format>
    MFPF_PUSH_DB_USERNAME: <username in base64 format>
    MFPF_PUSH_DB_PASSWORD: <password in base 64 format>
    MFPF_APPCNTR_DB_USERNAME: <username in base64 format>
    MFPF_APPCNTR_DB_PASSWORD: <password in base 64 format>
kind: Secret
metadata:
    name: mobilefoundation-db-secret
type: Opaque

#create the secret in openshift
oc apply -f scrt.yaml
```

* Reminder, this guide is focusing on using a DB2 instance for our database, to get the credentials for your chosen databse will require further research.

(OPTIONAL make PV and PVC for analytics, see [here](http://mobilefirstplatform.ibmcloud.com/tutorials/en/foundation/8.0/ibmcloud/mobilefoundation-on-openshift/#setup-openshift-for-mf))



## Step 4 - Deploy components

- Set the docker repository url in deploy/crds/charts_v1_mfoperator_cr.yaml by replacing the placeholder REPO_URL
- Set the db2 details for mfpserver
- Set number of replicas for each service, in this case 1 replica is used for each service

```
apiVersion: mf.ibm.com/v1
kind: MFOperator
metadata:
  name: sample-mf
  labels:
    app.kubernetes.io/name: mf-operator
    app.kubernetes.io/instance: mf-instance
    app.kubernetes.io/managed-by: helm
    helm.sh/chart: mf-operator-1.0.1
    release: mf-operator-1.0.1
spec:
  ###############################################################################
  # Licensed Materials - Property of IBM.
  # Copyright IBM Corporation 2019. All Rights Reserved.
  # U.S. Government Users Restricted Rights - Use, duplication or disclosure
  # restricted by GSA ADP Schedule Contract with IBM Corp.
  #
  # Contributors:
  # IBM Corporation - initial API and implementation
  ###############################################################################

  ###############################################################################
  # Default values for MFOperator
  # This is a YAML-formatted file.
  # Declare variables to be passed into your templates.
  ###############################################################################

  ###############################################################################
  # Specify architecture (amd64, ppc64le, s390x) and weight to be  used for scheduling as follows :
  #   0 - Do not use
  #   1 - Least preferred
  #   2 - No preference
  #   3 - Most preferred
  # Note: IBM Mobile Foundation Package for Openshift currently supports only amd64 architecture.
  ###############################################################################

  ###############################################################################
  ## Global configuration
  ###############################################################################
  global:
    arch:
      amd64: "3 - Most preferred"
      ppc64le: "0 - Do not use"
      s390x: "0 - Do not use"
    image:
      pullPolicy: IfNotPresent
      pullSecret:
    ingress:
      hostname: ""
      secret: ""
      sslPassThrough: false
    https: false
    dbinit:
      enabled: true
      repository: docker-registry.default.svc.cluster.local:5000/<my_namespace>/mfpf-dbinit
      tag: 2.0.2
      resources:
        requests:
          cpu: 300m
          memory: 300Mi
        limits:
          cpu: 400m
          memory: 400Mi
  ###############################################################################
  ## MFP Server configuration
  ###############################################################################
  mfpserver:
    enabled: true
    repository: docker-registry.default.svc.cluster.local:5000/<my_namespace>/mfpf-server
    tag: 2.0.2
    consoleSecret: ""
    db:
      type: "DB2"
      host: "<db2_url>"
      port: "50000"
      name: "BLUDB"
      secret: "mobilefoundation-db-secret"
      schema: "MFPOC"
      ssl: false
      driverPvc: ""
      adminCredentialsSecret: ""
    adminClientSecret: ""
    pushClientSecret: ""
    internalConsoleSecretDetails:
      consoleUser: "admin"
      consolePassword: "admin"
    internalClientSecretDetails:
      adminClientSecretId: mfpadmin
      adminClientSecretPassword: nimdapfm
      pushClientSecretId: push
      pushClientSecretPassword: hsup
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 1000m
        memory: 1536Mi
      limits:
        cpu: 2000m
        memory: 2048Mi
  ###############################################################################
  ## MFP Push configuration
  ###############################################################################
  mfppush:
    enabled: true
    repository: docker-registry.default.svc.cluster.local:5000/<my_namespace>/mfpf-push
    tag: 2.0.2
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 750m
        memory: 1024Mi
      limits:
        cpu: 1000m
        memory: 2048Mi
  ###############################################################################
  ## MFP Analytics configuration
  ###############################################################################
  mfpanalytics:
    enabled: false
    repository: docker-registry.default.svc.cluster.local:5000/<my_namespace>/mfpf-analytics
    tag: 2.0.2
    consoleSecret: ""
    internalConsoleSecretDetails:
      consoleUser: "admin"
      consolePassword: "admin"
    replicas: 2
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    shards: "3"
    replicasPerShard: "1"
    persistence:
      enabled: true
      useDynamicProvisioning: false
      volumeName: "data-stor"
      claimName: ""
      storageClassName: ""
      size: 20Gi
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 750m
        memory: 1024Mi
      limits:
        cpu: 1000m
        memory: 2048Mi
  ###############################################################################
  ## MFP Application center configuration
  ###############################################################################
  mfpappcenter:
    enabled: false
    repository: docker-registry.default.svc.cluster.local:5000/<my_namespace>/mfpf-appcenter
    tag: 2.0.2
    consoleSecret: ""
    internalConsoleSecretDetails:
      consoleUser: "admin"
      consolePassword: "admin"
    db:
      db:
      type: "DB2"
      host: "dashdb-txn-flex-yp-dal13-82.services.dal.bluemix.net"
      port: "50000"
      name: "BLUDB"
      secret: "mobilefoundation-db-secret"
      schema: "APPCNTROC"
      ssl: false
      driverPvc: ""
      adminCredentialsSecret: ""
    replicas: 1
    autoscaling:
      enabled: false
      min: 1
      max: 10
      targetcpu: 50
    pdb:
      enabled: true
      min: 1
    customConfiguration: ""
    keystoreSecret: ""
    resources:
      requests:
        cpu: 750m
        memory: 1024Mi
      limits:
        cpu: 1000m
        memory: 2048Mi
```




Finally check that your routes have been generated.

```
oc get routes

NAME                                      HOST/PORT               PATH        SERVICES             PORT      TERMINATION   WILDCARD
 ibm-mf-cr-1fdub-mfp-ingress-57khp   myhost.mydomain.cloud   /imfpush          ibm-mf-cr--mfppush     9080                    None
 ibm-mf-cr-1fdub-mfp-ingress-8skfk   myhost.mydomain.cloud   /mfpconsole       ibm-mf-cr--mfpserver   9080                    None
 ibm-mf-cr-1fdub-mfp-ingress-dqjr7   myhost.mydomain.cloud   /doc              ibm-mf-cr--mfpserver   9080                    None
 ibm-mf-cr-1fdub-mfp-ingress-ncqdg   myhost.mydomain.cloud   /mfpadminconfig   ibm-mf-cr--mfpserver   9080                    None
 ibm-mf-cr-1fdub-mfp-ingress-x8t2p   myhost.mydomain.cloud   /mfpadmin         ibm-mf-cr--mfpserver   9080                    None
 ibm-mf-cr-1fdub-mfp-ingress-xt66r   myhost.mydomain.cloud   /mfp              ibm-mf-cr--mfpserver   9080                    None
```
* Please note, your routes may vary depending on if you choose to install the analytics.

The installation of Mobile Foundation is now complete. For more information on how you can use Mobile Foundation to support your application development, [click here](#).

## Step 5 - Uninstall Mobile Foundation.

If you wish to uninstall Mobile Foundation you can do so simply by running the following steps:

```
oc delete -f deploy/crds/charts_v1_mfoperator_cr.yaml
oc delete -f deploy/
oc delete -f deploy/crds/charts_v1_mfoperator_crd.yaml
oc patch crd/ibmmf.charts.helm.k8s.io -p '{"metadata":{"finalizers":[]}}' --type=merge
```

You should also finish the cleanup by removing your database instance, the process to do this will depend on the database you choose at the begining of the guide.

Please note, this guide is an adaptation of the guide found [here](#). We have adapted the steps and added guidence where appropriate to match our experience in installing Mobile Foundation on OpenShift 3.11.