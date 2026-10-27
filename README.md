# Kubernetes The Hard Way

I think that the best way to learn Kubernetes is not just memorizing the commands by heart, but rather building a naive Kubernetes cluster in a cloud-native environment, starting from distributed systems concepts, Kubernetes networking, and then moving on to Kubernetes security. This project is fork of the initial tutorial, but with additional comments to help you actually understand what's going on, rather than just copy-pasting the given commands.

## Labs

This tutorial requires four (4) ARM64 or AMD64 based virtual or physical machines connected to the same network.

* [Prerequisites](docs/01-prerequisites.md)
* [Setting up the Jumpbox](docs/02-jumpbox.md)
* [Provisioning Compute Resources](docs/03-compute-resources.md)
* [Provisioning the CA and Generating TLS Certificates](docs/04-certificate-authority.md)
* [Generating Kubernetes Configuration Files for Authentication](docs/05-kubernetes-configuration-files.md)
* [Generating the Data Encryption Config and Key](docs/06-data-encryption-keys.md)
* [Bootstrapping the etcd Cluster](docs/07-bootstrapping-etcd.md)
* [Bootstrapping the Kubernetes Control Plane](docs/08-bootstrapping-kubernetes-controllers.md)
* [Bootstrapping the Kubernetes Worker Nodes](docs/09-bootstrapping-kubernetes-workers.md)
* [Configuring kubectl for Remote Access](docs/10-configuring-kubectl.md)
* [Provisioning Pod Network Routes](docs/11-pod-network-routes.md)
* [Smoke Test](docs/12-smoke-test.md)
* [Cleaning Up](docs/13-cleanup.md)

## Lab 0

Before creating virtual machines, we needed to set up a GCP project. A GCP project is a logical container for cloud resources, such as VMs, networks, and firewall rules.

After this step, we need to associate our project with a billing account. We define a budget for this step for 50€ and create an alert to notify us if we exceed this defined amount. 

We created a VPC with the name "kubernetes-vpc". We defined a subnetwork with the name "kubernetes-subnet" within the Europe-West9 (Paris) region, with an IPv4 address range of 
10.0.0.0/24. All our nodes will be inside the same subnetwork.

As a first step to install Kubernetes in our cloud cluster, we decided not to consider the resilience of the control plane. We just want to have something that works. In future versions, we will use the conensus protocol raft to define our architecture. For now, we will use 4 VMs running Debian 12 (Bookworm). The following table lists the four machines and their CPU, memory, and storage requirements, as defined by the initial tutorials.

| Name    | Description            | CPU | RAM   | Storage |
|---------|------------------------|-----|-------|---------|
| jumpbox | Administration host    | 1   | 512MB | 10GB    |
| server  | Kubernetes server      | 1   | 2GB   | 20GB    |
| node-0  | Kubernetes worker node | 1   | 2GB   | 20GB    |
| node-1  | Kubernetes worker node | 1   | 2GB   | 20GB    |

In fact, all our nodes are inside the same subnetwork and they can communicate with each other, but we need to define a firewall to determine if the traffic is authorized or denied.

The first firewall rule should define which hosts or networks (sourceTanges: 10.0.0.0/24) are allowed to access the Kubernetes control plane (targetTags: control-plane) through the kube-apiserver which listens on TCP port 6443 by default.

# Lab 1

We launch our first VM named jumbox which is the administration machine that you will use as a home base when setting up your Kubernetes cluster from the ground up.


