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

The first firewall rule should define which hosts or networks (sourceRanges: 10.0.0.0/24) are allowed to access the Kubernetes control plane (targetTags: control-plane) through the kube-apiserver which listens on TCP port 6443 by default.

## Lab 1 & 2 : Provisioning the cluster infrastructure

### Administration machine

We first create a VM named `jumbox`. This machine acts as the administration host from which we will configure the Kubernetes cluster. We assign it the network tag jumbox. To connect to jumbox from GCP shell, we create an ingress firewall rule named `allow-ssh-iap-jumpbox`. This rule allows SSH traffic on TCP port 22 from the IAP IPv4 range 35.235.240.0/20 to VMs with the target tag jumbox.

### SSH access to the server nodes

We then create a second VM named `server`, which will host the Kubernetes control plane. We assign it the network tag `control-plane`. To allow SSH connections from jumbox to server, we create an ingress firewall rule named `allow-ssh-jumpbox-server`. It allows TCP port 22 from VMs with the source tag jumbox to VMs with the target tag control-plane.

### SSH authentication

Network connectivity alone is not sufficient to establish an SSH session. We also need to configure authentication. On jumbox, we generate an Ed25519 public and private key pair. We keep the private key secure on jumbox and copy only the public key, id_ed25519.pub.

To configure the public key on server, we create a temporary ingress firewall rule named `allow-ssh-iap-server`. This rule allows TCP port 22 from the IAP IPv4 range 35.235.240.0/20 to VMs with the target tag control-plane. We then connect to server through IAP using an administrator account with sudo privileges. We add the public key from jumbox to /root/.ssh/authorized_keys.

To allow root login using a public key, we check the effective SSH daemon configuration. If necessary, we set PermitRootLogin to prohibit-password, validate the configuration with sshd -t, and reload the SSH service. We can then connect from jumbox to server as root using the server's internal IP address. After setup, we can remove the temporary IAP-to-server rule if it is no longer needed.

> Direct root login is used here for the lab. For routine administration, an individual account with sudo is generally preferable.

### Worker nodes and IP forwarding

Finally, we create two additional VMs, `node-0` and `node-1`, which will act as the Kubernetes worker nodes. For this lab, we enable IP forwarding on both VMs by using the `--can-ip-forward` option when creating them. This supports a routed Pod network in which workers forward packets on behalf of Pods.

### SSH access to the worker nodes

We repeat the same steps as to have ssh access to the server node.

### Install Command Line Utilities

Now that we are logged into the jumpbox machine as the root user, we install the command line utilities that we will use throughout the tutorial.

```bash
{
  apt-get update
  apt-get -y install wget curl vim openssl git
}
```

### Sync GitHub Repository

We then download a copy of the original tutorial, which contains the configuration files and templates we will use to build our Kubernetes cluster. From `/root` on the jumpbox, we clone the repository with `--depth 1` because we only need its latest files, without the full commit history.

```bash
git clone --depth 1 https://github.com/kelseyhightower/kubernetes-the-hard-way.git

cd /root/kubernetes-the-hard-way
```

### Downloading the binaries

We download the binaries on the jumpbox before transferring them to the other machines with `scp` in the following labs. This allows us to download each file once and prepare the components in one place.

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads-$(dpkg --print-architecture).txt
```

### Extracting the archives

We prepare four directories to separate the client tools, network plugins, control plane components, and worker components.

```bash
cd /root/kubernetes-the-hard-way

ARCH=$(dpkg --print-architecture)

mkdir -p downloads/{client,cni-plugins,controller,worker}
```

We can now extract the archives. 

```bash
tar -xf downloads/crictl-v1.32.0-linux-${ARCH}.tar.gz \
  -C downloads/worker/

tar -xf downloads/containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz \
  --strip-components 1 \
  -C downloads/worker/

tar -xf downloads/cni-plugins-linux-${ARCH}-v1.6.2.tgz \
  -C downloads/cni-plugins/

