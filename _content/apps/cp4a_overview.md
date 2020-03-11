---
title:  Introduction
weight: 100
---

### Table of Contents
* [Prerequisites](../cp4a-prereq/)
* [Installing Cloud Pak for Applications](../cp4a_installation/)
* [Install Cloud Pak for Applications (Mac OSx) Developer Tools](../cp4a_install_dev_tools_mac/)
* [Install Cloud Pak for Applications (Windows) Developer Tools](../cp4a_install_dev_tools_win/)
* [Modernize existing applications](../cp4a_use_case_app_mod/)
* [Install Mobile Foundation](../cp4a_install_mobile_foundation/)
* [Building new applications](../cp4a_use_case_cloud_native/)
* [Learn more: Next steps](../cp4a_learn_more/)


## Introduction

The Cloud Pak for Applications provides product offerings to support modernizing existing applications and building new cloud native applications.
The applications run within a Kubernetes cluster provided with the Red Hat OpenShift Container Platform.
The focus provided here is on running application workloads as containers.
The Cloud Pak for Applications is a bundle of multiple offerings.
This diagram provides an overview of what offerings are included and what they would be used for.

![Overview](https://www.ibm.com/support/knowledgecenter/SSCSJL/images/icpa_overview.png)

The key offerings reviewed in this installation and usage scenarios are:

| Offering | Installation Steps | Description |
| -------- | ------------------ | ----------- |
| Red Hat OpenShift Container Platform | [Install](../../ocp/introduction) | Kubernetes platform required for running application workloads |
| IBM Kabanero Enterprise | [Install](../cp4a_installation) | Open source projects to build, deploy and run applications.  Installs into an OpenShift Container Platform cluster. |
| Developer Tools | [MacOS](../cp4a_install_dev_tools_mac) / [Windows](../cp4a_install_dev_tools_win) | Tools needed for a developer to build, test and debug applications.
| Red Hat Runtimes | [Install](https://www.ibm.com/support/knowledgecenter/SSCSJL/install-icpa.html) | Application runtimes and framework for JBoss, Vert.x and Node. |

These offerings are also included in the Cloud Pak for Applications but not focus within this material.  These offerings support running existing applications but not focused on the container platform.

| Offering | Installation Steps | Description |
| -------- | ------------------ | ----------- |
| IBM WebSphere Application Server Network Deployment | [Install](https://www.ibm.com/support/knowledgecenter/SSCSJL/install-icpa.html) | Continue to run existing WebSphere apps. |
| IBM WebSphere Application Server | [Install](https://www.ibm.com/support/knowledgecenter/SSCSJL/install-icpa.html) | Continue to run existing WebSphere apps. |
| IBM WebSphere Application Server Liberty Core | [Install](https://www.ibm.com/support/knowledgecenter/SSCSJL/install-icpa.html) | Continue to run existing Liberty apps.
| IBM Mobile Foundation | [Install](https://www.ibm.com/support/knowledgecenter/SSCSJL/install-icpa.html) | Run existing mobile apps. |
| IBM Cloud Private |[Install](https://www.ibm.com/support/knowledgecenter/SSCSJL/install-icpa.html) | Migrate existing workloads to OpenShift Container Platform. |

## Installation Overview

The primary method for installing the Cloud Pak for Applications follows the key high level steps of:
- **Install Red Hat OpenShift Container Platform** -  Cloud Pak provides OpenShift Container Platform to create a new cluster.  You can also use any existing OpenShift 3.11 to install the Cloud Pak into.
- **Install Cloud Pak for Applications**  - The Cloud Pak is installed into the cluster and provides Transformation Advisor and IBM Kabanero Enterprise.  The installation provides IBM runtimes for Liberty, Microprofile, and Spring, as well as several open source projects which include Appsody, Tekton, and Knative.
- **Install developer tools** - A developer is provided tools for an IDE and key CLIs to access the cluster.  Instructions are available for both Mac OSx and Windows.
