+++
title = "Install Kubeflow"
description = "Instructions for deploying Kubeflow with the shell"
weight = 4
+++

This guide describes how to use the kfctl binary to
deploy Kubeflow on IBM Cloud.

## Prerequisites

### Installing the IBM Cloud developer tools

> If you already have `ibmcloud` installed with the latest `ibmcloud ks` (Kubernetes Service) plug-in, you
> can skip these steps.

1.  Download and install the `ibmcloud` command line tool:
    https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started#overview

1.  Install the [Kubernetes Service plug-in](https://cloud.ibm.com/docs/cli?topic=containers-cli-plugin-kubernetes-service-cli):
    ```
    ibmcloud plugin install container-service
    ```
1.  Authorize `ibmcloud`:
    ```
    ibmcloud login
    ```

### Setting environment variables

To simplify the command lines for this walkthrough, you need to define a few
environment variables.

1.  Set `CLUSTER_NAME` and `CLUSTER_ZONE` variables:

    ```
    export CLUSTER_NAME=kubeflow
    export CLUSTER_ZONE=dal13
    ```

    - `CLUSTER_NAME` must be lowercase and unique among any other Kubernetes
      clusters in the specified CLUSTER_ZONE.
    - `CLUSTER_ZONE` identifies the location where CLUSTER_NAME will be created. Run `ibmcloud ks locations` to list supported IBM Cloud Kubernetes Service locations. For example, choose `dal13` to create CLUSTER_NAME in the Dallas (US) data center. 

### Creating a IBM Cloud Kubernetes cluster

To make sure the cluster is large enough to host all the Knative and Istio
components, the recommended configuration for a cluster is:

- Kubernetes version 1.15
- 4 vCPU nodes with 16GB memory (`b2c.4x16`)

1.  Create a Kubernetes cluster on IKS with the required specifications:

    ```
    ibmcloud ks cluster create classic \
      --flavor b2c.4x16 \
      --name $CLUSTER_NAME \
      --zone=$CLUSTER_ZONE \
      --workers=3
    ```

    If you're starting in a fresh account with no public and private VLANs, they
    are created automatically for you. If you already have VLANs configured in
    your account, get them via `ibmcloud ks vlans --zone $CLUSTER_ZONE` and
    include the public/private VLAN id in the `cluster create` command:

    ```
    ibmcloud ks cluster create classic \
      --machine-type=b2c.4x16 \
      --name=$CLUSTER_NAME \
      --zone=$CLUSTER_ZONE \
      --workers=3 \
      --private-vlan $PRIVATE_VLAN_ID \
      --public-vlan $PUBLIC_VLAN_ID 
    ```

1.  Wait until your Kubernetes cluster is deployed:

    ```
    ibmcloud ks clusters | grep $CLUSTER_NAME
    ```

    It can take a while for your cluster to be deployed. Repeat the above
    command until the state of your cluster is "normal".


1.  Point `kubectl` to the cluster:

    ```
    ibmcloud ks cluster config --cluster $CLUSTER_NAME
    ```

1.  Make sure all nodes are up:

    ```
    kubectl get nodes
    ```

    Make sure all the nodes are in `Ready` state. You are now ready to install
    Istio into your cluster.
    
## IBM Cloud Block Storage Setup
By default, IBM Cloud Kubernetes cluster uses [IBM Cloud File Storage](https://www.ibm.com/cloud/file-storage) based on NFS as the default storage class. File Storage is designed to run RWX (read-write multiple nodes) workloads with proper security built around it. Therefore, File Storage [does not allow `fsGroup` securityContext](https://cloud.ibm.com/docs/containers?topic=containers-security#container) which is needed for DEX and Kubeflow Jupyter Server.

[IBM Cloud Block Storage](https://www.ibm.com/cloud/block-storage) provides a fast way to store data and
satisfy many of the Kubeflow persistent volume requirements such as `fsGroup` out of the box and optimized RWO (read-write single node) which is used on all Kubeflow's persistent volume claim. 

Therefore, we strongly recommend to set up [IBM Cloud Block Storage](https://cloud.ibm.com/docs/containers?topic=containers-block_storage#add_block) as the default storage class so that you can
get the best experience from Kubeflow.

1. [Follow the instructions](https://helm.sh/docs/intro/install/) to install the Helm version 3 client on your local machine.

1. Add the IBM Cloud Helm chart repository to the cluster where you want to use the IBM Cloud Block Storage plug-in.
    ```shell
    helm repo add iks-charts https://icr.io/helm/iks-charts
    helm repo update
    ```

1. Install the IBM Cloud Block Storage plug-in. When you install the plug-in, pre-defined block storage classes are added to your cluster.
    ```shell
    helm install 1.6.0 iks-charts/ibmcloud-block-storage-plugin -n kube-system
    ```
    
    Example output:
    ```
    NAME: 1.6.0
    LAST DEPLOYED: Thu Feb 27 11:41:35 2020
    NAMESPACE: kube-system
    STATUS: deployed
    REVISION: 1
    NOTES:
    Thank you for installing: ibmcloud-block-storage-plugin.   Your release is named: 1.6.0
    ...
    ```

1. Verify that the installation was successful.
    ```shell
    kubectl get pod -n kube-system | grep block
    ```
    
1. Verify that the storage classes for Block Storage were added to your cluster.
    ```
    kubectl get storageclasses | grep block
    ```

1. Set the Block Storage as the default storageclass.
    ```shell
    kubectl patch storageclass ibmc-block-gold -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    kubectl patch storageclass ibmc-file-bronze -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
    
    # Check the default storageclass is block storage
    kubectl get storageclass | grep \(default\)
    ```
    
    Example output:
    ```
    ibmc-block-gold (default)   ibm.io/ibmc-block   65s
    ```

## Understanding the Kubeflow deployment process

The deployment process is controlled by the following commands:

* **build** - (Optional) Creates configuration files defining the various
  resources in your deployment. You only need to run `kfctl build` if you want
  to edit the resources before running `kfctl apply`.
* **apply** - Creates or updates the resources.
* **delete** - Deletes the resources.

### App layout

Your Kubeflow application directory **${KF_DIR}** contains the following files and 
directories:

* **${CONFIG_FILE}** is a YAML file that defines configurations related to your 
  Kubeflow deployment.

  * This file is a copy of the GitHub-based configuration YAML file that
    you used when deploying Kubeflow. For example, {{% config-uri-k8s-istio %}}.
  * When you run `kfctl apply` or `kfctl build`, kfctl creates
    a local version of the configuration file, `${CONFIG_FILE}`,
    which you can further customize if necessary.

* **kustomize** is a directory that contains the kustomize packages for Kubeflow applications.
    * The directory is created when you run `kfctl build` or `kfctl apply`.
    * You can customize the Kubernetes resources (modify the manifests and run `kfctl apply` again).


## Kubeflow installation
Run the following commands to set up and deploy Kubeflow.

1. Download the kfctl {{% kf-latest-version %}} release from the
  [Kubeflow releases 
  page](https://github.com/kubeflow/kfctl/releases/tag/{{% kf-latest-version %}}).

1. Unpack the tar ball
      ```
      tar -xvf kfctl_{{% kf-latest-version %}}_<platform>.tar.gz
      ```

1. Run the following commands to set up and deploy Kubeflow. The code below includes an optional command to add the binary kfctl to your path. If you don’t add the binary to your path, you must use the full path to the kfctl binary each time you run it.

    ```
    # The following command is optional, to make kfctl binary easier to use.
    export PATH=$PATH:<path to where kfctl was unpacked>

    # Set KF_NAME to the name of your Kubeflow deployment. This also becomes the
    # name of the directory containing your configuration.
    # For example, your deployment name can be 'my-kubeflow' or 'kf-test'.
    export KF_NAME=<your choice of name for the Kubeflow deployment>

    # Set the path to the base directory where you want to store one or more 
    # Kubeflow deployments. For example, /opt/.
    # Then set the Kubeflow application directory for this deployment.
    export BASE_DIR=<path to a base directory>
    export KF_DIR=${BASE_DIR}/${KF_NAME}

    # Set the configuration file to use, such as the file specified below:
    export CONFIG_URI="https://raw.githubusercontent.com/kubeflow/manifests/master/kfdef/kfctl_ibm.yaml"

    # Generate and deploy Kubeflow:
    mkdir -p ${KF_DIR}
    cd ${KF_DIR}
    kfctl apply -V -f ${CONFIG_URI}
    ```

    * **${KF_NAME}** - The name of your Kubeflow deployment.
      If you want a custom deployment name, specify that name here.
      For example,  `my-kubeflow` or `kf-test`.
      The value of KF_NAME must consist of lower case alphanumeric characters or
      '-', and must start and end with an alphanumeric character.
      The value of this variable cannot be greater than 25 characters. It must
      contain just a name, not a directory path.
      This value also becomes the name of the directory where your Kubeflow 
      configurations are stored, that is, the Kubeflow application directory. 

    * **${KF_DIR}** - The full path to your Kubeflow application directory.

1. Check the resources deployed correctly in namespace `kubeflow`

     ```
     kubectl get all -n kubeflow
     ```

1. Open Kubeflow Dashboard. The default installation does not create an external endpoint but you can use port-forwarding to visit your cluster. Run the following command and visit http://localhost:8080.

     ```
     kubectl port-forward svc/istio-ingressgateway -n istio-system 8080:80
     ```

In case you want to expose the Kubeflow Dashboard over an external IP, you can change the type of the ingress gateway. To do that, you can edit the service:

     kubectl edit -n istio-system svc/istio-ingressgateway

From that file, replace `type: NodePort` with `type: LoadBalancer` and save.

While the change is being applied, you can watch the service until below command prints a value under the `EXTERNAL-IP` column:

     kubectl get -w -n istio-system svc/istio-ingressgateway

The external IP should be accessible by visiting http://<EXTERNAL-IP>. Note that above installation instructions do not create any protection for the external endpoint so it will be accessible to anyone without any authentication. 

## Additional information

You can find general information about Kubeflow configuration in the guide to [configuring Kubeflow with kfctl and kustomize](/docs/other-guides/kustomize/).
