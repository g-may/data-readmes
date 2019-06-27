![](https://raw.githubusercontent.com/IBM-Bluemix-Docs/images/master/discovery-icp4d-banner.png)

# IBM Watson Discovery

## Introduction
IBM Watson™ Discovery for IBM® Cloud Private is an AI-powered search and content analytics engine that finds answers and insights from complex business content with speed and accuracy. Answers can be surfaced to users through a conversational dialog driven by Watson Assistant or embedded in your own user-interface. With its Smart Document Understanding training interface, Watson Discovery can learn where answers live in complex business content based on a visual understanding of documents. Further enhance Watson Discovery's ability to understand domain specific language by teaching it with Watson Knowledge Studio.

Watson Discovery brings together a functionally rich set of integrated and automated Watson APIs to:
  - Convert, enrich, and normalize data.
  - Securely explore your proprietary content as well as free and licensed public content.
  - Apply additional enrichments such as keywords through Natural Language Understanding (NLU).
  - Simplify development while still providing direct access to APIs.

## Chart Details

This chart deploys a single Watson Discovery node with a default pattern.

It includes the following endpoints:

 - Document endpoints accessible on `/api/v1/environments/{environment_id}/collections/{collection_id}/documents`.
 - Training data endpoints accessible on `/api/v1/environments/{environment_id}/collections/{collection_id}/training_data`.
 - A query endpoint accessible on `/api/v1/environments/{environment_id}/collections/{collection_id}/query`.
 - An autocomplete endpoint accessible on `api/v1/environments/{environment_id}/collections/{collection_id}/autocompletion`.

## Prerequisites

  - IBM® Cloud Private for Data 2.1

Before installing Watson Discovery, __you must install and configure [ICP4D](https://www.ibm.com/support/knowledgecenter/SSQNUZ_current/com.ibm.icpdata.doc/zen/overview/relnotes-2.1.0.0.html)__.

## PodSecurityPolicy Requirements

The predefined PodSecurityPolicy name: [`ibm-anyuid-psp`](https://ibm.biz/cpkspec-psp) has been verified for this chart. By default, the chart will automatically install the necessary RBAC roles and rolebindings for running the service. It will provide two service accounts: one for privileged access to kubernetes resources and one without. Both service accounts will be bound to `ibm-anyuid-psp`. If, however, `global.serviceAccount.create` and `global.rbac.create` or `global.privilegedServiceAccount.create` and `global.privilegedRbac.create` is set to `false`, then the chart expects `global.serviceAccount.name` or `global.privilegedServiceAccount.name` to be set to the name of a valid service account that is rolebound to an appropriate PSP, `ibm-anyuid-psp` or have the namespace itself be bound to the PSP. Furthermore, in either of the two alternatives, the `privilegedServiceAccount` should also be bound to a role with elevated privileges that have access to kubernetes resources.

We provide multiple templates that can be used to create different Kubernetes resources. A cluster admin can save these templates to separate yaml files and run the command below for each file:

```
kubectl create -f <some-resource->.yaml
```

Template for a PodSecurityPolicy definition, currently equivalent `ibm-anyuid-psp`, to bind to and create your own RBAC resources.

Custom PodSecurityPolicy definition:
```
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    kubernetes.io/description: "This policy allows pods to run with
      any UID and GID, but preventing access to the host."
  name: ibm-discovery-custom-psp
spec:
  allowPrivilegeEscalation: true
  fsGroup:
    rule: RunAsAny
  requiredDropCapabilities:
  - MKNOD
  allowedCapabilities:
  - SETPCAP
  - AUDIT_WRITE
  - CHOWN
  - NET_RAW
  - DAC_OVERRIDE
  - FOWNER
  - FSETID
  - KILL
  - SETUID
  - SETGID
  - NET_BIND_SERVICE
  - SYS_CHROOT
  - SETFCAP
  runAsUser:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - configMap
  - emptyDir
  - projected
  - secret
  - downwardAPI
  - persistentVolumeClaim
  forbiddenSysctls:
  - '*'
```

You can then pass this custom PSP through the CLI when installing the chart, and the chart will create RBAC resources based on this PSP `<...> --set global.podSecurityPolicy=ibm-discovery-custom-psp`.

We also give users even further control by giving them the option to create their own RBAC resource.

Template for custom Role resource to replace the default privileged role:
```
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

Template for custom Role resource to replace the default standard role:
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name:  discovery-custom-role
rules:
- apiGroups: ["policy"]
  resources: ["podsecuritypolicies"]
  resourceNames: [ibm-discovery-custom-psp]
  verbs: ["use"]
```

To use a custom Role resource you must have custom ServiceAccount resource.

**NOTE:** You **must** replace <namespace-name> with the actual name of the namespace you're planning to deploy to because these K8 resources are being created outside of a Helm release.

Template for custom ServiceAccount resource to replace the default privileged service account:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: discovery-custom-priv-service-account
imagePullSecrets:
- name: sa-<namespace-name>
```

Template for custom ServiceAccount resource to replace the default standard service account:
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: discovery-custom-service-account
imagePullSecrets:
- name: sa-<namespace-name>
```

To bind your service accounts to your role you need to create a rolebinding.

Template for role binding your custom privileged service account to your custom privileged role:
```
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

Template for role binding your custom standard service account to your custom standard role:
```
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name:  discovery-custom-role-binding
subjects:
- kind: ServiceAccount
  name: discovery-custom-service-account
roleRef:
  kind: Role
  name:  discovery-custom-role
  apiGroup: rbac.authorization.k8s.io
```

Finally, you can specify the name of the custom service account when installing the chart. For example, if you want to override the privileged service account with `discovery-custom-priv-service-account`, add the following to your helm install command:

```
<...> --set global.privilegedServiceAccount.name=discovery-custom-priv-service-account --set global.privilegedServiceAccount.create=false --set global.privilegedRbac.create=false --tls
```

To override the normal service account with `discovery-custom-service-account` add the following to your helm install command:

```
<...> --set global.serviceAccount.name=discovery-custom-service-account --set global.serviceAccount.create=false --set global.rbac.create=false --tls
```

## Resources Required

In addition to the [general hardware requirements and recommendations](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/supported_system_config/hardware_reqs.html), the IBM Watson Discovery has the following requirements:

  - Minimum CPU - 16 for dev, 24 for HA
  - Minimum RAM - 50GB for dev, 83GB for HA
  - Cluster configuration:
     - Dev: 1 load balancing node, 3 master nodes, 3 worker nodes
     - HA: 1 load balancing node, 3 master nodes, 4 worker nodes

See [HA-configuration](#ha-configuration) for more information on configuration needed for HA.

## Storage

Parenthetical numbers are the PVs required/created when deploying with the recommended HA configuration. See [HA-configuration](#ha-configuration) for more information.

| Component      | Number of replicas | Space per PVC | Storage type            |
|----------------|--------------------|---------------|-------------------------|
| Postgres       |                  1 |         10 GB | oketi-nfs |
| Etcd           |               1(3) |         10 GB | oketi-nfs |
| Elastic        |               1(2) |         10 GB | oketi-nfs |
| Minio          |               1(4) |          5 GB | oketi-nfs |
| Hdp worker     |                  2 |        100 GB | oketi-nfs |
| Hdp nn         |                  1 |         10 GB | oketi-nfs |
| Core common    |                  1 |        100 GB | oketi-nfs |
| Core ingestion |                  1 |          1 GB | oketi-nfs |

**Note:** By default the product assumes a storage provider (e.g. `oketi-nfs`) which supports disks with access mode `ReadWriteMany`. If this is not the case, `minio.persistence.accessMode` will need to be set to `ReadWriteOnce`.


## Pre-install steps


### Setting up Environment

1. Initialize Helm client by running the following command. For further details of Helm CLI setup, see [Installing the Helm CLI (helm)](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/app_center/create_helm_cli.html)

    ```bash
    export HELM_HOME={helm_home_dir}
    helm init --client-only
    ```

     - `{helm_home_dir}` is your Helm config directory. For example, `~/.helm`.

2. Log into the ICP cluster and target the namespace the chart will be deployed using `cloudctl`. See the [add-on installation](https://docs-icpdata.mybluemix.net/docs/content/SSQNUZ_current/com.ibm.icpdata.doc/watson/discovery-install.html) and [bundle installation](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-install#install) for more detail.

    ```bash
    cloudctl login -a https://{cluster_CA_domain}:8443 -u {user} -p {password}
    ```
3. Create the Kubernetes namespace that you wish to deploy to.

4. Get a certificate from your cluster and install it to Docker or add the {cluster_CA_domain} as a Docker Daemon insecure registry. You must do one or the other for Docker to be able to pull from your cluster.

    See [Specifying your own certificate authority (CA)](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.0/installing/create_ca_cert.html).

5. Log into the docker registry.

    ```bash
    docker login {cluster_CA_domain}:8500
    ```

6. To load the file from Passport Advantage into IBM Cloud Private, enter the following command in the IBM Cloud Private command line interface.

    ```bash
    cloudctl target -n {namespace_name}
    cloudctl catalog load-archive --archive {compressed_file_name}
    ```

    - `{compressed_file_name}` is the name of the file that you downloaded from Passport Advantage.
    - `{namespace_name}` is the Docker namespace that hosts the Docker image. This is the namespace you created in Step 1.
7. Once loading PPA is finished, the Helm chart of the IBM Watson Discovery is added to the local Helm repository of the cluster. To install the chart, add cluster local helm repository `local-charts` to your Helm CLI. See [Adding the internal Helm repository to Helm CLI](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.2/app_center/add_int_helm_repo_to_cli.html) for more detail.

    ```bash
    sudo helm repo add local-charts https://{cluster_CA_domain}:8443/helm-repo/charts --ca-file $HELM_HOME/ca.pem --cert-file $HELM_HOME/cert.pem --key-file $HELM_HOME/key.pem
    ```

8. Run `kubectl edit ClusterImagePolicy ibmcloud-default-cluster-image-policy`, and add an entry under spec.repositories `- name: '{cluster_CA_domain}:8500/{namespace}/*'`, replacing `{cluster_CA_domain}` and `{namespace}` with the appropriate values. For further details of ImagePolicy, see [Enforcing container image security](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.2/manage_images/image_security.html).

9. This script has to be run once per cluster by a cluster admin. It runs a kubectl command to add a label to the given namespace.

    From `ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration/`, run  `./labelNamespace.sh ICP4D_NAMESPACE`, where `ICP4D_NAMESPACE` is the namespace where ICP4D is installed (usually `zen`).

    The namespace `ICP4D_NAMESPACE` **must** have a label for the `NetworkPolicy` to allow nginx and zen pods to communicate with the pods in the namespace where this chart is installed.

10. Your ICP4D cluster may have been deployed without the `oketi-nfs` storage class: `kubectl get StorageClass`. If this is the case, and you would like to use `oketi-nfs` as the storage class for your PVs, you must create the storage class first.

    ```
    cat > oketi-nfs.yaml << EOF
      apiVersion: storage.k8s.io/v1
      kind: StorageClass
      metadata:
        name: oketi-nfs
      parameters:
        nodes: |
          [
            { "Hostname":"HOST", "IP":"$NFS_SERVER", "Path":"$NFS_DIR" }
          ]
        volumeType: nfs
      provisioner: oketi
      reclaimPolicy: Retain
      volumeBindingMode: Immediate
    EOF
    ```
    `$NFS_SERVER` should be the address of the NFS server and `$NFS_DIR` should be the root of the nfs directory.
    After modifying and saving the file appropriately, run the command:
    `kubectl apply -f oketi-nfs.yaml`.


## Installing the Chart

Installing the Helm chart deploys a single Watson Discovery node with a default pattern into an IBM Cloud Private environment.

To change the value of a parameter from its default value, add `--set <parameter-name>=<parameter-value>` to the helm install command.
See [HA-configuration](#ha-configuration) for details on the configuration needed for an HA deployment.
See [Configuration](#configuration) for details on these configurations needed to install the chart. 

To install Watson Discovery you must provide the IP and Hostname of an IBM Cloud Private for Data master node. The IP address must be addressable by pods running inside the cluster. To install Watson Discovery as `my-release` using the `oketi-nfs` dynamic provisioner configured above run this command:

```bash
helm install --name my-release local-charts/ibm-watson-discovery-prod --tls \
  --set global.license=accept \
  --set core.common.persistence.storageClassName=oketi-nfs \
  --set core.ingestion.mount.storageClassName=oketi-nfs \
  --set elastic.backup.persistence.storageClassName=oketi-nfs \
  --set elastic.persistence.storageClassName=oketi-nfs \
  --set etcd.persistence.storageClassName=oketi-nfs \
  --set hdp.persistence.storageClassName=oketi-nfs \
  --set minio.persistence.storageClass=oketi-nfs \
  --set postgresql.persistence.storageClassName=oketi-nfs \
  --set global.icp.masterHostname="${master_hostname}" \
  --set global.icp.masterIP="${master_ip}"
```

This command accepts the license, and, after the command runs, it prints the current status of the release.

### Verifying the Chart

See the instruction (from NOTES.txt within chart) after the helm installation completes for chart verification. The instruction can also be viewed by running the command:

```bash
$ helm status my-release --tls.
```


### Backup and Restore

This chart currently does not support upgrades or rollbacks. We recommend users to periodically backup their installation. For more information on the backup and restore procesures, please see the [product documentation](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-backup-restore). 

### Uninstalling the chart

To uninstall and delete the `my-release` deployment, run the following command:

```bash
$ helm delete --tls my-release
```

To irrevocably uninstall and delete the `my-release` deployment, run the following command:

```bash
$ helm delete --purge --tls my-release
```

If you omit the `--purge` option, Helm deletes all resources for the deployment, but retains the record with the release name. This allows you to roll back the deletion. If you include the `--purge` option, Helm removes all records for the deployment, so that the name can be used for another installation.

### Post Uninstall Cleanup

Certain Kubernetes resources outside of a helm release may not be deleted from a `helm delete --purge`.
These should be deleted manually.

To delete these resources from the `my-release` deployment that had been deployed in `my-namespace`, run this command:

```bash
kubectl delete --namespace=my-namespace configmaps,jobs,pods,statefulsets,deployments,roles,rolebindings,secrets,serviceaccounts --selector=release=my-release
```

If you are sure you want to delete any persistent storage, first delete the Persistent Volume Claims:
```bash
kubectl delete --namespace=my-namespace persistentvolumeclaims --selector=release=my-release
```

Then you can delete any corresponding persistent volumes associated with those deleted claims:
```bash
kubectl delete persistentvolumes $(kubectl get persistentvolumes \
  --output=jsonpath='{range .items[*]}{@.metadata.name}:{@.status.phase}:{@.spec.claimRef.name}{"\n"}{end}' \
  | grep ":Released:" \
  | grep "my-release-" \
  | cut -d ':' -f 1)
```

Also if you intend to re-use the release name on a future install, you must run the following command to purge the ICP4D Addon service instance database

```
kubectl -n zen exec zen-metastoredb-0 \
  -- sh /cockroach/cockroach.sh sql \
  --insecure -e "DELETE FROM zen.service_instances WHERE deleted_at IS NOT NULL RETURNING id;" \
  --host='zen-metastoredb-public'
```

## Configuration

The following tables lists the configurable parameter(s) of the Watson Discovery chart and their default values.
Before configuring a value that is not listed here, please double check what you're doing and/or consult IBM.

### Global parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `global.license` | Users must read the license and set the value to `accepted` | `not accepted` |
| `global.icpDockerRepo` | When installing via the CLI, this value must be set to `<cluster_CA_domain>:8500/<namespace>/` | `""` |
| `global.podSecurityPolicy` | The PSP of the RBAC resources the chart will create if toggled on | `ibm-anyuid-psp` |
| `global.icp.masterHostname` | Hostname of the ICP cluster Master node - i.e., the name where you Log into the cluster https://{{ masterHostname }}:8443 | `""` |
| `global.icp.masterIP` | IP (v4) address of the master node. Has to be specified if the global.icp.masterHostname cannot be resolved inside the pods (i.e., if the global.icp.masterHostname is not a valid DNS entry). This IP address has to be accessible from inside of the cluster. | `` |
| `global.rbac.create` | Set to true to create RBAC resources for the standard service account | `true` |
| `global.serviceAccount.create` | Set to true to create the standard service account. | `true` |
| `global.serviceAccount.name` | Set this value to provide your own standard service account. | `""` |
| `global.privilegedRbac.create` | Set to true to create RBAC resources for the privileged service account | `true` |
| `global.privilegedServiceAccount.create` | Set to true to create the privileged service account. | `true` |
| `global.privilegedServiceAccount.name` | Set this value to provide your own privileged service account. | `""` |
| `global.imagePullSecretName` | Set this value to use your own image pull secret that provides access to the cluster docker registry. If left blank, the the secret will resolve to sa-{{ .Release.Namespace }} | `""`|
| `global.tls.clusterDomain` | Cluster domain name that is used to generate a self-signed certificate.| `"cluster.local"` |


### Tooling parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `tooling.replicas` | Number of replicas of the pod. | `1` |
| `tooling.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"0.1"` |
| `tooling.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `2Gi` |
| `tooling.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"0.2"` |
| `tooling.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `4Gi` |
| `tooling.wcn.autoscaling.minReplicas` | Minimum number of available pod. This number should be less than or equal to the number of replicas. | `1` |
| `tooling.wcn.autoscaling.maxReplicas` | Maximum number of available pod. | `2` |
| `tooling.wcn.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `10m` |
| `tooling.wcn.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `80Mi` |
| `tooling.wcn.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `100m` |
| `tooling.wcn.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `100Mi` |


### Core parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `core.persistence.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |
| `core.common.persistence.storageClassName` | Storage class name for core common. `oketi-nfs` is supported. | `""` |
| `core.common.persistence.size` | Size of the persistent volume created. | `"100Gi"` |
| `core.gateway.replica` | Number of replicas of the pod. | `1` |
| `core.gateway.minAvailable` | Minimum number of available pod. This number should be less than or equal to the number of replicas. | `1` |
| `core.gateway.server.maxMemory` | Max memory size of server in the pod. | `"1024m"` |
| `core.gateway.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"1000m"` |
| `core.gateway.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `core.gateway.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `core.gateway.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `core.ingestion.replica` | Number of replicas of the pod. | `1` |
| `core.ingestion.minAvailable` | Minimum number of available pod. This number should be less than or equal to the number of replicas. | `1` |
| `core.ingestion.mount.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |
| `core.ingestion.mount.storageClassName` | Storage class name for core ingestion. `oketi-nfs` is supported. | `""` |
| `core.ingestion.mount.size` | Storage class name for core ingestion. `oketi-nfs` is supported. | `"1Gi"` |
| `core.ingestion.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"500m"` |
| `core.ingestion.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `core.ingestion.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `core.ingestion.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `core.orchestrator.replica` | Number of replicas of the pod. | `1` |
| `core.orchestrator.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"1000m"` |
| `core.orchestrator.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `core.orchestrator.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `core.orchestrator.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `core.wksml.replica` | Number of replicas of the pod. | `1` |
| `core.wksml.minAvailable` | Minimum number of available pod. This number should be less than or equal to the number of replicas. | `1` |
| `core.wksml.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"500m"` |
| `core.wksml.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `core.wksml.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `core.wksml.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |


### Elastic parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `elastic.replica` | Number of replicas of the pod. | `1` |
| `elastic.minAvailable` | Minimum number of available pod. This number should be less than or equal to the number of replicas. | `1` |
| `elastic.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"750m"` |
| `elastic.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `elastic.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `elastic.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `elastic.persistence.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |
| `elastic.persistence.storageClassName` |  Storage class name for elastic. `oketi-nfs` is supported. | `""` |
| `elastic.persistence.size` | Size of the persistent volume created. | `"10Gi"` |
| `elastic.backup.persistence.storageClassName` |  Storage class name for elastic backup. `oketi-nfs` is supported. | `""` |
| `elastic.backup.persistence.size` | Size of the persistent volume created for elastic backup. | `"30Gi"` |


### Etcd parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `etcd.replica` | Number of replicas of the pod. | `1` |
| `etcd.minAvailable` | Minimum number of available pod. This number should be less than or equal to the number of replicas. | `1` |
| `etcd.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"750m"` |
| `etcd.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `etcd.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `etcd.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `etcd.persistence.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |
| `etcd.persistence.storageClassName` |  Storage class name for etcd. `oketi-nfs` is supported. | `""` |
| `etcd.persistence.size` | Size of the persistent volume created. | `"10Gi` |


### Hdp parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `hdp.persistence.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |
| `hdp.persistence.storageClassName` |  Storage class name for etcd. `oketi-nfs` is supported. | `""` |
| `hdp.nn.replica` | Number of replicas of the pod. | `1` |
| `hdp.nn.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"500m"` |
| `hdp.nn.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `hdp.nn.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `hdp.nn.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `hdp.nn.persistence.size` | Size of the persistent volume created. | `"10Gi"` |
| `hdp.rm.replica` | Number of replicas of the pod. | `1` |
| `hdp.rm.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"500m"` |
| `hdp.rm.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `hdp.rm.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `hdp.rm.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `hdp.worker.replica` | Number of replicas of the pod. | `2` |
| `hdp.worker.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"500m"` |
| `hdp.worker.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `hdp.worker.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `hdp.worker.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `hdp.worker.persistence.size` | Size of the persistent volume created. | `"100Gi"` |


### Postgresql parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `postgresql.replica` | Number of replicas of the pod. | `1` |
| `postgresql.minAvailable` | Minimum number of available pod. This number should be less than or equal to the number of replicas. | `1` |
| `postgresql.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"750m"` |
| `postgresql.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `postgresql.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"99"` |
| `postgresql.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `postgresql.persistence.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |
| `postgresql.persistence.storageClassName` | Storage class name for etcd. `oketi-nfs` is supported. | `""` |
| `postgresql.persistence.size` | Size of the persistent volume created. | `"10Gi"` |


### SDU parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `sdu.sdu.replicas` | Number of replicas of the pod. | `1` |
| `sdu.sdu.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"2000m"` |
| `sdu.sdu.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2Gi"` |
| `sdu.sdu.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"4000m"` |
| `sdu.sdu.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `sdu.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"256Mi"` |
| `sdu.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `".1"` |
| `sdu.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"512Mi"` |
| `sdu.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `".3"` |


### Minio parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `minio.mode` | Minio server mode, `standalone` or `distributed`. | `"standalone"` |
| `minio.persistence.useDynamicProvisioning` | True to allow the cluster to automatically provision new storage resource and create PersistentVolume objects. | `true` |
| `minio.persistence.storageClass` |  Storage class name for etcd. `oketi-nfs` is supported. | `""` |
| `minio.persistence.size` | Size of the persistent volume created. | `"5Gi"` |
| `minio.replicas` | Number of replicas of the pod. | `1` |
| `minio.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"256Mi"` |
| `minio.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"250M"` |
| `minio.tls.clusterDomain` | Cluster domain name that is used to generate a self-signed certificate. Must be the same value as `global.tls.clusterDomain`.| `"cluster.local"` |


### WIRE parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `wire.trainingRest.trainingRestReplicas` | Number of replicas of the pod. | `1` |
| `wire.trainingRest.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"240m"` |
| `wire.trainingRest.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"1250Mi"` |
| `wire.trainingRest.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"480m"` |
| `wire.trainingRest.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2750Mi"` |
| `wire.trainingCrud.trainingCrudReplicas` | Number of replicas of the pod. | `1` |
| `wire.trainingCrud.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"300m"` |
| `wire.trainingCrud.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"3500Mi"` |
| `wire.trainingCrud.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"2010m"` |
| `wire.trainingCrud.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4700Mi"` |
| `wire.serveRanker.serveRankerReplicas` | Number of replicas of the pod. | `1` |
| `wire.serveRanker.mmRuntime.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"120m"` |
| `wire.serveRanker.mmRuntime.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"1000Mi"` |
| `wire.serveRanker.mmRuntime.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"240m"` |
| `wire.serveRanker.mmRuntime.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"1536Mi"` |
| `wire.serveRanker.mmServer.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"120m"` |
| `wire.serveRanker.mmServer.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"500Mi"` |
| `wire.serveRanker.mmServer.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"240m"` |
| `wire.serveRanker.mmServer.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"1024Mi"` |
| `wire.rankerRest.replicas` | Number of replicas of the pod. | `1` |
| `wire.rankerRest.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"120m"` |
| `wire.rankerRest.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"5000Mi"` |
| `wire.rankerRest.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"240m"` |
| `wire.rankerRest.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"7500Mi"` |
| `wire.dataPrepAgent.replicas` | Number of replicas of the pod. | `1` |
| `wire.dataPrepAgent.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"33m"` |
| `wire.dataPrepAgent.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"1000Mi"` |
| `wire.dataPrepAgent.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"66m"` |
| `wire.dataPrepAgent.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2000Mi"` |
| `wire.rankerMonitorAgent.replicas` | Number of replicas of the pod. | `1` |
| `wire.rankerMonitorAgent.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"7m"` |
| `wire.rankerMonitorAgent.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"50Mi"` |
| `wire.rankerMonitorAgent.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"34m"` |
| `wire.rankerMonitorAgent.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"100Mi"` |
| `wire.rankerCleanerAgent.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"7m"` |
| `wire.rankerCleanerAgent.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"50Mi"` |
| `wire.rankerCleanerAgent.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"34m"` |
| `wire.rankerCleanerAgent.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"100Mi"` |
| `wire.trainingDataDeletionAgent.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"7m"` |
| `wire.trainingDataDeletionAgent.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"50Mi"` |
| `wire.trainingDataDeletionAgent.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"34m"` |
| `wire.trainingDataDeletionAgent.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"100Mi"` |
| `wire.trainingAgent.replicas` | Number of replicas of the pod. | `1` |
| `wire.rankerMaster.etcdConfigJob.trainCPURequestMilli` | CPU Request in milli cpus for ranker training job | `2000` |
| `wire.rankerMaster.etcdConfigJob.trainCPULimitMilli` | CPU Limits in milli cpus for ranker training job | `2000` |
| `wire.rankerMaster.etcdConfigJob.trainingMemoryMB` | Memory Request in MB for ranker training job | `4096` |
| `wire.rankerMaster.replicas` | Number of replicas of the pod. | `1` |
| `wire.rankerMaster.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"120m"` |
| `wire.rankerMaster.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `wire.rankerMaster.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"240m"` |
| `wire.rankerMaster.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"4Gi"` |
| `wire.haywire.replicas` | Number of replicas of the pod. | `1` |
| `wire.haywire.resources.requests.cpu` | The minimum required CPU core. Specify integers, fractions (e.g. 0.5), or millicore values(e.g. 100m, where 100m is equivalent to .1 core). | `"84m"` |
| `wire.haywire.resources.requests.memory` | The minimum memory in bytes. Specify integers with one of these suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"1000Mi"` |
| `wire.haywire.resources.limits.cpu` | The upper limit of CPU core. Specify integers, fractions (e.g. 0.5), or millicores values(e.g. 100m, where 100m is equivalent to .1 core). | `"419m"` |
| `wire.haywire.resources.limits.memory` | The memory upper limit in bytes. Specify integers with suffixes: E, P, T, G, M, K, or power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. | `"2000Mi"` |


### AntiAffinity parameters

| Parameter                     | Description                                                           | Default |
| ----------------------------- | --------------------------------------------------------------------- | ------- |
| `core.gateway.antiAffinity`   | Anti-affinity policy for gateway pod. `soft` or `hard` can be set.    | `soft`  |
| `core.ingestion.antiAffinity` | Anti-affinity policy for ingestion pod. `soft` or `hard` can be set.  | `soft`  |
| `core.wksml.antiAffinity`     | Anti-affinity policy for wksml pod. `soft` or `hard` can be set.      | `soft`  |
| `elastic.antiAffinity`        | Anti-affinity policy for elastic pod. `soft` or `hard` can be set.    | `soft`  |
| `etcd.antiAffinity`           | Anti-affinity policy for etcd pod. `soft` or `hard` can be set.       | `soft`  |
| `hdp.worker.antiAffinity`     | Anti-affinity policy for hdp-worker pod. `soft` or `hard` can be set. | `soft`  |
| `postgresql.antiAffinity`     | Anti-affinity policy for postgres pod. `soft` or `hard` can be set.   | `soft`  |

Behavior of the policies:

| Policy    | Behavior |
|-----------|----------|
| soft      | Tries to create the pods on different nodes. If it is not possible, it will still create the pods.  |
| hard      | Tries to create the pods on different nodes. If it is co-located with others, it will not create a pod. |


### HA configuration

These are the recommended configuration settings for an HA deployment. Besides increasing the replica count for various micro-services, cpu and memory may also be increased if there are enough resources on the cluster. Furthermore, for true HA, the cluster requires 4 worker nodes because there are 4 minio pods and each minio pod must be scheduled on a different node.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `wire.haywire.replicas` | Set to `2` for HA. | `1` |
| `wire.rankerMaster.replicas` | Set to `2` for HA. | `1` |
| `wire.rankerRest.replicas` | Set to `2` for HA. | `1` |
| `wire.serveRanker.serveRankerReplicas` | Set to `2` for HA. | `1` |
| `wire.trainingCrud.trainingCrudReplicas` | Set to `2` for HA. | `1` |
| `wire.trainingRest.trainingRestReplicas` | Set to `2` for HA. | `1` |
| `sdu.sdu.replicas` | Set to `2` for HA. | `1` |
| `tooling.replicas` | Set to `2` for HA. | `1` |
| `elastic.replica` | Set to `2` for HA. | `1` |
| `etcd.replica` | Set to `3` for HA. | `1` |
| `core.gateway.replica` | Set to `2` for HA. | `1` |
| `core.ingestion.replica` | Set to `2` for HA. | `1` |
| `core.wksml.replica` | Set to `2` for HA. | `1` |
| `minio.replicas` | Set to `4` for HA. | `1` |
| `minio.mode` | Set to `distributed` for HA and when `minio.replicas` >= 4. | `"standalone"` |
| `minio.persistence.accessMode` | Set to `ReadWriteOnce` when `minio.mode=distributed` | `ReadWriteMany`|


## Limitations

- Watson Discovery can currently run only on Intel 64-bit architecture.
- This chart should only use the default image tags provided with the chart. Different image versions might not be compatible with different versions of this chart.
- Watson Discovery deployment supports a single service instance.
- The chart must be installed through the CLI.
- The chart must be installed by a ClusterAdministrator see [Pre-install steps](#pre-install-steps).
- This chart currently does not support upgrades or rollbacks. Please see the [product documentation](https://cloud.ibm.com/docs/services/discovery-data?topic=discovery-data-backup-restore) on backup and restore procedures.
- Release names cannot be longer than 20 characters

## Documentation

Find out more about IBM Watson Discovery by reading the [product documentation](https://cloud.ibm.com/docs/services/discovery-data).

## Integrate with other IBM Watson services

Watson Discovery is one of many IBM Watson services. Additional Watson services on the public IBM Cloud and IBM Cloud Private allow you to bring Watson's AI platform to your business application, and to store, train, and manage your data in the most secure cloud.

For the full list of available Watson services, see the IBM Watson catalog on the public IBM Cloud at [https://console.cloud.ibm.com/catalog/](https://console.cloud.ibm.com/catalog/?category=watson) (**Note**: This link takes you out of IBM Cloud Private to the public IBM Cloud.)

Watson services are currently organized into the following categories for different requirements and use cases:

  - **Assistant**: Integrate diverse conversation technology into your application
  - **Empathy**: Understand tone, personality, and emotional state
  - **Knowledge**: Get insights through accelerated data optimization capabilities
  - **Language**: Analyze text and extract metadata from unstructured content
  - **Speech**: Convert text and speech with the ability to customize models
  - **Vision**: Identify and tag content, then analyze and extract detailed information found in images

_Copyright©  IBM Corporation 2019. All Rights Reserved._
