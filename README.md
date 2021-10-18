# Kubernetes the Hard Way in Docker (No Cloud)

This lab follows roughly the same steps as the popular [Kubernetes the
Hard Way] "tutorial" from Kelsey Hightower, but without the cost and
Google Cloud Platform hassle. We just use Docker instead and create
one container for each machine that we'll call *host containers* (or
simply *hosts* from now on). Host containers are easier and more
flexible to setup than host machines, virtual or real --- especially
since we can just mount `/var/run/docker.sock` into each and everything
ends up using a single Docker runtime engine. This is faster and easier
to get through and focuses on the core learning content. We use the
built-in networking that Docker provides to simulate different bridged
networks and even data centers.

> 💬
> This Docker approach is the similar to the approach both Kind and
> Minikube use, but without the magic leaving that for you you to learn.

## Overview

We take a phased, repetitive approach, adding complexity on each
repetition but starting with just the minimum, a single-node cluster:

Stage One:

1. Create a single node to contain control-plane and worker
1. Install control-plane components
1. Install the worker components
1. Joint the worker to the control-plan (both on same node)
1. Test basic Kubernetes functionality

Stage Two:

1. Create two bridged networks: `control-net` and `worker-net`
1. Create `control-1` control node host on `control` network
1. Create `worker-1` worker node host on `workers` network
1. Install control-plane components on `control-1`
1. Install worker components on `worker-1`
1. Join `worker-1` to the cluster
1. Test basic Kubernetes functionality

## Stage One

**Create a single node to contain control-plane and worker.**

Our hosts will be running Ubuntu (but you could be using any other OS
that you want). Let's create a base host container:

```
docker run -itd -h single --name single ubuntu
```

Make sure you can connect to it.

```
docker exec -it single bash
```

We will need to turn this into more of an actual host than is available
from just a container. Let's install some tools we will need. 

First let's update `apt` and do the usual things when dealing with a
container. You might consider putting this into a Dockerfile as you go
so you can save the work later.

```
apt upgrade
apt update
```

Now let's get `vim` and the package necessary to do installation:

```
apt install vim-tiny curl gnupg
```

Now we are ready to install all the control-plane components:

1. Container Engine (Docker)
1. Kubelet
1. KubeProxy

Let's start with the container engine (Docker). Even though we will be
cheating and using the docker engine running on system by mounting
`/var/run/docker.sock` we still need the `docker` command. There are
many ways to get this, but let's use the official docker package
archive.

We'll need to get the public key from docker.

```
curl -sSL -o /tmp/docker.pub https://download.docker.com/linux/ubuntu/gpg
```

Now we can turn it back into a binary file.

```
gpg --dearmor /tmp/docker.pub
```

And add it to the keyrings that the `apt` command uses. Note that the
`.gpg` suffix was added by `--dearmor`.

```
mv /tmp/docker.pub.gpg /usr/share/keyrings/docker-archive-keyring.gpg
```

Now we have to add the docker package archive to our list of approved
`apt` archive sources. You could add the following directly to
`/etc/apt/sources.list` but it is probably better to add a file to the
`/etc/apt/sources.list.d` directory instead. Change the following to
match your architecture and release. When in doubt, open the
`/etc/apt.sources.list` file to see what should be in yours (or use
`dpkg --print-architecture`). You can edit the file directly with `vi`
(`vim`) or just `echo` it to the file with a redirect.

```
deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu focal stable
```

Now just update your apt repository cache and install Docker Community
Edition. (Note that `docker-cd-cli` and `containerd.io` packages will
also be installed with `docker-cd`.)

```
apt update
apt -y install docker-ce docker-ce-cli containerd.io
```

Take your new `docker` command for spin to test it out.

```
docker version
```

Note the error is produces.

```
docker version
Client: Docker Engine - Community
 Version:           20.10.9
 API version:       1.41
 Go version:        go1.16.8
 Git commit:        c2ea9bc
 Built:             Mon Oct  4 16:08:29 2021
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

To get around this we will exit, stop, and remove our container and
restart it with the volume mount connecting the `/var/run/docker.sock`
of the host machine. This is a regular practice to allow accessing the
same docker runtime engine from containers as the host of the
containers.

> 🛑This will completely discard all the work you have done so far. So
> you will have to repeat all the previous steps (which is probably best
> to practice them). Make sure you write everything down (or just go
> back in this document). If you wish (and know how), you can begin to
> create your own Dockerfile image and start using that instead.

```
exit
docker stop single
docker rm single
docker run -itd -h single --name single -v /var/run/docker.sock:/var/run/docker.sock ubuntu
```

You can see from the `inspect` output that a new volume has been bound.

```
docker inspect single |jq '.[0].Mounts'
```

After you have repeated all the steps to get `docker-ce` installed you
can try `docker ps` and should see your host machine container running
(yes, even though you are actually in it).

```
CONTAINER ID   IMAGE     COMMAND   CREATED          STATUS          PORTS     NAMES
54348c3700aa   ubuntu    "bash"    26 minutes ago   Up 26 minutes             single
```

Huzzah! Docker is not installed. On to Kubelet.

----








TODO Rethink where the following should go.


## Steps

**Create two Docker networks** one for the control plane (control nodes)
and one for the workers (worker nodes).

```bash
docker network create k8s-control
docker network create k8s-workers
```

* docker network create \| Docker Documentation  
  <https://docs.docker.com/engine/reference/commandline/network_create/>

**Create some control nodes for control plane.**

```bash
for i in {1..3};do
  docker run -itd \
    --network=k8s-control \
    --name control-$i \
    -h control-$i \
    ubuntu
done
```

**Create some workers.**

```bash
for i in {1..3};do
  docker run -itd \
    --network=k8s-workers \
    --name worker-$i \
    -h worker-$i \
    ubuntu
done
```

**Note the IP addresses of all the nodes (masters and workers)** which
will be needed when creating the certificates.

```bash
mapfile -t names < <(d ps --format '{{.Names}}')
for n in "${names[@]}"; do
  echo -n "$n "
  d inspect --format '{{json .}}' $n \
    | jq -r '.NetworkSettings.Networks[].IPAddress'
done
```

**Test access to masters and workers** using `docker exec -it` instead
of `ssh` to access these systems.

```bash
docker exec -it worker-1 bash
```

Should dump you onto a prompt:

```
root@worker-1:/#
```

**Install dependencies on all host containers, workers and .**[



> ⚠️
> At this point you can choose either to create certificates using your
> own Certificate Authority, or you can just use `kubeadm` and have your
> certificates managed with Kubernetes.

## Next Steps

**Simulate two master control planes in different data centers
(networks).**

## Alternate Paths

**Provision a Certificate Authority** using the original, standard,
ubiquitous `openssl` tool (there are others you can learn later).

**Generate CA key** which will be used to create and sign all the other
keys and create certificates.

```bash
openssl genrsa -o ca.key 2048
```

**Generate CA X.509 certificate** from the key. `req` is from the
term Certificate Signing Request (CSR). We'll use 10,000 days since we
don't need to worry about certificates expiring, but in real scenarios
you should setup a reminder immediately for before the cert expires to
be sure it gets replaced before then (or very bad things will happen).

```bash
openssl req -x509 
```
