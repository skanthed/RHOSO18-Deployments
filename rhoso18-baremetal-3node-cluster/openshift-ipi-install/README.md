# OpenShift 4.18 Bare-Metal IPI Installation

Scripts and configuration for deploying a compact (3-master, 0-worker) OpenShift 4.18 cluster on bare metal using the IPI (Installer-Provisioned Infrastructure) method with Redfish virtual media provisioning.

---

## Prerequisites

Before starting the OpenShift 4.18 bare-metal IPI installation, prepare the dedicated jump host and installation files.

The jump host setup follows the Red Hat bare-metal IPI environment preparation workflow through:

- 3.1 Installing RHEL on the jump host
- 3.2 Preparing the jump host
- 3.3 Checking NTP synchronization
- 3.4 Configuring networking on the jump host and use the Example bare-metal bridge interface configuration that uses DHCP

Red Hat reference:

https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/deploying_installer-provisioned_clusters_on_bare_metal/ipi-install-installation-workflow

On Jumphost follow the steps below:

### 1. Download and extract the OpenShift client

Download `openshift-client-linux-<version>.tar.gz` from the [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/downloads).

Extract `oc` and `kubectl`, then either:

- Install them globally (for example, `/usr/local/bin`), **or**
- Place them in the repository's `bin/` directory (this repository assumes `bin/`).

```bash
tar -xzf openshift-client-linux-<version>.tar.gz -C bin/ oc kubectl
chmod +x bin/oc bin/kubectl
```

### 2. Download your pull secret

Download your pull secret from the [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/downloads) under **Downloads → Pull Secret** and save it as:

```
assets/pull_secret.json
```

### 3. Configure DNS on the Jump Host
The OpenShift cluster requires DNS records for the API, internal API, wildcard applications domain, and control-plane nodes.

In this deployment, `dnsmasq` runs on the jump host and provides DNS service through the `baremetal` bridge.

#### 3.1 Find the DNS Server IP

The DNS server IP is the IPv4 address assigned to the jump host's `baremetal` bridge.

```bash
ip -4 addr show baremetal
```

#### 3.2 Install and Configure dnsmasq

```bash
sudo dnf install -y dnsmasq
sudo cp dnsmasq-cluster.conf.example /etc/dnsmasq.d/<CLUSTER_NAME>.conf
sudo vi /etc/dnsmasq.d/<CLUSTER_NAME>.conf
```

#### 3.3 Enable and Verify dnsmasq

```bash
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --reload

sudo systemctl enable --now dnsmasq
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
```


### 4. Configure the install-config template

Update `assets/install-config-template.yaml` with your environment-specific configuration, including:

- Cluster name
- Base domain
- Network configuration
- API & Ingress VIPs
- BMC/iDRAC information
- Node IP addresses
- Pull Secret
- SSH Public Key

---

## Installation

```
00. Complete all prerequisites above
01. ./01_extract_installer.sh       # Pulls openshift-baremetal-install from the release image
02. ./02_run_installation.sh        # Runs the full cluster installation (~45–90 min)
03. ./cleanup.sh                    # Run only if you need to tear down the cluster
```

After a successful install, cluster credentials are written to:

```
<CLUSTER_NAME>/auth/kubeconfig
<CLUSTER_NAME>/auth/kubeadmin-password
```

Export the kubeconfig to start using the cluster:

```bash
export KUBECONFIG=$(pwd)/<CLUSTER_NAME>/auth/kubeconfig
```

Verify the cluster:

```bash
oc get nodes
oc get clusteroperators
```

---

## MachineConfigs

The `assets/MC/` directory contains optional MachineConfig resources for customizing the networking configuration of the control-plane nodes.

Apply the required MachineConfigs after the cluster installation, depending on your environment and deployment requirements.

```bash
oc apply -f assets/MC/
```

---

## Notes

* `<CLUSTER_NAME>/` is created automatically during installation and contains cluster state, credentials, certificates, and logs.
* `assets/install-config.yaml` is generated from `assets/install-config-template.yaml` before each installation.
* `assets/pull_secret.json` contains registry credentials and must never be committed to Git.
* `cleanup.sh` removes the generated installation artifacts and powers off the configured bare-metal nodes.
