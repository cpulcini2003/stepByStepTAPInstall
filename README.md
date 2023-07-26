# TAP 1.4 Step-by-step Installation on vSphere with Tanzu

This procedure allows you to deploy TAP from command line on a basic Workload Cluster deployend on vSphere with Tanzu version 7.0.3. Kubenernetes version is 1.23.
An embedded harbor registry has benn configured.
This proceedure should work alkso with vSphere with Tanzu v8. But it's not been tested jet.
Essential Pack should come installed with 1.23. If needed procede with installation
Tanzu cli must be installed on laptop or server.  

## Login to the cluster and check status

Login to the Workload cluster:
```bash
export SUPERVISOR_IP=10.x.x.242

export KUBECTL_VSPHERE_PASSWORD=********

kubectl vsphere login --server=${SUPERVISOR_IP} --vsphere-username administrator@vsphere.local --tanzu-kubernetes-cluster-name tap-cluster --tanzu-kubernetes-cluster-namespace test-env --insecure-skip-tls-verify
```

Verify Carvel tool to be present in the Workload Cluster:
```bash
tanzu package installed list
  NAME  PACKAGE-NAME  PACKAGE-VERSION  STATUS
```
If previous command do not returns an error but just empty answer it means you can proceed.


Create clusterrolebinding as specified below:
```bash
kubectl create clusterrolebinding default-tkg-admin-privileged-binding --clusterrole=psp:vmware-system-privileged --group=system:authenticated 
```
Failing to run this command will deny application PODs to be created.


# Installation

Create namespace to receive packages:
```bash
kubectl create ns tap-install
```

## Create registries secrets

Create secret for Tanzu Registry specifying credentials, registry path:
```bash
export INSTALL_REGISTRY_HOSTNAME=registry.tanzu.vmware.com
export INSTALL_REGISTRY_USERNAME=accountname@vmware.com
export INSTALL_REGISTRY_PASSWORD=********

tanzu secret registry add tap-registry \
  --username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} \
  --server ${INSTALL_REGISTRY_HOSTNAME} \
  --export-to-all-namespaces --yes --namespace tap-install
```

Create secret for TBS for local Harbor registry:
```bash
export MY_REGISTRY_HOSTNAME=10.*.*.244
export MY_REGISTRY_USERNAME=administrator@vsphere.local
export MY_REGISTRY_PASSWORD=********

tanzu secret registry add tbs-registry-credentials --username ${MY_REGISTRY_USERNAME} --password ${MY_REGISTRY_PASSWORD} --server ${MY_REGISTRY_HOSTNAME} --export-to-all-namespaces --yes --namespace tap-install
```


## Download TAP and TBS packages

Set envs:
```bash
export INSTALL_REPO=tanzu-application-platform
export TAP_VERSION=1.4.0
export TBS_VERSION=1.9.0
```

Download TAP Packages:
```bash
tanzu package repository add tanzu-tap-repository \
  --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tap-packages:$TAP_VERSION \
  --namespace tap-install
```

Check installation status:
```bash
tanzu package repository get tanzu-tap-repository --namespace tap-install
```

Download TBS Packages:
```bash
tanzu package repository add tbs-full-deps-repository --url ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/full-tbs-deps-package-repo:${TBS_VERSION} --namespace tap-install
```

Check installation status:
```bash
tanzu package repository get tbs-full-deps-repository --namespace tap-install
```

## Prepare tap value yaml file

A sample tap-value.yaml file for a single cluster TAP implementation is provided in the currenr git repo.

https://github.com/cpulcini2003/stepByStepTAPInstall

Please modify accordingly with your configuration and proceed with next steps


## Install TAP & TBS Packages

In order to install all TAP packages components run:
```bash
tanzu package install tap -n tap-install -p tap.tanzu.vmware.com -v ${TAP_VERSION} --values-file tap-value.yaml 
```

In case of changes on configuration file after previous installation run:
```bash
tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tap-value.yaml -n tap-install
```

In order to install all TBS packages components run:
```bash
tanzu package install full-tbs-deps -p full-tbs-deps.tanzu.vmware.com -v ${TBS_VERSION} -n tap-install --poll-timeout 30m 
```


Check reconciliation status:
```bash
tanzu package installed list -n tap-install
```

# Set DNS wildcard

Retrieve envoy ingress external IP:
```bash
kubectl get service envoy -n tanzu-system-ingress

NAME    TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
envoy   LoadBalancer   100.70.189.170   10.220.36.4   80:30293/TCP,443:32602/TCP   8m16s 
```

and set you DNS with a wildcard to your domain

*.h2o-4-7299.h2o.vmware.com >> 10.220.36.4



# Connect to TAP GUI

Retrieve TAP full domain path:
```bash
k get httpproxy -A | grep tap-gui

tap-gui          tap-gui                  tap-gui.h2o-4-7299.h2o.vmware.com                   tap-gui-cert   valid    Valid HTTPProxy
```

Open browser at: 
tap-gui.h2o-4-7299.h2o.vmware.com


# Create Developer namespace

Create Namespace
```bash
k create ns cpu
```

# Add label to let the system creare secret, sa and rolebindings

Create label and check
```bash
k label namespace cpu apps.tanzu.vmware.com/tap-ns=""
k get secret,serviceaccount,rolebinding,pods,workload,configmap -n cpu


```

