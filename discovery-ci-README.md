![](https://raw.githubusercontent.com/ibm-cloud-docs/images/master/discovery-icp4d-new-banner.png)

# IBM Watson Discovery for Content Intelligence

## Introduction
IBM Watson™ Discovery for Content Intelligence is an optional component for Discovery that applies AI to understand the structure and content of contracts, invoices, and purchase orders.

## Prerequisites

  - IBM® Cloud Pak for Data 2.5.0.0
  - IBM® Watson Discovery PPA

Before installing Watson Discovery, __you must install and configure [IBM CP4D](https://www.ibm.com/support/knowledgecenter/SSQNUZ_current/cpd/overview/relnotes-2.5.0.0.html)__.

**Note: The Content Intelligence component is to be installed along with the Watson Discovery installation.**


## Installation Instructions

Steps to install Watson Discovery is available [here](https://github.com/ibm-cloud-docs/data-readmes/blob/master/discovery-README.md).

For installing `Discovery for Content Intelligence` component follow the steps mentioned in the above Discovery Installation link.

1. Use the `ci-override.yaml` file present in the IBM Passport Advantage® file for Content Intelligence as the `override.yaml` file during Discovery Installation.
    If you already have an override.yaml file add the key and values available in the `ci-override.yaml` to the file.
    Run the deploy script with -O <override-file.yaml> while Watson Discovery installation.

    ```
     ./deploy.sh -d path/to/ibm-watson-discovery -w <tiller-namespace> -O <override-file>.yaml  -e <release-name-prefix>
    ```

2. After `Watson Discovery for Content Intelligence` is installed, perform the following steps to add `Software Identfication Tags` to the required pod:
   Note: `Discovery for Content Intelligence PPA` contains the Tag file ending with `.swidtag`
   ```
   oc cp ibm-discovery-content-intelligence-ppa/ibm.com_IBM_Watson_Discovery_for_Cloud_Pak_for_Data_Content_Intelligence-2.1.0.swidtag "$(oc get pod | awk '{print $1}' | grep gateway-0)":/swidtag/
   ```


_Copyright©  IBM Corporation 2019. All Rights Reserved._
