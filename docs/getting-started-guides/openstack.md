---
---

* TOC
{:toc}

## Getting started with OpenStack

This guide will take you through the steps of deploying Kubernetes to Openstack using `kube-up.sh`. The primary mechanisms for this are [OpenStack Heat](https://wiki.openstack.org/wiki/Heat) and the [SaltStack](https://github.com/kubernetes/kubernetes/tree/master/cluster/saltbase) distributed with Kubernetes.

The default OS is CentOS 7, this has not been tested on other operating systems.

This guide assumes you have a working OpenStack cluster.

## Pre-Requisites
If you already have the OpenStack CLI tools installed and configured, you can move on to the [Starting a cluster](#starting-a-cluster) section.

#### Install OpenStack CLI tools

```sh
sudo pip install -U --force 'python-heatclient>=0.9.0'
sudo pip install -U --force 'python-swiftclient>=2.7.0'
sudo pip install -U --force 'python-glanceclient>=1.2.0'
sudo pip install -U --force 'python-novaclient>=3.2.0'
```

#### Configure Openstack CLI tools

Please talk to your local OpenStack administrator for an `openrc.sh` file.

Once that file is sourced into your environment, this provider will consume the [correct variables](http://docs.openstack.org/user-guide/common/cli_set_environment_variables_using_openstack_rc.html) to talk to OpenStack and turn-up the Kubernetes cluster.

Otherwise, you must set the following appropriately:

```sh
export OS_USERNAME=username
export OS_PASSWORD=password
export OS_TENANT_NAME=projectName
export OS_AUTH_URL=https://identityHost:portNumber/v2.0
export OS_TENANT_ID=tenantIDString
export OS_REGION_NAME=regionName
```

#### Set additional configuration values

In addition, here are some commonly changed variables specific to this provider, with example values. Under most circumstances you will not have to change these.

```sh
export STACK_NAME=KubernetesStack
export NUMBER_OF_MINIONS=3
export MAX_NUMBER_OF_MINIONS=3
export MASTER_FLAVOR=m1.small
export MINION_FLAVOR=m1.small
export EXTERNAL_NETWORK=public
export DNS_SERVER=8.8.8.8
export IMAGE_URL_PATH=http://cloud.centos.org/centos/7/images
export IMAGE_FILE=CentOS-7-x86_64-GenericCloud-1510.qcow2
export SWIFT_SERVER_URL=http://192.168.123.100:8080
export ENABLE_PROXY=false
```
#### Manually overriding configuration values

If you do not have your environment variables set, or do not want them consumed, modify the variables in the following files under `cluster/openstack`:

- **config-default.sh** Sets all parameters needed for heat template.
- **config-image.sh** Sets parameters needed to download and create new OpenStack image via glance.
- **openrc-default.sh** Sets environment variables for communicating to OpenStack. These are consumed by the cli tools (heat, glance, swift, nova).
- **openrc-swift.sh** Some OpenStack setups require the use of seperate swift credentials. Put those credentials in this file.

Please see the contents of these files for documentation regarding each variable's function.

## Starting a cluster

Once you've installed the OpenStack CLI tools and have set your OpenStack environment variables, issue this command:

```sh
export KUBERNETES_PROVIDER=openstack; curl -sS https://get.k8s.io | bash
```
Alternatively, you can download [Kubernetes release](https://github.com/kubernetes/kubernetes/releases) and extract the archive. To start your cluster, open a shell and run:

```sh
cd kubernetes # Or whichever path you have extracted the release to
KUBERNETES_PROVIDER=openstack ./cluster/kube-up.sh
```
Or, if you are working from a checkout of the Kubernetes code base, and want to build/test from source:

```sh
cd kubernetes # Or whatever your checkout root directory is called
make clean
make quick-release
KUBERNETES_PROVIDER=openstack ./cluster/kube-up.sh
```
## Inspect your cluster

Once kube-up is finished, your cluster should be running:

```console
$ ./cluster/kubectl.sh get cs
```

You can list the nodes in your cluster:

```console
$ ./cluster/kubectl.sh get nodes
NAME                            LABELS                                                 STATUS    AGE
kubernetesstack-node-tc9f2tfr   kubernetes.io/hostname=kubernetesstack-node-tc9f2tfr   Ready     21h
```
Being a new cluster, there will be no pods, services or replication controllers.

```console
$ ./cluster/kubectl.sh get pods
NAME        READY     STATUS    RESTARTS   AGE

$ ./cluster/kubectl.sh get services
NAME              CLUSTER_IP       EXTERNAL_IP       PORT(S)       SELECTOR               AGE

$ ./cluster/kubectl.sh get replicationcontrollers
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR   REPLICAS
```

You are now ready to create Kubernetes objects.

## Using your cluster

```
kubectl run nginx --image=nginx --generator=run-pod/v1
kubectl port-forward nginx 8888:80
```

You should now see nginx on http://localhost:8888

For more complex examples please see the [examples directory](https://github.com/kubernetes/kubernetes/tree/{{page.githubbranch}}/examples/)

## Administering your cluster with Openstack

You can manage the nodes in your cluster using the OpenStack CLI Tools.

First, set your environment variables:

```
cd kubernetes
. cluster/openstack/config-default.sh
. cluster/openstack/openrc-default.sh
```

To get all information about your cluster, use heat:

```
heat stack-show $STACK_NAME
```

To see a list of nodes, use nova:

```
nova list --name=$STACK_NAME
```

See the [OpenStack CLI Reference](http://docs.openstack.org/cli-reference/) for more details.

## SSHing to your nodes

Your public key was added during the cluster turn-up, so you can easily ssh to them.

```
$ ssh minion@IP_ADDRESS
```

## Cluster deployment customization examples
You may find the need to modify environment variables to change the behaviour of kube-up. Here are some common scenarios:

#### Proxy configuration
If you are behind a proxy, and have your local environment variables setup, you can use these variables to setup your Kubernetes cluster:

```sh
ENABLE_PROXY=true KUBERNETES_PROVIDER=openstack ./kube-up.sh
```

#### Setting different Swift URL
Some deployments differ from the default Swift URL:

```sh
 SWIFT_SERVER_URL="http://10.100.0.100:8080" KUBERNETES_PROVIDER=openstack ./kube-up.sh
```

#### Public network name.
Sometimes the name of the public network differs from the default `public`:

```sh
EXTERNAL_NETWORK="network_external" KUBERNETES_PROVIDER=openstack ./kube-up.sh
```

#### Spinning up additional clusters.
You may want to spin up another cluster within your OpenStack project. Use the `$STACK_NAME` variable to accomplish this.

```sh
STACK_NAME=k8s-cluster-2 KUBERNETES_PROVIDER=openstack ./kube-up.sh
```

For more configuration examples, please browse the files mentioned in the [Configuration](#set-additional-configuration-values) section.


## Tearing down your cluster

To bring down your cluster, issue the following command:

```sh
KUBERNETES_PROVIDER=openstack ./kube-down.sh
```
If you have changed the default `$STACK_NAME`, you must specify the name. Note that this will not remove any Cinder volumes created by Kubernetes.