tar -xf downloads/etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz \
  -C downloads/ \
  --strip-components 1 \
  etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl \
  etcd-v3.6.0-rc.3-linux-${ARCH}/etcd
```

### Organizing the binaries

We then move the remaining binaries into the directories that correspond to their role. The `client` directory contains `kubectl` for interacting with the Kubernetes API and `etcdctl` for interacting with etcd. The `controller` directory contains `etcd` and the control plane components: `kube-apiserver` exposes the Kubernetes API, `kube-controller-manager` runs the controllers that reconcile the cluster's current state with its desired state, and `kube-scheduler` selects a worker node for each Pod that needs one.

On the workers, `kubelet` manages the Pods assigned to the node, and `kube-proxy` configures network rules for reaching Kubernetes Services. We also rename `runc.${ARCH}` to `runc`, the executable name that containerd will use to run containers. These directories will make it easier to copy the required components to each machine later.

```bash
mv downloads/{etcdctl,kubectl} downloads/client/

mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
  downloads/controller/

mv downloads/{kubelet,kube-proxy} downloads/worker/

mv downloads/runc.${ARCH} downloads/worker/runc
```

We make the binaries executable.

```bash
chmod +x downloads/{client,cni-plugins,controller,worker}/*
```

### Installing kubectl

We install `kubectl` on the jumpbox by copying it to `/usr/local/bin/`, a directory in our PATH.

```bash
cd /root/kubernetes-the-hard-way

cp downloads/client/kubectl /usr/local/bin/
```

## Lab 3 : Preparing the machines and local name resolution

### Creating the machine inventory

Now that our four VMs exist and we have configured SSH access, we prepare a `machines.txt` file on the jumpbox. This file serves as an inventory for the loops that will configure the three cluster machines. We do not include the jumpbox because it is the administration host from which we run those commands.

Each line contains the machine's internal IP address, its FQDN, and its hostname. For the workers, we also specify the subnet from which Pods running on that node will receive their IP addresses. These Pod subnets are separate from the `10.0.0.0/24` subnet used by the VMs, and each worker has its own range. We will configure routing between these subnets so that Pods on different workers can communicate.

```bash
cat > machines.txt <<'EOF'
10.0.0.3 server.kubernetes.local server
10.0.0.4 node-0.kubernetes.local node-0 10.200.0.0/24
10.0.0.5 node-1.kubernetes.local node-1 10.200.1.0/24
EOF
```

### Checking SSH access

From the jumpbox, we check that we can connect to each machine using the private key we created earlier. 

```bash
while read -r IP FQDN HOST SUBNET; do
  ssh -n -i /root/.ssh/id_ed25519 root@${IP} hostname
done < machines.txt
```

### Checking the fully qualified domain names

We then repeat the check with `hostname --fqdn` to see the fully qualified domain name currently resolved by each machine.

```bash
while read -r IP FQDN HOST SUBNET; do
  echo "Machine : $HOST"
  ssh -n -i /root/.ssh/id_ed25519 root@"$IP" hostname --fqdn
done < machines.txt
```

> On our GCP machines, we observe names in the format `{server_name}.{zone}.{project_id}.internal`. In the following steps, we add local name mappings for the names in our inventory. These entries do not change Google Cloud DNS or the machines configured hostnames.

### Configuring name resolution on the jumpbox

We create a `hosts` file mapping each machine's IP address to its FQDN and hostname, then append these entries to `/etc/hosts` on the jumpbox so that we can access the cluster machines by name.

```bash
while read -r IP FQDN HOST SUBNET; do
  echo "$IP $FQDN $HOST" >> hosts
done < machines.txt

cat hosts >> /etc/hosts
```

### Configuring name resolution on the cluster machines

We copy the generated `hosts` file to each server and worker using `scp`, then use SSH to append its contents to the remote `/etc/hosts`, quoting the command so that the redirection runs on the remote machine.

```bash
while read -r IP FQDN HOST SUBNET; do
  scp -i /root/.ssh/id_ed25519 hosts root@"$IP":/root/kthw-hosts
  ssh -n -i /root/.ssh/id_ed25519 root@"$IP" 'cat /root/kthw-hosts >> /etc/hosts'
done < machines.txt
```

At this point, all four machines have local mappings for the cluster names. These mappings allow name lookup, but they do not replace the firewall rules and SSH authentication configured earlier.


## Lab 4 : Provisioning a CA and generating TLS certificates

Now that our machines can communicate, we create a Certificate Authority and the certificates used to authenticate Kubernetes components and establish TLS connections. We run the following commands on the jumpbox.

### Certificate configuration

We use the `ca.conf` file from the tutorial to define the identities, usages, and names associated with our certificates.

```bash
cd /root/kubernetes-the-hard-way

cat ca.conf
```

For API access, we will use `server.kubernetes.local`, which is already included in the API server's certificate configuration. Each worker's client certificate identifies its kubelet as `system:node:node-0` or `system:node:node-1`, matching the name it will use to register with the cluster.

### Creating the Certificate Authority

We first generate the CA private key, `ca.key`, and its self-signed certificate, `ca.crt`. We use the private key to sign the other certificates, while the CA certificate allows clients and servers to verify them.

```bash
{
  openssl genrsa -out ca.key 4096
  openssl req -x509 -new -sha512 -noenc \
    -key ca.key -days 3653 \
    -config ca.conf \
    -out ca.crt
}
```

### Generating the certificates

We prepare the list of identities for which we will generate private keys and certificates, including the `admin` user, the Kubernetes components, and the ServiceAccount signing key pair.

```bash
certs=(
  "admin" "node-0" "node-1"
  "kube-proxy" "kube-scheduler"
  "kube-controller-manager"
  "kube-api-server"
  "service-accounts"
)
```

For each identity, we generate a private key (`.key`), create a certificate signing request (`.csr`) using its section in `ca.conf`, and sign that request with our CA to produce the certificate (`.crt`).

```bash
for cert in "${certs[@]}"; do
  openssl genrsa -out "${cert}.key" 4096

  openssl req -new -key "${cert}.key" -sha256 \
    -config ca.conf -section "${cert}" \
    -out "${cert}.csr"

  openssl x509 -req -days 3653 -in "${cert}.csr" \
    -copy_extensions copyall \
    -sha256 -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out "${cert}.crt"
done
```

The `service-accounts.key` file will be used to sign ServiceAccount tokens, and the public key contained in `service-accounts.crt` will be used to verify them.

### Checking the certificates

We list the generated files and verify that each certificate is signed by our CA.

```bash
ls -1 *.crt *.key *.csr

for cert in "${certs[@]}"; do
  openssl verify -CAfile ca.crt "${cert}.crt"
done
```

### Distributing the certificates to the workers

We copy the CA certificate and each worker's certificate and private key to `/var/lib/kubelet/`, using the filenames that the kubelet configuration expects.

```bash
for host in node-0 node-1; do
  ssh -n -i /root/.ssh/id_ed25519 root@"${host}" \
    mkdir -p /var/lib/kubelet/

  scp -i /root/.ssh/id_ed25519 ca.crt \
    root@"${host}":/var/lib/kubelet/

  scp -i /root/.ssh/id_ed25519 "${host}.crt" \
    root@"${host}":/var/lib/kubelet/kubelet.crt

  scp -i /root/.ssh/id_ed25519 "${host}.key" \
    root@"${host}":/var/lib/kubelet/kubelet.key
done
```

### Distributing the certificates to the server

We copy the CA, API server, and ServiceAccount files to `/root/` on the server, where we will use them when configuring the control plane. We also copy `ca.key` because the controller manager will use it to sign certificates.

```bash
scp -i /root/.ssh/id_ed25519 \
  ca.key ca.crt \
  kube-api-server.key kube-api-server.crt \
  service-accounts.key service-accounts.crt \
  root@server:/root/
```

We keep the generated files on the jumpbox to create the client authentication configuration files, called kubeconfigs, in the next lab.
