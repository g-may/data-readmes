![IBM Watson Compare and Comply logo](https://raw.githubusercontent.com/ibm-cloud-docs/images/master/cnc-prod-banner.png)

# IBM Watson Compare and Comply

## Introduction

IBM Watson&trade; Compare and Comply enables understanding of governing business documents with pre-trained models so enterprises can get started in minutes. The document conversion (programmatic and scanned PDF, TIFF, JPEG, Word) capabilities enable both machine-to-machine and machine-to-human readable formats. The table understanding, element classification, and comparison capabilities of Compare and Comply enable automation of complex business processes such as contract review and negotiation, invoice reconciliation, software entitlement verification, and more. Such automation of processes result in increased productivity, minimization of costs, and reduced exposure.

Compare and Comply provides:

  - Natural language understanding of contracts and other governing documents
  - Conversion of PDFs, images (PNG, TIFF, JPEG), and Word into HTML
  - Identification of parties in the contracts and the obligations and rights assigned to each
  - Automatic labeling of sentences in contracts with categories such as termination, privacy, payment terms, and more
  - A Compare API that analyzes two contracts, side-by-side, and highlights similarities and differences at the level of individual clauses
  - Table extraction, which parses each cell in the table and associates metadata such as row and column headers

## Chart Details

This chart deploys a single IBM Watson Compare and Comply node with a default pattern.

It includes the following endpoints:

  - A health-check endpoint accessible on `/api/health/check`
  - An element-classification endpoint accessible on `/api/v1/element_classification`
  - An HTML-conversion endpoint accessible on `/api/v1/html_conversion`
  - A table-analyzer endpoint accessible on `/api/v1/tables`
  - A file-comparison endpoint accessible on `/api/v1/comparison`

## Resources Required

  - Minimum CPU: 2200m
  - Minimum memory: 10.25Gi

## Installing the chart

Perform the following steps to install of the Compare and Comply chart on IBM Cloud Pak for Data.

### Prerequisites
   
  - IBM Cloud Pak for Data 2.1.0.1 or later OR Red Hat OpenShift 3.11 or later
  - Kubernetes 1.12 or later
  - Tiller 2.9.1 or later

Required command-line tools include:

  - [cloudctl](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/manage_cluster/install_cli.html)
  - [kubectl](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/manage_cluster/install_kubectl.html)
  - [helm](https://www.ibm.com/support/knowledgecenter/SSBS6K_3.1.2/app_center/create_helm_cli.html)
  - [docker](https://www.docker.com/)

### Documentation

For information on installing or upgrading to IBM Cloud Pak for Data, See [Installing Cloud Pak for Data](https://docs-icpdata.mybluemix.net/docs/content/SSQNUZ_current/com.ibm.icpdata.doc/zen/install/ovu.html).


### Preparing to install the chart

Perform the following steps to prepare to install the chart.

1. Purchase and download the Compare and Comply archive file in `.tgz` format from [IBM Passport Advantage](https://www.ibm.com/software/passportadvantage/).

1. Use secure copy (`scp`) to copy the archive file to your IBM Cloud Pak for Data master node:
   ```bash
   scp {local_file_location} root@{master_node}:{filename}
   ```

1. Login to the master node of your IBM Cloud Pak for Data master node as root:
   ```bash
   ssh root@{master_node}
   ```
    
1. Untar the archive file:
   ```bash
   tar -xvfz ibm-watson-compare-comply-prod-1.1.5.tgz -C {path/filename}
   ```
    
1. Log in to `cloudctl`:
   ```bash
   cloudctl login -a https://{hostname}:8443 --skip-ssl-validation
   ```
   **Notes**:
    - When you log into `cloudctl`, the system prompts you for a target namespace. Select the `default` namespace.
    - By default, `{hostname}` is `mycluster.icp`.

1. Login to `docker`:
   ```bash
   docker login -u admin -p {foundation_admin_password} {hostname}:8500
   ```

1. Load the archive:
   ```bash
   cloudctl catalog load-archive --archive {filename}
   ```

### Running pre-installation scripts

Run the pre-installation scripts:

  1. On each cluster, run the `labelNamepsace.sh` script once:
     ```bash
     ./ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration/labelNamespace.sh {CP4D_namespace}
     ```
    where `{CP4D_namespace}` is the namespace where Cloud Pak for Data is installed; the standard value is `zen`. The specified namespace **must** have a label for the `NetworkPolicy` to work correctly. Only `nginx` and `zen` pods can to communicate with the pods in the namespace where the chart is installed.

  1. Before each installation, run the `deleteInstances.sh` script:
    ```bash
    ./ibm_cloud_pak/pak_extensions/pre-install/clusterAdministration/deleteInstances.sh {CP4D_namespace}
    ```
    where `{CP4D_namespace}` is the namespace where Cloud Pak for Data is installed; the standard value is `zen`. The script cleans up any instances that were deleted as part of a previous installation.

### Installing the chart

Install the Helm chart:

  1. Run the `helm install` command:
     ```bash
      helm install {path_to_untarred_archive}/ibm-watson-compare-comply-prod-1.1.6.tgz --name {release_name} --tls --namespace {namespace}
    ```
   **Note**: By default, `{namespace}` is `default`.

### Creating a Compare and Comply instance

1. After the Helm installation completes, log in to the IBM Cloud Pak for Data UI at `https://{hostname}:31843/`.

1. Click the **Add-on** icon at the upper right corner, then locate and click the **Watson Comply and Comply** tile.

1. Click **Provision Instance**.

1. When the instance is ready, click **Manage Instances**.

1. Click the **...** icon on the right, then click **View Details**. The UI displays the URL endpoint and access token for the instance.

Proceed as described in the [Compare and Comply documentation](https://cloud.ibm.com/docs/services/compare-comply-data?topic=compare-comply-data-getting-started).


## Configuration

The following table lists the configurable parameters of the IBM Watson Compare and Comply chart and their default values.

|         Parameter        |                       Description                       |                         Default                          |
|--------------------------|---------------------------------------------------------|-------------------------------------------------------------------|
| `workerSize`             | Size of the Worker processing a contract document       | `2Cores 10G 1 concurrent document (2 VPC)` |

## Limitations

IBM Watson Compare and Comply can currently run only on Intel architecture nodes.

## PodSecurityPolicy Requirements

The predefined PodSecurityPolicy name [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp) has been verified for this chart. If your target namespace is bound to this PodSecurityPolicy, you can proceed to install the chart.

Custom PodSecurityPolicy definition:
```yml
---
```

Similarly, the chart has been verified with the PodSecurityPolicy name [`ibm-restricted-scc`](https://ibm.biz/cpkspec-scc) on Red Hat OpenShift. 

# Integrate with other IBM Watson services

Compare and Comply is one of many IBM Watson services. Additional Watson services on the public IBM Cloud and IBM Cloud Pak for Data allow you to bring Watson's AI platform to your business application, and to store, train, and manage your data in the most flexible hybrid cloud.

For the full list of available Watson services, see the IBM Watson catalog on the public IBM Cloud at [https://cloud.ibm.com/catalog/](https://cloud.ibm.com/catalog/?category=ai).

Watson services are currently organized into the following categories for different requirements and use cases:

  - **Assistant**: Integrate diverse conversation technology into your application
  - **Empathy**: Understand tone, personality, and emotional state
  - **Knowledge**: Get insights through accelerated data optimization capabilities
  - **Language**: Analyze text and extract metadata from unstructured content
  - **Speech**: Convert text and speech with the ability to customize models
  - **Vision**: Identify and tag content then analyze and extract detailed information found in images


_Copyright© IBM Corporation 2018, 2019. All Rights Reserved._
