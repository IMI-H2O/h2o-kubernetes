# H2O-Kubernetes [![GitHub release](https://img.shields.io/github/v/release/imi-h2o/h2o-kubernetes)](https://github.com/IMI-H2O/h2o-kubernetes/releases/latest)
The Kubernetes stack of H2O extensions to the RADAR-base platform

## About
RADAR-base is an open-source platform designed to support remote clinical trials by collecting continuous data from wearables and mobile applications. [RADAR-Kubernetes](https://github.com/radar-base/radar-kubernetes) enables installing the RADAR-base platform onto Kubernetes clusters. This repository adds some components specific to IMI H2O to RADAR-base. RADAR-base platform can be used for wide range of use-cases. Depending on the use-case, the selection of applications need to be installed can vary. Please read the [component overview and breakdown](https://radar-base.atlassian.net/wiki/spaces/RAD/pages/2673967112/Component+overview+and+breakdown) to understand the role of each component and how components work together. 

H2O-Kubernetes setup uses [Helm 3](https://github.com/helm/helm) charts to package necessary Kubernetes resources for each component and [helmfile](https://github.com/roboll/helmfile) to modularize and deploy Helm charts of the platform on a Kubernetes cluster. This setup is designed to be a lightweight way to install and configure the RADAR-base components. The original images or charts may provide more and granular configurations. Please visit the `README` of respective charts in [h2o-helm-charts](https://github.com/IMI-H2O/h2o-helm-charts) to understand the configurations and visit the main repository for in depth knowledge.

### Disclaimer
This documentation assumes familiarity with all referenced Kubernetes concepts, utilities, and procedures and familiarity with Helm charts and helmfile. While this documentation will provide guidance for installing and configuring RADAR-base platform on a Kubernetes cluster, it is not a replacement for the official detailed documentation or tutorial of Kubernetes, Helm or Helmfile. If you are not familiar with these tools, we strongly recommend you to get familiar with these tools. Here is a [list of useful links](https://radar-base.atlassian.net/wiki/spaces/RAD/pages/2731638785/How+to+get+started+with+tools+around+RADAR-Kubernetes) to get started. 

## Prerequisites

### Local machine

The following tools should be installed in your local machine to install the RADAR-Kubernetes on your Kubernetes cluster.

| Component | Description |
|-----|------|
| [Git](https://git-scm.com/downloads) | RADAR-Kubernetes uses Git-submodules to use some third party Helm charts. Thus Git is required to properly download and sync correct versions of this repository and its dependent repositories |
| [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)| Kubernetes command-line tool, kubectl, allows you to run commands against Kubernetes clusters|
| [helm 3](https://github.com/helm/helm#install)| Helm Charts are used to package Kubernetes resources for each component|
| [helmfile](https://github.com/roboll/helmfile#installation)| RADAR-Kubernetes uses helmfiles to deploy Helm charts.|
| [helm-diff](https://github.com/databus23/helm-diff#install)| A dependency for Helmfile| 

**Once you have a working installation of a Kubernetes cluster, please [configure Kubectl with the appropriate Kubeconfig](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#verify-kubectl-configuration) to enable Kubectl to find and access your cluster. Then proceed to the installation section.** 

## Installation
> The following instructions on this guide are for local machines which runs on Linux operating systems. You can still use the same instructions with small to no changes on a MacOS device as well.

### Prepare
1. Clone the repository to your local machine by using following command.

    ```shell
    git clone https://github.com/IMI-H2O/h2o-kubernetes.git
    ```
 
2. Create basic config files using template files.

    ```shell
    cd h2o-kubernetes
    cp environments.yaml.tmpl environments.yaml
    cp etc/base.yaml etc/production.yaml
    cp etc/base.yaml.gotmpl etc/production.yaml.gotmpl
    ```
   
    It is recommended make a private clone of this repository, if you want to version control your configurations and/or share with other people.
      
    The best practise to share your platform configurations is by **sharing the encrypted version of `production.yaml`**.

### Configure

#### Project Structure
- [/bin](bin): Contains initialization scripts
- [/etc](etc): Contains configurations for some Helm charts.
- [/secrets](secrets): Contains secrets configuration for helm charts.
- [/helmfile.d](helmfile.d): Contains Helmfiles for modular deployment of the platform
- [environments.yaml](environments.yaml): Defines current environment files in order to be used by helmfile. Read more about `bases` [here](https://github.com/roboll/helmfile/blob/master/docs/writing-helmfile.md).
- `etc/production.yaml`: Production helmfile template to configure and install H2O components. Inspect the file to enable, disable and configure components required for your use case.
- `etc/production.yaml.gotmpl`: Change setup parameters that require Go templating, such as reading input files

1. Configure the [environments.yaml](environments.yaml) to use the files that you have created by copying the template files.
    ```shell
    vim environments.yaml # use the files you just created
    ```
2. Configure the `etc/production.yaml`. In this file you are required to fill in secrets and passwords used by RADAR-base applications. It is strongly recommended to use random password generator to fill these secrets. **You must keep this file secure and confidential once you have started installing the platform.**
    
    To create an encrypted password string and put it inside `kube_prometheus_stack.nginx_auth` variable. Please make sure you are using MD5 encryption. It appears `bcrypt` encryption isn't supported in current ingress-nginx.
    
    Optionally, you can also enable or disable other components that are configured otherwise by default.
  
    ```shell
    vim etc/production.yaml  # Change setup parameters and configurations
    ```
3. In `etc/production.yaml.gotmpl` file, change setup parameters that require Go templating, such as reading input files and selecting an option for the `keystore.p12`
    ```shell
    vim etc/production.yaml.gotmpl 
    ```
4. Run `bin/keystore-init` to create the Keystore file which used to sign JWT access tokens by [Management Portal](https://github.com/RADAR-base/radar-helm-charts/blob/main/charts/management-portal/README.md)
    ```shell
    ./bin/keystore-init
    ```
### Install
Once you are done with all configurations, H2O-Kubernetes can be deployed on a Kubernetes cluster. 

#### Install H2O-Kubernetes on your cluster. 

```shell
helmfile sync
```

The `helmfile sync` will synchronize all the Kubernetes resources defined in the helmfiles with your Kubernetes cluster.

#### Monitor and verify the installation process.

Once the installation is done or in progress, you can check the status using `kubectl get pods`.

If the installation has been successful, you should see an output with all pods marked as Running and READY.

Other ways to ensure that installation have been successful is to check application logs for errors and exceptions. 

#### Troubleshoot
If an application doesn't become fully ready installation will not be successful. In this case, you should investigate the root cause by investigating the relevant component. 

Some useful commands for troubleshooting a component are mentioned below.

1. Describe a pod to understand current status
```shell
kubectl describe pods <podname>
```
2. Investigate the logs of the pod
```shell
kubectl logs <podname>
```
To check last few lines
```shell
kubectl logs --tail 100 <podname>
```
To continue monitoring the logs
```shell
kubectl logs -f <podname>
```
For more information on how `kubectl` can be used to manage a Kubernetes application, please visit [Kubectl documentation](https://kubernetes.io/docs/reference/kubectl/cheatsheet/). 
 
If you have enabled monitoring you should also check **Prometheus** to see if there are any alerts. In next section there is a guide on how to connect to Prometheus.

## Update charts

To find any updates to the Helm charts that are listed in the repository, run
```
bin/chart-updates
```
