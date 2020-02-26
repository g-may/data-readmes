![](https://raw.githubusercontent.com/ibm-cloud-docs/images/master/discovery-icp4d-new-banner.png)

# IBM Watson Discovery

## Introduction
IBM Watson™ Discovery for IBM® Cloud Pak is an AI-powered search and content analytics engine that finds answers and insights from complex business content with speed and accuracy. Answers can be surfaced to users through a conversational dialog driven by Watson Assistant or embedded in your own user-interface. With its Smart Document Understanding training interface, Watson Discovery can learn where answers live in complex business content based on a visual understanding of documents. Further enhance Watson Discovery's ability to understand domain specific language by teaching it with Watson Knowledge Studio.

Watson Discovery brings together a functionally rich set of integrated and automated Watson APIs to:
  - Convert, enrich, and normalize data.
  - Securely explore your proprietary content as well as free and licensed public content.
  - Apply additional enrichments such as keywords through Natural Language Understanding (NLU).
  - Simplify development while still providing direct access to APIs.

## Chart Details

This chart deploys a single Watson Discovery node with a default pattern. It includes the endpoints listed [here](https://cloud.ibm.com/apidocs/discovery/discovery-data-v2).


## Prerequisites

Before installing Watson Discovery, you must install:

  - IBM® Cloud Pak for Data 2.5.0.0 or 2.1.0.2
    - [IBM® Cloud Pak for Data 2.5.0.0 installation instructions](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.5.0/cpd/install/install.html). (Also see [IBM Cloud Pak for Data 2.5.0.0 release notes](https://www.ibm.com/support/knowledgecenter/SSQNUZ_current/cpd/overview/relnotes-2.5.0.0.html)).
    - [IBM® Cloud Pak for Data 2.1.0.2 installation instructions](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.1.0/com.ibm.icpdata.doc/zen/install/ovu.html)

**IMPORTANT**: Portworx **must** be installed before you install Watson Discovery. A Portworx license is included with IBM® Cloud Pak for Data 2.5.0.0.

## Limitations

- Watson Discovery must be deployed within the same namespace as IBM Cloud Pak for Data (by default the `zen` namespace).
- A Watson Discovery deployment supports a single service instance.
- Watson Discovery can currently run only on Intel 64-bit architecture.
- Release names cannot exceed 13 characters
- This chart should be used with the default image tags provided. Other image versions might be incompatible.
- This chart must be installed through the CLI.
- This chart must be installed by a ClusterAdministrator
- This chart currently does not support upgrades or rollbacks. See [Backing up and restoring data](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-backup-restore) for instructions.


## Resources Required

In addition to the [System requirements for IBM Cloud Pak for Data](https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_current/cpd/plan/rhos-reqs.html), IBM Watson Discovery has the following requirements.

For installation (at minimum):

|                                                           |Disk        | Docker registry |
|-----------------------------------------------------------|:----------:|:---------------:|
| Discovery                                                 | 50 GB      | 45 GB           |
| Additional needed for Discovery for Content Intelligence  | 40 GB      | 20 GB           |


For use:

|                      | Minimum VPC available | Minimum RAM available |
|----------------------|:---------------------:|:---------------------:|
| Development (non-HA) | 21                    | 86 GB                 |
| Production (HA)      | 26                    | 116 GB                |


## Storage

Parenthetical numbers are the PVs required/created when deploying with the recommended high availability (HA) configuration. See [High availability (Production) configuration](#high-availability-configuration) for more information.

| Component      | Number of replicas | Space per PVC | Storage type            |
|----------------|--------------------|---------------|-------------------------|
| Postgres       |               2(3) |         10 Gi | portworx-sc |
| Etcd           |                  3 |         10 Gi | portworx-sc |
| Elastic        |               1(2) |         10 Gi | portworx-sc |
| Elastic Backup |                  1 |         30 Gi | portworx-sc |
| Minio          |                  4 |         25 Gi | portworx-sc |
| MongoDB        |                  3 |         20 Gi | portworx-sc |
| Hdp worker     |                  2 |        100 Gi | portworx-sc |
| Hdp nn         |                  1 |         10 Gi | portworx-sc |
| Core common    |                  1 |        100 Gi | portworx-sc |
| Core ingestion |                  1 |          1 Gi | portworx-sc |

**Note:** Gluster File System (GlusterFS) is not a supported storage option for Watson Discovery.

## Documentation

IBM Watson Discovery documentation:

- [Watson Discovery for IBM Cloud Pak for Data](https://cloud.ibm.com/docs/services/discovery-data)
- [API reference](https://cloud.ibm.com/apidocs/discovery/discovery-data-v2)

IBM® Cloud Pak for Data:
- [IBM Cloud Pak for Data V2.5.0 Knowledge Center](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.5.0/cpd/overview/welcome.html)
- [IBM Cloud Pak for Data V2.1.0 Knowledge Center](https://www.ibm.com/support/knowledgecenter/SSQNUZ_2.1.0/com.ibm.icpdata.doc/zen/overview/welcome.html)


## Pre-install steps

### Setting up an OpenShift Environment

**NOTE:** Skip this section if you are deploying to IBM Cloud Private. See [Setting up an IBM Cloud Private environment](#setting-up-an-ibm-cloud-private-environment).

If you're deploying to an OpenShift cluster, `helm` will be installed and configured as part of running `deploy.sh`.

1. Log into the Kubernetes cluster and target the namespace or project the chart will be deploy to. Use the `oc` cli.

    ```bash
    oc login https://{cluster_hostname}:8443 -u {user} -p {password}
    ```

2. Log into the cluster's docker registry.

    ```bash
    docker login $(oc get routes docker-registry -n default -o template={{.spec.host}}) -u {user} -p {password}
    ```

3. Download Discovery from Passport Advantage (if you haven't already) and create a directory to extract its content to:

    ```
    mkdir ibm-watson-discovery-ppa
    tar -xJf ibm-watson-discovery-2.1.1.tar.xz -C ibm-watson-discovery-ppa
    cd ibm-watson-discovery-ppa
    ```

   If you have purchased `Discovery for Content Intelligence` along with `Watson Discovery`, also download and extract the `Discovery for Content Intelligence PPA`:

   ```
   mkdir ibm-discovery-content-intelligence-ppa
   tar -xJf <ci-service-ppa-archive>  -C ibm-discovery-content-intelligence-ppa
   ```
4. If you're deploying to an OpenShift cluster, there is a good chance that you'll need to increase the system's virtual memory max map count. ElasticSearch will not start unless `vm.max_map_count` is greater than or equal to `262144`. This should be set automatically as part of the cluster provisioning script for IBM Cloud Private clusters, but will need to be done for an OpenShift cluster.

To make these changes, add `vm.max_map_count=262144` to `/etc/sysctl.conf` on each node.


### Setting up an IBM Cloud Private environment

**NOTE:** Skip this section if you are deploying to OpenShift. See [Setting up an OpenShift Environment](#setting-up-an-openshift-environment).

1. Initialize Helm client by running the following command. For further details of Helm CLI setup, see [Installing the Helm CLI](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/app_center/create_helm_cli.html).

    ```bash
    export HELM_HOME={helm_home_dir}
    helm init --client-only
    ```

     - `{helm_home_dir}` is your Helm config directory. For example, `~/.helm`.

2. Log into the Kubernetes cluster and target the namespace or project the chart will be deploy to. Use the `cloudctl` cli.

    ```bash
    cloudctl login -a https://{cluster_hostname}:8443 -u {user} -p {password}
    ```


3. Log into the cluster's docker registry.

    ```bash
    docker login {cluster_hostname}:8500 -u {user} -p {password}
    ```

4. Download Discovery from Passport Advantage (if you haven't already) and create a directory to extract its content to:

    ```
    mkdir ibm-watson-discovery-ppa
    tar -xJf ibm-watson-discovery-2.1.1.tar.xz -C ibm-watson-discovery-ppa
    cd ibm-watson-discovery-ppa
    ```

   If you have purchased `Discovery for Content Intelligence` along with `Watson Discovery`, also download and extract the `Discovery for Content Intelligence PPA`:

   ```
   mkdir ibm-discovery-content-intelligence-ppa
   tar -xJf <ci-service-ppa-archive>  -C ibm-discovery-content-intelligence-ppa
   ```

## Installing the Chart

Installing the Helm chart deploys a single Watson Discovery application into an IBM Cloud Pak environment. You can deploy to  a `development` or `production` environment. By default, the chart will install in `Development` mode. See [High availability (Production) configuration](#high-availability-configuration) for instructions on deploying to `production`.


To deploy Watson Discovery to your cluster, run the `deploy.sh` script from the `deploy` subdirectory.
Run `./deploy.sh  -h` for help.

Include the flag:

```
 ./deploy.sh -d path/to/ibm-watson-discovery
```

Your deploy command `-d` should point to the `ibm-watson-discovery` directory from the release artifact.

**NOTE: If you are deploying to an IBM Cloud Private Foundations cluster you must add the `--openshift false` flag to your `./deploy.sh` command**

By default the `deploy.sh` script will look for tiller in the namespace that IBM Cloud Pak for Data is running in on an OpenShift cluster, or `kube-system` on an IBM Cloud Private Foundations cluster. If your tiller pod is located in a different namespace, you can override it with the `-w` flag.

If you would like to change the value of a parameter from its default value, you must create a yaml file in `ibm-watson-discovery-ppa/deploy/`, overriding any values you want and then pass it in your command with `-O {override-file}.yaml`.

The name of the your helm release will be `{release-name-prefix}`. The total length of `{release-name-prefix}` must be [13 characters or less or the installation will fail](#important-restrictions-and-limitations). If you do not pass in `-e flag`, `{release-name-prefix}` the name will be `disco`.

**If you've purchased and downloaded Discovery for Content Intelligence, you must install it now**:
  1. Use the values provided in `ci-override.yaml` file under `ibm-discovery-content-intelligence-ppa` folder, i.e., run the deploy script with `-O ci-override.yaml`.  If you already have an `{override-file.yaml}`, first append these values of `ci-override.yaml` into your `<override-file.yaml>` and then run the deploy script with your `-O {override-file}.yaml`.
  2. After Discovery for Content Intelligence is installed, add the `Software Identification Tags` to the required pod: 
  
      ```
      oc cp ibm-discovery-content-intelligence-ppa/ibm.com_IBM_Watson_Discovery_for_Cloud_Pak_for_Data_Content_Intelligence-2.1.0.swidtag "$(oc get pod | awk '{print $1}' | grep gateway-0)":/swidtag/
      ```
  
**Note**: The `Discovery for Content Intelligence PPA` includes the tag file ending with `.swidtag`.

**Tip**: Run `-h` to view the list of available flags and help.

### High availability configuration

To deploy in High Availability (Production) mode,

1. Create an `override.yaml` in `ibm-watson-discovery-ppa/deploy/`

2. Add `global.deploymentType=Production` to your `override.yaml` file and specify the file with `-O override.yaml` in your deploy.sh command.

```
cat > override.yaml << 'EOF'
global:
  deploymentType: "Production"
EOF
```

### Verifying the Chart installation

Run the command:

```bash
$ helm test my-release --tls --tiller-namespace=<tiller-namespace>
```

The installation is complete.


### Installing the optional language pack

The following enrichments are supported in English only, unless you download and install the language extension pack `ibm-watson-discovery-pack1` from Passport Advantage. For a breakdown of supported languages, see [Language support](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-language-support).

- Entities
- Keywords
- Sentiment of documents

Prerequisite:
- Watson Discovery must be installed and running

To install `ibm-watson-discovery-pack1`:

1. Unpack the language pack.

```bash
$ mkdir ibm-watson-discovery-language-pack
$ cd ibm-watson-discovery-language-pack
$ mv ~/Downloads/ibm-watson-discovery-pack1-2.1.0.tar.xz .
$ tar xJf ibm-watson-discovery-pack1-2.1.0.tar.xz
$ ls .
deploy  ibm-watson-discovery-pack1
```
2. Verify you are authenticated with Docker and your cluster (`oc` on OpenShift, or `cloudctl` for IBM Cloud Private). (See [Setting up an OpenShift Environment](#setting-up-an-openshift-environment) or [Setting up an IBM Cloud Private environment](#setting-up-an-ibm-cloud-private-environment)).

3. Run `deploy.sh`. The `-d` flag is required and must point at the `ibm-watson-discovery-pack1` directory from the unpacked archive. The `-e` flag is also required and you must provide the same release-name that you used for your Discovery install.

```bash
$ ./deploy/deploy.sh -d ./ibm-watson-discovery-pack1 -e $discovery_install_name
```

**NOTE: If you are deploying to an IBM Cloud Private Foundations cluster you must add the `--openshift false` flag to your `./deploy.sh` command**

### Collecting OpenShift Support Logs

From `ibm_cloud_pak/pak_extensions/common/`, run  `./openshiftCollector.sh -c OPENSHIFT_CLUSTER_HOST -n ICP4D_NAMESPACE -r HELM_RELEASE_NAME -u OPENSHIFT_ADMIN_USERNAME -p OPENSHIFT_ADMIN_PASSWORD`, where
- `OPENSHIFT_CLUSTER_HOST` is the cluster without the protocol/port
- `ICP4D_NAMESPACE` is the namespace where ICP4D is installed (usually `zen`).
- `OPENSHIFT_ADMIN_USERNAME` is the username for the openshift administrator
- `OPENSHIFT_ADMIN_PASSWORD` is the password for the openshift administrator


For more information and options, run `./ibm_cloud_pak/pak_extensions/common/openshiftCollector.sh --help`

This will produce a `.tgz` file in your current directory with the format `${OPENSHIFT_CLUSTER_HOST}_${DATE_TIME}.tgz`. This file will be what the support team can use to debug issues on the cluster.

You can follow the process of the log collection by running a command like `tail -f ./${OPENSHIFT_CLUSTER_HOST}_${DATE_TIME}/watson-diagnostics-data.log` in the directory that you started the command.

### Uninstalling Watson Discovery

To delete these resources from the `my-release` deployment that had been deployed in `my-namespace`, run this command:

```bash
kubectl delete --namespace=my-namespace all,configmaps,jobs,networkpolicies,poddisruptionbudgets,roles,rolebindings,clusterroles,clusterrolebindings,secrets,serviceaccounts --selector=release=my-release
kubectl delete --namespace=my-namespace configmaps stolon-cluster-my-release-postgresql my-release.v1
```

If you are sure you want to delete any persistent storage, first delete the Persistent Volume Claims:
```bash
kubectl delete --namespace=my-namespace persistentvolumeclaims --selector=release=my-release
```

## Security reference

**NOTE**: This information is provided for reference. The install will create the security requirements for you.

### PodSecurityPolicy Requirements

**NOTE**: This information is provided for reference. The install will create the security requirements for you.

This chart requires a PodSecurityPolicy to be bound to the target namespace prior to installation. The predefined PodSecurityPolicy name: [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp) has been verified for this chart.

These PodSecurityPolicy resources can also be created manually. A cluster admin can save these templates to separate yaml files and run the command below for each file:

```
kubectl create -f <some-resource->.yaml
```

Template for a PodSecurityPolicy definition, currently equivalent `ibm-restricted-psp`, to bind to and create your own RBAC resources.

Custom PodSecurityPolicy definition:
  ```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    kubernetes.io/description: This policy is the most restrictive, requiring pods
      to run with a non-root UID, and preventing pods from accessing the host.
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
    seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
  name: ibm-discovery-custom-psp
spec:
  allowPrivilegeEscalation: false
  forbiddenSysctls:
  - '*'
  fsGroup:
    ranges:
    - max: 65535
      min: 1
    rule: MustRunAs
  requiredDropCapabilities:
  - ALL
  runAsUser:
    rule: MustRunAsNonRoot
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    ranges:
    - max: 65535
      min: 1
    rule: MustRunAs
  volumes:
  - configMap
  - emptyDir
  - projected
  - secret
  - downwardAPI
  - persistentVolumeClaim
```

Template for custom Role resource to replace the default privileged role:
  ```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name:  discovery-custom-priv-role
rules:
- apiGroups: ["", "batch", "extensions"]
  resources: ["jobs", "jobs/status", "secrets", "pods", "pods/exec", "configmaps"]
  verbs: ["get", "watch", "create", "apply", "list", "update", "patch", "delete"]
- apiGroups: ["policy"]
  resources: ["podsecuritypolicies"]
  resourceNames: [ibm-discovery-custom-psp]
  verbs: ["use"]
- apiGroups: [""]
  resources: ["resourcequotas", "resourcequotas/status"]
  verbs: ["get", "list", "watch"]
```

To use a custom Role resource you must have custom ServiceAccount resource.

**NOTE:** You **must** replace <pull-secret-name> with the name of the image pull secret in the namespace of your cluster. For IBM Cloud Private Foundations that secret name is sa-<namespace-name>. For RedHat OpenShift you can find the secret name by running
```bash
oc get secrets \
  --namespace "${NAMESPACE}" \
  --output=jsonpath='{ range .items[*] }{@.metadata.name}{"\n"}{end}' \
| grep default-dockercfg \
| tr -d '[:space:]'
```

Template for custom ServiceAccount resource to replace the default privileged service account:
  ```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: discovery-custom-priv-service-account
imagePullSecrets:
- name: <pull-secret-name>
```

To bind your service accounts to your role you need to create a rolebinding.

Template for role binding your custom privileged service account to your custom privileged role:
  ```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name:  discovery-custom-priv-role-binding
subjects:
- kind: ServiceAccount
  name: discovery-custom-priv-service-account
roleRef:
  kind: Role
  name:  discovery-custom-priv-role
  apiGroup: rbac.authorization.k8s.io
```

Finally, you can specify the name of the custom service account when installing the chart. For example,if you want to override the privileged service account with `discovery-custom-priv-service-account`, add the following to your helm install command:

```
<...> --set global.privilegedServiceAccount.name=discovery-custom-priv-service-account --set global.privilegedServiceAccount.create=false --set global.privilegedRbac.create=false
```

### Red Hat OpenShift SecurityContextConstraints Requirements

**NOTE**: This information is provided for reference. The install will create the security requirements for you.

If running in a Red Hat OpenShift cluster, this chart requires a `SecurityContextConstraints` to be bound to the target namespace prior to installation. The predefined PodSecurityPolicy name: [`restricted`](https://ibm.biz/cpkspec-scc) has been verified for this chart.

The SecurityContextConstraint resource can also be created manually. A cluster admin can save this template to a yaml file and run the command below:

```
kubectl create -f ibm-discovery-prod-scc.yaml
```

Template for a SecurityContextConstraints definition, currently equivalent `restricted`, to bind to your namespace.

  * Custom `SecurityContextConstraints` definition:

  ```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: "This policy is the most restrictive,
      requiring pods to run with a non-root UID, and preventing pods from accessing the host."
    cloudpak.ibm.com/version: "1.0.0"
  name: ibm-discovery-prod-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
allowedCapabilities: []
allowedFlexVolumes: []
allowedUnsafeSysctls: []
defaultAddCapabilities: []
defaultPrivilegeEscalation: false
forbiddenSysctls:
  - "*"
fsGroup:
  type: MustRunAs
  ranges:
  - max: 65535
    min: 1
readOnlyRootFilesystem: false
requiredDropCapabilities:
- ALL
runAsUser:
  type: MustRunAsNonRoot
seccompProfiles:
- docker/default
seLinuxContext:
  type: RunAsAny
supplementalGroups:
  type: MustRunAs
  ranges:
  - max: 65535
    min: 1
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
priority: 0
```

After creating the scc, you can bind the SCC to the namespace with this command, replacing `$namespace` with the namespace you're deploying to:

```
oc adm policy add-scc-to-group ibm-discovery-prod-scc system:serviceaccounts:$namespace
```

## Configuration

Contact Support.

## Backup and Restore

This chart currently does not support upgrades or rollbacks. See [Backing up and restoring data](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-backup-restore) for instructions.

## Integration with other IBM Watson services

Watson Discovery is one of many IBM Watson services. Additional Watson services on IBM Cloud Pak for Data and the IBM Public Cloud allow you to bring Watson's AI platform to your business application, and to store, train, and manage your data in the most secure cloud.

For the full list of available Watson services, see:

- [IBM Cloud Pak for Data](https://www.ibm.com/products/cloud-pak-for-data)
- [IBM Watson Public Cloud catalog](https://cloud.ibm.com/catalog?category=ai)

Watson services are currently organized into the following categories for different requirements and use cases:

  - **Assistant**: Integrate diverse conversation technology into your application
  - **Empathy**: Understand tone, personality, and emotional state
  - **Knowledge**: Get insights through accelerated data optimization capabilities
  - **Language**: Analyze text and extract metadata from unstructured content
  - **Speech**: Convert text and speech with the ability to customize models
  - **Vision**: Identify and tag content, then analyze and extract detailed information found in images

_Copyright©  IBM Corporation 2020. All Rights Reserved._
