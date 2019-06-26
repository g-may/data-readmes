# IBM Watson™ Speech Services 1.0.0

## Introduction

This document contains installation instructions for both IBM Watson™ Speech to Text and Text to Speech solutions.

IBM Watson™ Speech to Text (*STT*) provides speech recognition capabilities for your solutions. The service leverages machine learning to combine knowledge of grammar, language structure, and the composition of audio and voice signals to accurately transcribe the human voice. It continuously updates and refines its transcription as it receives more speech.

IBM Watson™ Text to Speech (*TTS*) service converts written text to natural-sounding speech to provide speech-synthesis capabilities for applications. It gives you the freedom to customize your own preferred speech in different languages. This cURL-based tutorial can help you get started quickly with the service.

## Chart details

This chart can be used to install a single instance of both the *STT* and *TTS* solutions. The solutions can be installed separately, however if both are installed together they share datastores for a more efficient utilization of resources and simplified support.

## Prerequisites

- IBM Cloud Private for Data 2.1.0
- Kubernetes 1.12.4 or later
- Helm 2.9.1 or later

Because the installation of the chart needs to be performed from the command line, __you must install and configure [helm](https:helm.sh) and [kubectl](https://docs-icpdata.mybluemix.net/docs/content/SSQNUZ_current/com.ibm.icpdata.doc/zen/install/kubectl-access.html)__.

## Select the components to install

This chart can be used to install both the *STT* and *TTS* solutions, the following tags, which are defined in the `values.yaml` file, can be used to enable the installation of each of the different components shipped in this solution:

```yml
tags:
  sttAsync: true
  sttCustomization: true
  ttsCustomization: true
  sttRuntime: true
  ttsRuntime: true
```

- `sttAsync` enables the installation of the asynchronous API to access the STT service, which corresponds to the `/recognitions` API endpoints.
- `sttCustomization` enables the installation of the STT customization functionality, which lets you customize the STT base models for improved accuracy and corresponds to the `/customizations` API endpoints.
- `ttsCustomization` enables the installation of the TTS customization functionality, which lets you customize the TTS base voices for improved voice quality and corresponds to the `/customizations` API endpoints.
- `sttRuntime` enables the installation of the core STT functionality, which lets you convert speech into text using the `/recognize` endpoint. Note that this component is installed if either the `sttRuntime`, `sttCustomization`, or `sttAsync` tags are set to `true`.
- `ttsRuntime` enables the installation of the core TTS functionality, which lets you convert text into speech using the `/synthesize` endpoint.  Note that this component is installed if either the `ttsRuntime` or `ttsCustomization` tags are set to `true`.

By default all the components are enabled, but each of them can be disabled separately. If you want to install *STT* only, you need to set `ttsRuntime` and `ttsCustomization` to `false`. Similarly, if you want to install *TTS* only, you need to set `sttRuntime`, `sttCustomization` and `sttAsync` to `false`. For example, if you want to install *STT* and *TTS* but do not want customization capabilities, then you need to set `sttCustomization` and `ttsCustomization` to `false`.


## Namespace

The first step consists of creating the Kubernetes namespace where the solution is installed. The example that follows shows how to create the namespace, which in this case is named `speech-services`. You can use a different name for the namespace.

Copy the following YAML to a file, then run the command that follows the YAML.

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: speech-services
```

```bash
kubectl create -f {namespace_file}
```

## Minio object storage

[TODO]

By default, Minio object storage, which is used for binary storage, is included in the  installation. However, you can choose to use your own Minio installation instead. The following sections describe both options.

### Using the default Minio installation

The default Minio service needs a persistent volume for storing data. Create a storage class named `local-storage` by copying the following YAML code to a file and running `kubectl apply -f {file}`:

```yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Create a persistent volume by using the previously created storage class. Copy the following YAML code to a file and run `kubectl apply -f {file}`:

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: speech-minio-pv
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 100Gi
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /mnt/local-storage/speech/minio/pv
    type: ""
```

Note that the storage size, `storage:`, must be at least the same size that you specify during installation. See the *Storage size calculation guidelines* section for information about the storage space that is required to install this product.

Before you install *STT-CC*, you need to provide a secret object that is used by Minio itself and by other service components that interact with Minio. This secret contains the security keys to access Minio.

The secret must contain the items `accesskey` (5 - 20 characters) and `secretkey` (8 - 40 characters) in base64 encoding. Therefore, before creating the secret, you need to perform the base64 encoding.

The following commands encode the `accesskey` and `secretkey` in base64. **Important:** For security reasons, you are strongly encouraged to create an `accesskey` and a `secretkey` that are different from the sample keys (`admin` and `admin1234`) that are shown in the examples.

```sh
echo -n "admin" | base64
YWRtaW4=

echo -n "admin1234" | base64
YWRtaW4xMjM0
```

Create the following secret object:

```yml
apiVersion: v1
kind: Secret
metadata:
  name: minio
type: Opaque
data:
  accesskey: YWRtaW4=
  secretkey: YWRtaW4xMjM0
```

You can create this secret object through the ICP Console by clicking on the configuration and the `Secrets` menu. You can also run `kubectl create -f {secrets-file}` on the command line.

In the configuration page of the Minio Chart, before installation, you also need to configure the following:

-   Check the `Use the Default Minio Installation` option [`global.datastore.minio.enabled=true`].
-   Pass the name (for example, `minio` from the previous example) of the secret object that contains the keys (this is the secret object that you just created) in the following fields:
    -   In the `Global Settings | Minio Access Secret Name` field [`global.datastores.minio.secretName=minio`]
    -   In the `Minio Object Storage | Minio Access Secret Name` field [`external.minio.minioAccessSecret=minio`]
-   Under `Minio Object Storage | Persistent Volume Size`, enter `100Gi` [`external.minio.persistence.size=100Gi`]. See the *Storage size calculation guidelines* section for information about the storage space that is required to install this product.

Optionally, you can choose to set Minio server mode to `distributed` if you need high availability:

  - Select the `distributed` option in `Minio Object Storage | Minio Server Mode` [`external.minio.mode=distributed`].
  - Set the number of replicas for distributed mode `Minio Object Storage | Number of Replicas`. This value is the number of cluster nodes; it must be `4 <= x <= 32` [`externam.minio.replicas=4`].

### Storage size calculation guidelines

Object storage is used for storing binary data from the following sources:

  - Base models (for example `en_US-NarrowbandModel`)

    On average, base models are each `1.5 GB`. Because models are updated regularly, you need to multiply that amount by three to make room for at least three different versions of each model.

  - Customization data (audio files and training snapshots)

    The storage required for customization data depends on how many hours of audio you use for training your custom models. On average, one hour of audio data needs `0.5 GB` of storage space. You can have multiple customizations, so you must factor in additional space.

  - Audio files from recognition jobs that are processed asynchronously, in case they need to be queued

    The storage required for asynchronous jobs depends on the use case. If you plan to submit large batches of audio files, expect the service to queue some jobs temporarily. This means that some audio files are held temporarily in binary storage. The amount of storage required for this purpose does not exceed the size of the largest batch of jobs that you plan to submit in parallel.

A few examples of how to calculate storage size [GB] follow:

-   6 models, 3 versions, 50 hours audio = `6 * 1.5 * 3 + 50 * 0.5 = 52`
-   2 models, 3 versions, 20 hours audio = `2 * 1.5 * 3 + 20 * 0.5 = 19`

The default storage size, `100 GB`, is a minimal starting point and is typically enough for operations with two to six models and about 50 hours of audio for the purpose of training custom models. That said, it is always a good idea to be generous in anticipation of future storage needs.

## Configuration of the PostgresSQL and RabbitMQ installation

### Create Local Persistent Volumes to persist data

If either STT customization, TTS customization or STT async are included in the installation (see the previous section: *Select the Components to Install*) an instance of the PostgresSQL database is installed. Additionally, if the STT async component is included in the installation, an instance of the RabbitMQ datastore is installed. The datastores are stateful and need to leverage Kubernetes persistent volumes to persist their data.

PostgresSQL and RabbitMQ require the availability of Kubernetes Local Persistent Volumes, which must be created before you install the chart. The volumes are used to persist the data used by the datastores so if a container restarts it can reattach to the original data. Given that both PostgresSQL and RabbitMQ are configured by default for high availability (HA) with three replicas, a minimum of three volumes need to be installed for each of the datastores.

```yml
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
  storageClassName: local-storage-local
  local:
    path: {path}
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - {node}
```

where:
  - `{name}` is the name of the Persistent Volume (PV) to be created, and needs to be unique.
  - `{size}` is the disk space that is allocated for the persistent volume.
  - `{path}` is the path in the host machine where the persistent volume is created; for example, `/mnt/local-storage/storage/pv_1`. Note that the permissions of this directory need to be open enough to allow non-root access; otherwise, pods running as non-root are unable to mount the volume in the directory.
  - `{node}` is the kubernetes *worker* node where the Local Volume is to be created. You can list the available nodes in your cluster by running `kubectl get nodes`.

Given that PostgresSQL and RabbitMQ pods are scheduled in the worker nodes where the PVs are created, the PVs need to be created in different nodes so in case a node goes down there are still at least two healthy replicas running. Recall that a minimum of three replicas are needed for high availability.

### Setting access credentials for PostgresSQL

The Postgres chart reads the credentials to access the Postgres database from the following secret file, which needs to be created before installing the chart. You need to set the attribute `data.pg_su_password` to the Postgres password that you want (base64 encoded). You also need to set the attribute `pg_repl_password`, which is the replication password and is also base64 encoded, to the value you want.

```yml
apiVersion: v1
data:
  pg_repl_password: cmVwbHVzZXI=
  pg_su_password: c3RvbG9u
kind: Secret
metadata:
  name: user-provided-postgressql # this name can be anything you choose
type: Opaque
```

To create this secret object you need to execute the command `kubectl create -f {secrets_file}`.

Finally, when installing the chart you need to set the following two values to the name of the secret created previously (`user-provided-postgressql`): `global.datastores.postgressql.auth.authSecretName` and `postgressql.auth.authSecretName`.

If you do not create the secret object, the system creates a secret object that contains randomly generated passwords when the Helm chart is installed. For security reasons you need to change the automatically generated passwords when the deployment is complete.

## Resources Required

In addition to the general requirements listed in [Pre-installation tasks](https://docs-icpdata.mybluemix.net/docs/content/SSQNUZ_current/com.ibm.icpdata.doc/zen/install/preinstall-overview.html), the IBM Watson Speech to Text service has its own requirements:

  - **x86** is the only architecture supported at this time.
  - If you need a highly available installation a minimum of three worker nodes are needed for the installation.
  - The resources required for the installation, in terms of CPUs and memory, depend on the configuration that you select. There are two typical installation configurations:
    - The **development configuration**, which is the configuration that is used in the default installation, has a minimal footprint and is meant for development purposes and as a proof of concept. It can handle several concurrent recognition sessions only and it is not highly available since some of the core component have no redundancy (single replica).
    - The **production configuration** is a highly available solution that is intended to run production workloads. This configuration can be achieved by scaling up the **development configuration** after installation, see the next section.

### Resources required by the standard IBM Speech To Text installation

While the default installation of the solution comes with the **development configuration**, you can easily obtain the **production configuration** by scaling up the number of pods/replicas of the deployment objects after installing the solution. How much to scale up each of the components depends on the degree of concurrency you need, and is limited by the amount of hardware resources available in your Kubernetes cluster/namespace.

##### Scaling up the PostgresSQL and RabbitMQ datastores

By default both PostgreSQL and RabbitMQ are installed with three replicas for high availability reasons. Each replica is typically scheduled within a different Kubernetes worker node if resources allow. Before performing the installation, you can configure the number of replicas and the CPU and memory resources for each replica by using Helm values (see the *Options* section).

You can also scale up the datastores on an already running solution by changing the number of replicas in the Deployment or StatefulSet objects. For example, you can scale up RabbitMQ as follows:

  1. Edit the StatefulSet object by running `kubectl edit statefulsets {release}-ibm-rabbitmq`.
  1. Change the value of the `spec.replicas:` attribute.
  1. Save and close the StatefulSet object.

In the case of PostgresSQL there are two deployment objects (`{release}-ibm-postgresql-proxy` and `{release}-ibm-postgresql-sentinel`) and a StatefulSet (`ibm-wc-ibm-postgresql-keeper`). The deployment objects can be scaled up by simply running `kubectl scale --replicas={n} {deployment_object}` where `{n}` is the new number of replicas; for example, `kubectl scale --replicas=3 deployment ibm-wc-ibm-postgresql-proxy`. The StatefulSet object can be scaled up following the process described previously for PostgreSQL.  

Note that a sufficient number of Persistent Local Volumes need to be created before scaling up the number of replicas (in the case of the StatefulSets) so the newly created pods can mount their volumes.

##### Scaling up the rest of the solution

You can learn about the list of deployments (Kubernetes `Deployment` objects) by running the `kubectl get deployment` command. You can then scale up the number of pods on each of the deployment objects to match the number of pods in the production configuration, as shown in the following table. You can do this by using the following command: `kubectl scale --replicas={n} deployment {deployment_object}`, where `{n}` is the desired number of replicas for the given deployment (`{deployment_object}`).

| Deployment              | default # replicas |
|-------------------------|-------------------|
| {release}-speech-to-text-stt-runtime          |  1 |
| {release}-speech-to-text-stt-customization    |  1 |
| {release}-speech-to-text-stt-am-patcher       |  1 |
| {release}-speech-to-text-stt-async            |  1 |
| {release}-speech-to-text-gdpr-data-deletion   |  1 |
| {release}-minio                               |  1 |
| {release}-rabbitmq                            |  1 |
| {release}-ibm-postgressql-proxy                   |  2 |
| {release}-ibm-postgressql-sentinel                |  3 |
| {release}-ibm-postgressql-keeper                  |  3 |

| Statefulset                  | # replicas |
|-------------------------|-------------------|
| ibm-wc-ibm-postgresql-keeper | 3          |
| ibm-wc-ibm-rabbitmq          | 3          |

The standard installation (*development configuration*) requires a total of **14.75** CPUs and **38.5** GBs of memory. These numbers are based on a standard installation that includes the US English models only. In general, the memory requirements vary depending on which models you include in the installation.

#### Setting the sessions/CPU ratio

[TODO]

### Dynamic resource calculation

[TODO]

For the `speech-to-text-stt-runtime` and `speech-to-text-stt-am-patcher` deployments, you can enable automatic resource calculation, which is based on the selected number of CPUs and the selected language models. Automatic resource calculation is enabled by default. You can modify this behavior by checking/unchecking `STT Runtime | Dynamic Memory Calculation (STT Runtime)` and `STT Runtime | Dynamic Memory Calculation (AMC Patcher)`, respectively.

## PodSecurityPolicy Requirements

The predefined PodSecurityPolicy name: [`ibm-anyuid-psp`](https://ibm.biz/cpkspec-psp) has been verified for this chart. By default the chart automatically installs the necessary RBAC roles and rolebindings for running the service.

Custom PodSecurityPolicy definition:
```yml
---
```

## Pre-install steps

Run: `./ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration/labelNamespace.sh ICP4D_NAMESPACE` where `ICP4D_NAMESPACE` is the namespace where ICP4D is installed (usually `zen`).

The `ICP4D_NAMESPACE` namespace **must** have a label for the `NetworkPolicy` to correctly work. Only `nginx` and `zen` pods are allowed to communicate with the pods in the namespace where this chart is installed.


## Installing the chart

[TODO]

Installing the Helm chart deploys a single IBM Watson™ Speech Services solution with the **development configuration**. You can then scale up this configuration to support up to 50 concurrent recognition sessions (see the earlier sections).

To install the chart, run the following command:

```bash
helm install --tls --name {my_release} -f {my_values.yaml} ibm-watson-speech-prod
```

-   Replace `{my_release}` with a name for your release.
-   Replace `{my_values.yaml}` with the path to a YAML file that specifies the values that are to be used with the `install` command. Specifying a YAML file is optional.

When the command completes, its output shows the current status of the release.

When the installation has completed and all the pods are in ready state, a series of Helm tests are available to validate the installation. They can be executed by using the following command:

```sh
helm test --tls --name {my_release}
```

**Important security notice:** After completing the installation, it is strongly recommended that you manually change any autogenerated passwords or certificate. If not, the Helm CLI can allow a user with operator role to see the password or certificate, which represents a security risk.

### PodSecurityPolicy requirements

This chart requires a PodSecurityPolicy to be bound to the target namespace prior to installation. This chart has been verified with the predefined [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp)' PodSecurityPolicy. Choose either a predefined PodSecurityPolicy or have your cluster administrator create a custom PodSecurityPolicy for you:

Custom PodSecurityPolicy definition:
```yml
---
```

### Uninstalling the chart

To uninstall and delete the `my_release` deployment, run the following command:

```bash
helm delete --tls my_release
```

To irrevocably uninstall and delete the `my_release` deployment, run the following command:

```bash
helm delete --purge --tls my_release
```

If you omit the `--purge` option, Helm deletes all resources for the deployment but retains the record with the release name. This allows you to roll back the deletion. If you include the `--purge` option, Helm removes all records for the deployment so that the name can be used for another installation.

## Configuration

[TODO]

The Helm chart has the following values that you can override by using the `--set` parameter with the `install` command. For example:

```bash
helm install --tls --set image.repository={my_image} stable/ibm-datapower-dev
```

Alternatively, you can provide a YAML file that specifies the values with the `install` command. For example:

```bash
helm install --tls -f {my_values.yaml}
```

### Language model selection

[TODO]

You can perform an installation that includes only a subset of the language models in the catalog. Installing all of the models in the catalog substantially increases the memory requirements. Therefore, it is strongly recommended that you install only those languages that you plan to use.

You can select the languages to be installed by checking/unchecking each of the models in `Global settings | {model_name} Model` options. By default, the dynamic resource calculation feature is enabled; it automatically computes the exact amount of memory that is required for the selected models.

-------->>> talk here about selecting models installed from an external PPA

### Storage of customer data (STT Runtime and AMC Patcher)

By default, payload data, including audio files, recognition hypotheses, and annotations, are temporarily stored in the running container. You can disable this behavior by checking `STT Runtime | Disable storage of customer data` option. Checking this option also removes sensitive information from container logs.

### Options

[TODO]

The following options apply to an IBM Watson Speech Services runtime configuration.

| Value                                                   | Description                                              | Default             |
|---------------------------------------------------------|----------------------------------------------------------|---------------------|
| `chuck.anonymizeLogs`                                     | Opt out of runtime logs and audio data.                                                                                                                                                                                                               | `False`                                                   |
| `chuck.groups.sttAMPatcher.resources.dynamicMemory`       | Calculate memory requirements for STT AMC patcher according to selected models. For more information, see the chart Overview.                                                                                                                                     | `True`                                                    |
| `chuck.groups.sttAMPatcher.resources.requestsCpu`         | Every customization session needs 4 CPUs.                                                                                                                                                                                                             | `8`                                                       |
| `chuck.groups.sttAMPatcher.resources.requestsMemory`      | Amount of memory depends on number of CPUs. Size can be calculated as #CPUs * 3 GB.                                                                                                                                                                    | `22000Mi`                                                 |
| `chuck.groups.sttAMPatcher.resources.threads`             | Number of parallel-processing threads for AMC. Note that fewer threads means longer training time.                                                                                                                                                    | `4`                                                       |
| `chuck.groups.sttRuntimeDefault.resources.dynamicMemory`  | Calculate memory requirements for STT runtime according to selected models. For more information, see the chart Overview.                                                                                                                                         | `True`                                                    |
| `chuck.groups.sttRuntimeDefault.resources.requestsCpu`    | Requested CPUs for STT runtime. Minimal value is 4.                                                                                                                                                                                                   | `8`                                                       |
| `chuck.groups.sttRuntimeDefault.resources.requestsMemory` | Calculation of the memory requirements can be found in the chart Overview. When dynamic memory is enabled, this option has no effect.                                                                                                                 | `22000Mi`                                                 |
| `external.minio.minioAccessSecret`                        | Create a secret that contains base64-encoded accesskey (5 - 20 characters) and secretkey (8 - 40 characters). The keys are used to access the Minio Object Server. You need to create the secret in the same namespace in which you deploy the chart. | `minio`                                                   |
| `external.minio.mode`                                     | Sets Minio server mode.                                                                                                                                                                                                                               | `standalone`                                              |
| `external.minio.persistence.size`                         | Size of persistent volume claim (PVC).                                                                                                                                                                                                                | `100Gi`                                                   |
| `external.minio.replicas`                                 | Number of nodes (applicable only for Minio distributed mode). Must be 4 <= x <= 32.                                                                                                                                                                   | `4`                                                       |
| `global.datastores.minio.enabled`                         | Minio is part of the installation. If you install Minio yourself, you must provide the endpoint for your Minio installation.                                                                                                                  | `True`                                                            |
| `global.datastores.minio.endpoint`                        | Endpoint in form 'http://{mino-service}.{namespace}:{port}'.                                                                                                                                                                                          | `http://minio-ibm-minio-objectstore.speech-services:9000` |
| `global.datastores.minio.secretName`                      | Minio object storage access secret name created as an installation prerequisite.                                                                                                                                                                      | `minio`                                                   |
| `global.sttModels.enUsBroadbandModel.enabled`             | Whether to include the en-US Broadband Model in the installation.  | `True` |
| `global.sttModels.enUsNarrowbandModel.enabled`            | Whether to include the en-US Narrowband Model in the installation. | `True` |
| `global.sttModels.enUsShortFormNarrowbandModel.enabled`   | Whether to include the en-US ShortForm NarrowbandModel Model in the installation. | `True` |
| `global.sttModels.jaJpBroadbandModel.enabled`             | Whether to include the jp-JP Broadband Model in the installation. | `False` |
| `global.sttModels.jaJpNarrowbandModel.enabled`            | Whether to include the jp-JP Narrowband Model in the installation. | `False` |
| `global.sttModels.koKrBroadbandModel.enabled`             | Whether to include the ko-KR Broadband Model in the installation. | `False` |
| `global.sttModels.koKrNarrowbandModel.enabled`            | Whether to include the ko-KR Narrowband Model in the installation. | `False` |
| `global.sttModels.esEsBroadbandModel.enabled`             | Whether to include the es-ES Broadband Model in the installation. | `False` |
| `global.sttModels.esEsNarrowbandModel.enabled`            | Whether to include the es-ES Narrowband Model in the installation. | `False` |
| `global.sttModels.frFrBroadbandModel.enabled`             | Whether to include the fr-FR Broadband Model in the installation. | `False` |
| `global.sttModels.frFrNarrowbandModel.enabled`            | Whether to include the fr-FR Narrowband Model in the installation. | `False` |
| `postgressql.auth.authSecretName`                         | PostgresSQL name of the secrets object that contains the credentials to access the datastore. | `user-provided-postgressql` |

## Limitations

The product supports only the following:

-   The `x86` architecture
-   IBM Cloud Private for Data version `2.1.0`

## Integrate with other IBM Watson services

IBM Watson™ Speech Services is one of many IBM Watson services that are available on the IBM Cloud. Additional Watson services on the IBM Cloud allow you to bring Watson's AI platform to your business applications and to store, train, and manage your data in the most secure cloud.

For the full list of available Watson services, see the IBM Watson catalog on the public IBM Cloud at [https://cloud.ibm.com/catalog/](https://cloud.ibm.com/catalog/).

Watson services are currently organized into the following categories for different requirements and use cases:

-   **Assistant**: Integrate diverse conversation technology into your applications.
-   **Empathy**: Understand tone, personality, and emotional state.
-   **Knowledge**: Get insights through accelerated data optimization capabilities.
-   **Language**: Analyze text and extract metadata from unstructured content.
-   **Speech**: Convert text and speech with the ability to customize models.
-   **Vision**: Identify and tag content, and then analyze and extract detailed information that is found in images.


_Copyright© IBM Corporation 2018, 2019. All Rights Reserved._

