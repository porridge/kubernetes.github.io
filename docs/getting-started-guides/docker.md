---
---

The following instructions show you how to set up a simple, single node Kubernetes cluster using Docker.

Here's a diagram of what the final result will look like:

![Kubernetes Single Node on Docker](/images/docs/k8s-singlenode-docker.png)

* TOC
{:toc}

## Prerequisites

1. You need to have docker installed on one machine.
2. Decide what Kubernetes version to use.  Set the `${K8S_VERSION}` variable to
   a released version of Kubernetes >= "1.2.0-alpha.7"

### Run it

```shell
docker run \
    --volume=/:/rootfs:ro \
    --volume=/sys:/sys:ro \
    --volume=/var/lib/docker/:/var/lib/docker:rw \
    --volume=/var/lib/kubelet/:/var/lib/kubelet:rw \
    --volume=/var/run:/var/run:rw \
    --net=host \
    --pid=host \
    --privileged=true \
    -d \
    gcr.io/google_containers/hyperkube-amd64:v${K8S_VERSION} \
    /hyperkube kubelet \
        --containerized \
        --hostname-override="127.0.0.1" \
        --address="0.0.0.0" \
        --api-servers=http://localhost:8080 \
        --config=/etc/kubernetes/manifests \
        --cluster-dns=10.0.0.10 \
        --cluster-domain=cluster.local \
        --allow-privileged=true --v=2
```

> Note that `--cluster-dns` and `--cluster-domain` is used to deploy dns, feel free to discard them if dns is not needed.

> If you would like to mount an external device as a volume, add `--volume=/dev:/dev` to the command above. It may however, cause some problems described in [#18230](https://github.com/kubernetes/kubernetes/issues/18230)

This actually runs the kubelet, which in turn runs a [pod](/docs/user-guide/pods/) that contains the other master components.

### Download `kubectl`

At this point you should have a running Kubernetes cluster.  You can test this
by downloading the kubectl binary for `${K8S_VERSION}` (look at the URL in the
following links) and make it available by editing your PATH environment
variable.
([OS X/amd64](http://storage.googleapis.com/kubernetes-release/release/{{page.version}}.0-alpha.7/bin/darwin/amd64/kubectl))
([OS X/386](http://storage.googleapis.com/kubernetes-release/release/{{page.version}}.0-alpha.7/bin/darwin/386/kubectl))
([linux/amd64](http://storage.googleapis.com/kubernetes-release/release/{{page.version}}.0-alpha.7/bin/linux/amd64/kubectl))
([linux/386](http://storage.googleapis.com/kubernetes-release/release/{{page.version}}.0-alpha.7/bin/linux/386/kubectl))
([linux/arm](http://storage.googleapis.com/kubernetes-release/release/{{page.version}}.0-alpha.7/bin/linux/arm/kubectl))

For example, OS X:

```shell
$ wget http://storage.googleapis.com/kubernetes-release/release/v${K8S_VERSION}/bin/darwin/amd64/kubectl
$ chmod 755 kubectl
$ PATH=$PATH:`pwd`
```

Linux:

```shell
$ wget http://storage.googleapis.com/kubernetes-release/release/v${K8S_VERSION}/bin/linux/amd64/kubectl
$ chmod 755 kubectl
$ PATH=$PATH:`pwd`
```

Create configuration:

```shell
$ kubectl config set-cluster test-doc --server=http://localhost:8080
$ kubectl config set-context test-doc --cluster=test-doc
$ kubectl config use-context test-doc
```

For Max OS X users instead of `localhost` you will have to use IP address of your docker machine,
which you can find by running `docker-machine env <machinename>` (see [documentation](https://docs.docker.com/machine/reference/env/)
for details).

### Test it out

List the nodes in your cluster by running:

```shell
kubectl get nodes
```

This should print:

```shell
NAME        LABELS                             STATUS
127.0.0.1   kubernetes.io/hostname=127.0.0.1   Ready
```

### Run an application

```shell
kubectl run nginx --image=nginx --port=80
```

Now run `docker ps` you should see nginx running.  You may need to wait a few minutes for the image to get pulled.

### Expose it as a service

```shell
kubectl expose deployment nginx --port=80
```

Run the following command to obtain the IP of this service we just created. There are two IPs, the first one is internal (CLUSTER_IP), and the second one is the external load-balanced IP (if a LoadBalancer is configured)

```shell
kubectl get svc nginx
```

Alternatively, you can obtain only the first IP (CLUSTER_IP) by running:

```shell
kubectl get svc nginx --template={{.spec.clusterIP}}
```

Hit the webserver with the first IP (CLUSTER_IP):

```shell
curl <insert-cluster-ip-here>
```

Note that you will need run this curl command on your boot2docker VM if you are running on OS X.

## Deploy a DNS

See [here](/docs/getting-started-guides/docker-multinode/deployDNS/) for instructions.

### A note on turning down your cluster

Many of these containers run under the management of the `kubelet` binary, which attempts to keep containers running, even if they fail.  So, in order to turn down
the cluster, you need to first kill the kubelet container, and then any other containers.

You may use `docker kill $(docker ps -aq)`, note this removes _all_ containers running under Docker, so use with caution.

### Troubleshooting

#### Node is in `NotReady` state

If you see your node as `NotReady` it's possible that your OS does not have memcg and swap enabled.

1. Your kernel should support memory and swap accounting. Ensure that the
following configs are turned on in your linux kernel:

    ```shell
    CONFIG_RESOURCE_COUNTERS=y
    CONFIG_MEMCG=y
    CONFIG_MEMCG_SWAP=y
    CONFIG_MEMCG_SWAP_ENABLED=y
    CONFIG_MEMCG_KMEM=y
    ```

2. Enable the memory and swap accounting in the kernel, at boot, as command line
parameters as follows:

    ```shell
    GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
    ```

    NOTE: The above is specifically for GRUB2.
    You can check the command line parameters passed to your kernel by looking at the
    output of /proc/cmdline:

    ```shell
    $ cat /proc/cmdline
    BOOT_IMAGE=/boot/vmlinuz-3.18.4-aufs root=/dev/sda5 ro cgroup_enable=memory swapaccount=1
    ```
