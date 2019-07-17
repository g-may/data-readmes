# IBM Watson Knowledge Studio

## Introduction

Use IBM Watson™ Knowledge Studio to create a machine learning model that understands the linguistic nuances, meaning, and relationships specific to your industry or to create a rule-based model that finds entities in documents based on rules that you define.

To become a subject matter expert in a given industry or domain, Watson must be trained. You can facilitate the task of training Watson with Knowledge Studio.

## Chart Details

This chart installs an ICP4D add-on of Watson Knowledge Studio. Once the installation is completed, Watson Knowledge Studio add-on becomes available on ICP4D console.

This chart deploys the following microservices per install:

- **WKS Front-end**: Provides the WKS tooling GUI.
- **SIREG**: Tokenizers/parsers.
- **SIRE Job queue**: ML Training framework that allows to queue and schedule jobs on Kubernetes.
- **SIRE Train Facade**: Manages interaction with the training framework and Minio storage.
- **MMA**: Manages interaction with WKS Front-end and Train Facade.
- **Watson Add-on**: Publishes WKS service as ICP4D add-on.

This chart installs the following stores:

- **PostgreSQL**: Stores training metadata.
- **MongoDB**: Stores WKS project data.
- **Minio**: Stores ML models and training data.

## Prerequisites

- IBM Cloud Private 3.1.2
- Kubernetes 1.12.4 or later
- Helm 2.9.1 or later

## PodSecurityPolicy Requirements

This chart requires a PodSecurityPolicy to be bound to the target namespace prior to installation.  Choose either a predefined PodSecurityPolicy or have your cluster administrator create a custom PodSecurityPolicy for you:

