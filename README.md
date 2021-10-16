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

**Access masters and workers** using `docker exec -it` instead of `ssh`
to access these systems.

## Next Steps

**Simulate two master control planes in different data centers
(networks).**
