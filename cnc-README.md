![IBM Watson Compare and Comply logo](https://raw.githubusercontent.com/IBM-Bluemix-Docs/images/master/cnc-prod-banner.png)

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

## Chart details

This chart deploys a single IBM Watson Compare and Comply node with a default pattern.

It includes the following endpoints:

  - A health-check endpoint accessible on `/api/health/check`
  - An element-classification endpoint accessible on `/api/v1/element_classification`
  - An HTML-conversion endpoint accessible on `/api/v1/html_conversion`
  - A table-analyzer endpoint accessible on `/api/v1/tables`
  - A file-comparison endpoint accessible on `/api/v1/comparison`

## Prerequisites

  - IBM CloudPak for Data 2.1.0 or later
  - Kubernetes 1.11 or later
  - Helm 2.9.1 or later

  The installation of the chart needs to be performed from the command line, you must install and configure [helm](https:helm.sh) and [kubectl](https://docs-icpdata.mybluemix.net/docs/content/SSQNUZ_current/com.ibm.icpdata.doc/zen/install/kubectl-access.html) before proceeding.

### Creating custom certificate secret

The `ibm-watson-compare-comply-prod` chart enables you to specify a Kubernetes secrets file that contains a certificate and a key. You need to create the secret and specify the name of secret when you deploy this chart. The secrets file must contain the certificate and key in base64 encoding.

Encode the certificate and secret key in base64 encoding:

```bash
echo -n "{certificate_text}" | base64
YWRtaW4=

echo -n "{key_text}" | base64
YWRtaW4xMjM0
```

Create the secret as follows:

```yml
apiVersion: v1
kind: Secret
metadata:
  name: {custom_secret_name}
  namespace: {namespace}
type: Opaque
data:
  server.crt: YWRtaW4=
  server.key: YWRtaW4xMjM0
```

```bash
kubectl create -f secrets.yaml
```

For more information about creating Kubernetes secrets, see the [Kubernetes documentation](https://kubernetes.io/docs/concepts/configuration/secret/).

Install the Helm chart by running the following command:

```bash
helm install --set tls.cncTlsSecret={custom_secret_name} cnc-charts/stable/ibm-watson-compare-comply-prod --tls
```

You can also set your preferred name by running the following command:

```bash
helm install --name {my_release} --set tls.cncTlsSecret={custom_secret_name} cnc-charts/stable/ibm-watson-compare-comply-prod --tls
```

You can also provide a secret name during the configuration in IBM CloudPak for Data.
1) Check the box `Custom TLS certificate`.
2) Specify the name of the secret that you created previously in `Secret resource name`.

When you install the chart, you must use the same release name that you used to generate the certificate.

## Installing the chart

Installing the Helm chart deploys a single IBM Watson Compare and Comply node with a default pattern into an IBM Cloud Private environment.

To install the chart with the release name `{my_release}`, run the following command.

```bash
helm install --name {my_release} stable/ibm-watson-compare-comply-prod --set global.icpDockerRepo="{ICP_cluster_host}:8500/{namespace}"
```

After the command runs, it prints the current status of the release. You can also access the admin console through the IBM CloudPak for Data UI:

 1. From Menu navigate to **Workloads -> Deployments**.
 
 1. Click the deployment. The default from above is `my_release`.

If you need to uninstall (delete) the `{my_release}` deployment, run the following command:

```bash
helm delete {my_release} --purge
```

## Resources Required

  - Minimum CPU - 2200m
  - Minimum Memory - 10.25Gi

## Configuration

The following tables lists the configurable parameters of the IBM Watson Compare and Comply chart and their default values.

|         Parameter        |                       Description                       |                         Default                          |
|--------------------------|---------------------------------------------------------|-------------------------------------------------------------------|
| `workerSize`             | Size of the Worker processing a contract document       | `2Cores 10G 1 concurrent document (2 VPC)` |

## Limitations

IBM Watson Compare and Comply can currently run only on Intel architecture nodes.

## PodSecurityPolicy requirements

The predefined PodSecurityPolicy name [`ibm-anyuid-psp`](https://ibm.biz/cpkspec-psp) has been verified for this chart. If your target namespace is bound to this PodSecurityPolicy, you can proceed to install the chart.

## Integrate with other IBM Watson services

Compare and Comply is one of many IBM Watson services. Additional Watson services on the IBM Cloud and IBM CloudPak for Data enable you to bring Watson's AI platform to your business application, and to store, train, and manage your data in the most secure cloud.

For the full list of available Watson services, see the IBM Watson catalog on the public IBM Cloud at [https://cloud.ibm.com/catalog/](https://cloud.ibm.com/catalog?category=ai).

Watson services are currently organized into the following categories for different requirements and use cases:

  - **Assistant**: Integrate diverse conversation technology into your application
  - **Empathy**: Understand tone, personality, and emotional state
  - **Knowledge**: Get insights through accelerated data optimization capabilities
  - **Language**: Analyze text and extract metadata from unstructured content
  - **Speech**: Convert text and speech with the ability to customize models
  - **Vision**: Identify and tag content then analyze and extract detailed information found in images


_Copyright© IBM Corporation 2018, 2019. All Rights Reserved._