- ICPv3.1 - Predefined  PodSecurityPolicy name: [`ibm-privileged-psp`](https://ibm.biz/cpkspec-psp)
- Custom PodSecurityPolicy definition:

```yaml
apiVersion: extensions/v1beta1
kind: PodSecurityPolicy
metadata:
  name: ibm-watson-ks-psp
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

## Resources Required

In addition to the [general hardware requirements and recommendations](https://www.ibm.com/support/knowledgecenter/en/SSQNUZ_2.1.0/com.ibm.icpdata.doc/zen/install/reqs-ent.html), this chart has the following requirements:

- x86 is the only architecture supported.
- Minimum CPU - 10
- Minimum Memory - 80Gi

### Storage

[Local volumes](https://kubernetes.io/docs/concepts/storage/volumes/#local) can be used as a storage type for this chart. Following numbers of persistent volumes are required by data stores.

| Component  | Number of replicas | Space per PVC | Storage type  | Number of PVs |
| ---------- | :----------------: | ------------- | ------------- | :-----------: |
| PostgreSQL |         2          | 10 Gi         | local-storage |       2       |
| Minio      |         4          | 10 Gi         | local-storage |       4       |
| MongoDB    |         2          | 20 Gi         | local-storage |       2       |

## Installing the Chart

### Setup environment

1. Initialize Helm client by running the following command. For further details of Helm CLI setup, see [Installing the Helm CLI (helm)](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/app_center/create_helm_cli.html)

    ```bash
    export HELM_HOME={helm_home_dir}
    helm init --client-only
    ```

     - `{helm_home_dir}` is your Helm config directory. For example `~/.helm`.

1. Login to the ICP cluster and target the namespace the chart will be deployed using `cloudctl`. See the [installation documentation](https://cloud.ibm.com/docs/services/watson-knowledge-studio-data?topic=watson-knowledge-studio-data-install) for more detail.

    ```bash
    cloudctl login -a https://{cluster_CA_domain}:8443 -u {user} -p {password}
    ```

1. Create a Kubernetes namespace with `ibm-priveleged-psp` PSP. See the [installation documentation](https://cloud.ibm.com/docs/services/watson-knowledge-studio-data?topic=watson-knowledge-studio-data-install) for more detail.

1. Get a certificate from your cluster and install it to Docker or add the {cluster_CA_domain} as a Docker Daemon insecure registry. You must do one or the other for Docker to be able to pull from your cluster.

    See [Configuring authentication for the Docker CLI](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/manage_images/configuring_docker_cli.html).

1. Login to the docker registry like the following.

    ```bash
    docker login {cluster_CA_domain}:8500
    ```

1. To load the file from Passport Advantage into IBM Cloud Private, enter the following command in the IBM Cloud Private command line interface.

    ```bash
    cloudctl target -n {namespace_name}
    cloudctl catalog load-archive --archive {compressed_file_name} --registry {cluster_CA_domain}:8500/{namespace_name}
    ```

    - `{compressed_file_name}` is the name of the file that you downloaded from Passport Advantage.
    - `{namespace_name}` is the Docker namespace that hosts the Docker image. This is the namespace you created in Step 1.
1. Once loading PPA is finished, the Helm chart of the IBM Watson Knowledge Studio is added to the local Helm repository of the cluster. To install the chart, add cluster local helm repository `local-charts` to your Helm CLI. See [Adding the internal Helm repository to Helm CLI](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.2/app_center/add_int_helm_repo_to_cli.html) for more detail.

    ```bash
    sudo helm repo add local-charts https://{cluster_CA_domain}:8443/helm-repo/charts --ca-file $HELM_HOME/ca.pem --cert-file $HELM_HOME/cert.pem --key-file $HELM_HOME/key.pem
    ```

1. Create a YAML file like the following, then run the apply command on the YAML file that you create. For example `kubectl apply -f image_policy.yaml`. For further details of ImagePolicy, see [Enforcing container image security](https://www.ibm.com/support/knowledgecenter/en/SSBS6K_3.1.2/manage_images/image_security.html)

    ```yaml
    apiVersion: securityenforcement.admission.cloud.ibm.com/v1beta1
    kind: ImagePolicy
    metadata:
      name: icp-image-policy
      namespace: {namespace}
    spec:
      repositories:
      - name: '{cluster_CA_domain}:8500/{namespace}/*'
    ```

    - `{namespace}`: your target namespace
    - `{cluster_CA_domain}`: your cluster CA domain name

1. To meet network security policy for ICP4D addon, update label of the namespaces by the following 2 commands.

    ```bash
    kubectl label --overwrite namespace zen ns=zen
    kubectl label --overwrite namespace {namespace} ns={namespace}
    ```

### Preparing persistent volumes

This chart uses persistent volumes for data stores. Local storage can be used for this chart. To create a persistent volume with `local-storage` storage class, you can define each volume configuration in a YAML file, and then use the `kubectl` command line to push the configuration changes to the cluster. See [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/volumes/#local) for more details about local storage.

1. Create a `local-storage` storage class on your cluster. See [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/storage-classes/#local) for more detail.

    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: local-storage
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
    ```

1. Prepare directories to be used by PVs on worker nodes.

1. Create 8 YAML files, one for each volume. The Number of required volumes can be different if you change the number of replicas of data stores. Default is 8.

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: {name}
    spec:
      capacity:
        storage: {size}Gi
      accessModes:
      - ReadWriteOnce
      persistentVolumeReclaimPolicy: Retain
      storageClassName: local-storage
      local:
        path: {path}
      nodeAffinity:
        required:
          nodeSelectorTerms:
          - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
              - {ip}
    ```

    - `{name}`: The name of the PV to be created.
    - `{size}`: Reflect the size specified in the storage requirement table above.
    - `{path}`: Path on the worker node where you create in the previous step. For example, `/mnt/local-storage/storage/{dir_name}`. For `{dir_name}`, use the same value that you use for `{name}` so you can map the volume name to its physical location.
    - `{ip}`: IP address of the worker node where you create the persistent volume.

1. Run the apply command on each YAML file that you create.

    For example: `kubectl apply -f pv_001.yaml`. Rerun this command for each file up to `kubectl apply -f pv_07.yaml`.

See the [installation document](https://cloud.ibm.com/docs/services/watson-knowledge-studio-data?topic=watson-knowledge-studio-data-install) for more detail.

### Deployment

To install the chart, run the following command and follow the instruction:

```bash
helm install local-charts/ibm-watson-ks-prod -n {release_name}  --timeout 1800 --tls
```

- `{release_name}` is the Helm release name of this installation.

### Verification of deployment

After installation completed successfully, you can verify the deployment by running `helm test`.

```bash
helm test {release_name} --cleanup --timeout 600 --tls
```

### Accessing WKS tooling UI

After the successful installation, a WKS add-on tile with the release name is shown up on your ICP4D console. You can provision a WKS instance and launch your WKS tooling application there.

1. Open `https://{cluster_CA_domain}:31843` by your web browser and login to ICP4D console.

1. Move to Add-on catalog. You can find the add-on of IBM Watson Knowledge Studio with the release name.

1. Click the Watson Knowledge Studio add-on tile and provision an instance.

1. Open the created instance and click `Launch Tool` button.

1. You can start using IBM Watson Knowledge Studio.

## Uninstalling the chart

1. Delete the existing WKS instances of the release on ICP4D console.

1. Run the following command to uninstall and delete the existing release.

    ```bash
    helm delete --tls {release_name}
    ```

    - `{release_name}` is the Helm release name of this installation.

    To irrevocably uninstall and delete the release, run the following command:

    ```bash
    helm delete --tls --no-hooks --purge {release_name}
    ```

If you omit the `--purge` option, Helm deletes all resources for the deployment but retains the record with the release name. This approach allows you to roll back the deletion. If you include the `--purge` option, Helm removes all records for the deployment so that the name can be used for another installation.

Before you can install a new version of the service on the same cluster, you must remove all content from any persistent volumes and persistent volume claims that were used for the previous deployment.

## Configuration

The following table lists the configurable parameters of the WKS chart and their default values.

### Global parameters

|    Parameter     |                                              Description                                              | Default |
| ---------------- | ----------------------------------------------------------------------------------------------------- | ------- |
| `license`        | Set `accept` if you accept the license agreement. The chart can be installed without this acceptance. | `""`    |

### WKS Front-end parameters

|       Parameter        |                                                           Description                                                            | Default |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `replicaCount`         | Number of replicas for WKS Front-end deployment.                                                                                  | `2`     |
| `frontend.initialUser` | Username of the first admin user in the WKS instance. The user also must be granted the access to the instance on ICP4D console. | `admin` |

### SIRE parameters

|                          Parameter                          |                                                                                          Description                                                                                           | Default |
| ----------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- |
| `global.highAvailabilityMode`                               | It this values is `true`, multiple replicas are created for SIRE training service (SIREG, SIRE Job queue, SIRE Train Facade). If set `false`, a single replica is created for each deployment. | `true`  |
| `sire.jobq.tenants.train.max_queued_and_active_per_user`    | Maximum amount of queued and active training jobs, any additional jobs will be rejected with an error.                                                                                        | `10`    |
| `sire.jobq.tenants.train.max_active_per_user`               | Maximum amount of training jobs running in parallel even if enough cpu/mem quota available.                                                                                                    | `2`     |
| `sire.jobq.tenants.evaluate.max_queued_and_active_per_user` | Maximum amount of queued and active evaluate jobs, any additional jobs will be rejected with an error.                                                                                        | `10`    |
| `sire.jobq.tenants.evaluate.max_active_per_user`            | Maximum amount of evaluate jobs running in parallel even if enough cpu/mem quota available.                                                                                                    | `2`     |
| `sireg.languages.en.enabled`                                | Toggle language support for English                                                                                                                                                            | `true`  |
| `sireg.languages.es.enabled`                                | Toggle language support for Spanish                                                                                                                                                            | `true`  |
| `sireg.languages.ar.enabled`                                | Toggle language support for Arabic                                                                                                                                                             | `true`  |
| `sireg.languages.de.enabled`                                | Toggle language support for German                                                                                                                                                             | `true`  |
| `sireg.languages.fr.enabled`                                | Toggle language support for French                                                                                                                                                             | `true`  |
| `sireg.languages.it.enabled`                                | Toggle language support for Italian                                                                                                                                                            | `true`  |
| `sireg.languages.ja.enabled`                                | Toggle language support for Japanese                                                                                                                                                           | `true`  |
| `sireg.languages.ko.enabled`                                | Toggle language support for Korean                                                                                                                                                             | `true`  |
| `sireg.languages.nl.enabled`                                | Toggle language support for Dutch                                                                                                                                                              | `true`  |
| `sireg.languages.pt.enabled`                                | Toggle language support for Portuguese                                                                                                                                                         | `true`  |
| `sireg.languages.zh.enabled`                                | Toggle language support for Chinese (simplified)                                                                                                                                               | `true`  |
| `sireg.languages.zht.enabled`                               | Toggle language support for Chinese (traditional)                                                                                                                                              | `true`  |

### MMA parameters

|   Parameter    |        Description         | Default |
| -------------- | -------------------------- | ------- |
| `mma.replicas` | Number of replicas for MMA | `2`     |

### MongoDB parameters

|                 Parameter                 |                                            Description                                             |      Default      |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------- | ----------------- |
| `mongodb.replicas`                        | Number of replicas for MongoDB StatefulSet.                                                       | `2`               |
| `mongodb.persistentVolume.enabled`        | If `true`, persistent volume claims are created for MongoDB.                                       | `true`            |
| `mongodb.persistenceVolume.useDynamicProvisioning` | If enabled, the PVC will use a storageClassName to bind the volume for MongoDB.                                                                                                                                                                         | `true`         |
| `mongodb.persistentVolume.storageClass`   | Persistent volume storage class for MongoDB.                                                       | `local-storage`              |
| `mongodb.persistentVolume.accessMode`     | Persistent volume access modes for MongoDB.                                                        | `[ReadWriteOnce]` |
| `mongodb.persistentVolume.size`           | Persistent volume size for MongoDB.                                                                | `20Gi`            |
| `mongodb.persistentVolume.selector.label` | Label for persistent volume claim selectors to control how pvc's bind/reserve storage for MongoDB. | `""`              |
| `mongodb.persistentVolume.selector.value` | Value for persistent volume claim selectors to control how pvc's bind/reserve storage for MongoDB. | `""`              |
| `mongodb.persistentVolume.annotations`    | Persistent volume annotations for MongoDB.                                                         | `{}`              |

### Minio (Object Storage) parameters

|                 Parameter                  |                                                                                                                      Description                                                                                                                      |     Default     |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| `minio.mode`                               | Minio server mode. Valid options are `standalone` or `distributed`.                                                                                                                                                                                   | `distributed`   |
| `minio.replicas`                           | Number of nodes (applicable only to Minio distributed mode). Must be 4 <= x <= 32                                                                                                                                                                     | `4`             |
| `minio.persistence.enabled`                | Use PV to store data on Minio.                                                                                                                                                                                                                        | `true`          |
| `minio.persistence.size`                   | Size of persistent volume claim (PVC) for Minio.                                                                                                                                                                                                      | `10Gi`          |
| `minio.persistence.existingClaim`          | Use an existing PVC to persist data for Minio.                                                                                                                                                                                                        | `""`            |
| `minio.persistence.useDynamicProvisioning` | If enabled, the PVC will use a storageClassName to bind the volume for Minio.                                                                                                                                                                         | `true`         |
| `minio.persistence.storageClass`           | Storage Class to bind PVC for Minio. You must specify a valid storage class if you selected `useDynamicProvisioning`.                                                                                                                                 | `local-storage`          |
| `minio.persistence.accessMode`             | Persistent volume storage class for Minio. `ReadWriteOnce` or `ReadOnly`                                                                                                                                                                              | `ReadWriteOnce` |
| `minio.persistence.subPath`                | Mount a sub directory of the persistent volume for Minio, if a sub directory is set.                                                                                                                                                                  | `""`            |

### PostgreSQL parameters

|              Parameter               |                                                                                                                                   Description                                                                                                                                    |     Default     |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| `postgresql.sentinel.replicas`                  | Number of sentinel nodes                                                                                                                                                                                                                                                         | `2`             |
| `postgresql.proxy.replicas`                     | Number of proxy nodes                                                                                                                                                                                                                                                            | `2`             |
| `postgresql.keeper.replicas`                    | Number of keeper nodes                                                                                                                                                                                                                                                           | `2`             |
| `postgresql.persistence.enabled`                | Use a PVC to persist data                                                                                                                                                                                                                                                        | `true`          |
| `postgresql.persistence.useDynamicProvisioning` | Enables dynamic binding of Persistent Volume Claims to Persistent Volumes                                                                                                                                                                                                        | `true`          |
| `postgresql.persistence.storageClassName`       | Storage class name of backing PVC                                                                                                                                                                                                                                                | `local-storage` |
| `postgresql.persistence.accessMode`             | Use volume as ReadOnly or ReadWrite                                                                                                                                                                                                                                              | `ReadWriteOnce` |
| `postgresql.persistence.size`                   | Size of data volume                                                                                                                                                                                                                                                              | `10Gi`          |
| `postgresql.dataPVC.name`                       | Prefix that gets the created Persistent Volume Claims                                                                                                                                                                                                                            | `stolon-data`   |
| `postgresql.dataPVC.selector.label`             | In case the persistence is enabled and useDynamicProvisioning is disabled the labels can be used to automatically bound persistent volumes claims to precreated persistent volumes. The persistent volumes to be used must have the specified label. Disabled if label is empty. | `""`            |
| `postgresql.dataPVC.selector.value`             | In case the persistence is enabled and useDynamicProvisioning is disabled the labels can be used to automatically bound persistent volumes claims to precreated persistent volumes. The persistent volumes to be used must have label with the specified value.                  | `""`            |

## Limitations

- Only the `x86` architecture is supported.
- The chart must be installed by a ClusterAdministrator.

## Documentation

Find out more about IBM Watson Knowledge Studio by reading the [product documentation](https://cloud.ibm.com/docs/services/watson-knowledge-studio-data?topic=watson-knowledge-studio-data-wks_overview_full).

**Note**: The documentation link takes you out of IBM Cloud Private to the public IBM Cloud.