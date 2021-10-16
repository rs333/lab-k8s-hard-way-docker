# Kubernetes the Hard Way in Docker (Only, No Cloud)

This lab follows roughly the same steps as the popular [Kubernetes the
Hard Way] "tutorial" from Kelsey Hightower, but without the cost and
Google Cloud Platform hassle. We just use Docker instead, each running
container as a "compute instance" (machine). This is faster and easier
to get through and focuses on the core learning content (without wasting
time on GCP). We use the built-in network that Docker provides.

> 💬
> This is the similar to the approach both Kind and Minikube use,
> but without the magic leaving that to you to learn.

## Steps


**Create two Docker networks** one for the control plane (master nodes)
and one for the workers (worker nodes).

```bash
docker network create k8s-control
docker network create k8s-workers
```

* docker network create \| Docker Documentation  
  <https://docs.docker.com/engine/reference/commandline/network_create/>

**Create some master nodes for control plane.**

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



## Next Steps

**Simulate two master control planes in different data centers
(networks).**
