---
title: Istio Kubernetes Mesh Expansion
overview: Instructions to add external machines and expanding Istio.

order: 25

layout: docs
type: markdown
---

Instructions to configure Istio on a Kubernetes Cluster so it can be expanded with
services running on cloud or on prem VMs.

_This document is under construction._

## Prerequisites

* You have [installed Istio](install-kubernetes.html) on Kubernetes.

* The machine must have IP connectivity to the endpoints in the mesh. This
typically requires same VPC or a VPN connection, as well as a container network that
provides direct (without NAT or firewall deny) routing to the endpoints. The machine
is not required to have access to the cluster IP addresses assigned by K8S.

* The control plane (Pilot, Mixer, CA) and Kubernetes DNS server must be accessible
from the VMs. This is typically done using an [Internal Load
Balancer](https://kubernetes.io/docs/concepts/services-networking/service/#internal-load-balancer).
Other options are possible, for example NodePort, or running Istio components on VMs, or
custom network configurations - separate documents will cover advanced configs.

## Installation steps

Setup consists of preparing the mesh for expansion and installing and configuring each VM.

An example script to help with K8S setup is available in
[install/tools/setupMeshEx.sh](https://raw.githubusercontent.com/istio/istio/master/install/tools/setupMeshEx.sh).

An example script to help configure a machine is available in [install/tools/setupIstioVM.sh](https://raw.githubusercontent.com/istio/istio/master/install/tools/setupIstioVM.sh).
You should customize it based on your provisioning tools and DNS requirements.

### Preparing the K8s for exapansion

* Setup internal load balancers for Kube DNS, Pilot, Mixer and CA. This step is specific to
each cluster, you may need to add annotations.

```bash
  admin$ install/tools/setupMeshEx initCluster
  
  or 
  
  admin$ kubectl apply -f install/kubernetes/meshex.yaml
```

* Generate the Istio 'cluster.env' config to be deployed in the VMs. This file contains
the cluster IP address ranges to intercept. 

```bash
  admin$ install/tools/setupMeshEx generateConfigs MY_CLUSTER_NAME
  
```

Example generated files:

   ```bash

    vm$ cat /usr/local/istio/proxy/cluster.env
   ISTIO_SERVICE_CIDR=10.23.240.0/20

  ```
* Generate DNS config file to be used in the VMs. This will allow apps on the VM to resolve
cluster service names, which will be intercepted by the sidecar and forwarded.

```bash
  admin$ install/tools/setupMeshEx generateConfigs MY_CLUSTER_NAME

```

Example generated files:

   ```bash

   vm$ cat /etc/dnsmasq.d/kubedns
   server=/svc.cluster.local/10.128.0.6
   address=/istio-mixer/10.128.0.7
   address=/mixer-server/10.128.0.7
   address=/istio-pilot/10.128.0.5
   address=/istio-ca/10.128.0.8

  ```

### Setting up the machines

* Copy the config files and istio debian files to each machine joining the cluster, as
/etc/dnsmasq.d/kubedns and /var/lib/istio/envoy/cluster.env.

* Configure and verify DNS settings - this may require installing dnsmasq, adding it to
/etc/resolv.conf directly or via DHCP scripts. To verify, check that the VM can resolve
names and connect to pilot, for example:

    ```bash

    vm$ dig istio-pilot.istio-system
    # This should be the same address shown as "EXTERNAL-IP" in 'kubectl get svc -n istio-system istio-pilot-ilb'
    ...
    istio-pilot.istio-system. 0	IN	A	10.128.0.5
    ...


    # Check that we can resolve cluster IPs. The actual IN A will depend on cluster config.
    vm$ dig istio-pilot.istio-system.svc.cluster.local.
    ...
    istio-pilot.istio-system.svc.cluster.local. 30 IN A 10.23.251.121

    vm$ dig istio-ingress.istio-system.svc.cluster.local.
    ...
    istio-ingress.istio-system.svc.cluster.local. 30 IN A 10.23.245.11


    ```

* Verify connectivity. We'll check if the VM can connect to the pilot, and it can connect to
an endpoint.

    ```bash
    vm$ curl -v 'http://istio-pilot.istio-system:8080/v1/registration/istio-pilot.istio-system.svc.cluster.local|http-discovery'
    ...
    "ip_address": "10.20.1.18",
    ...


    # Use the address above - it will connect directly the the pod running istio-pilot.
    vm$ curl -v 'http://10.20.1.18:8080/v1/registration/istio-pilot.istio-system.svc.cluster.local|http-discovery'

    ```

* Extract the initial Istio authentication secrets and copy them to the machine. The default
installation of Istio includes IstioCA and will generate Istio secrets even if automatic 'mTLS'
setting is disabled. It is recommended that you perform this step to make it easy to
enable mTLS in future and upgrade to future version that will have mTLS enabled by default.

'''bash
  # ACCOUNT defaults to 'istio.default', or SERVICE_ACCOUNT environment variable
  # NAMESPACE defaults to current namespace, or SERVICE_NAMESPACE environment variable

  admin$ install/tools/setupMeshEx machineCerts ACCOUNT NAMESPACE

```

The generated files (key.pem, root-cert.pem, cert-chain.pem) must be copied to /etc/certs on each machine, readable by istio-proxy.

* Install istio debian files and start 'istio' and 'istio-auth-node-agent' services.

  ```bash

      ISTIO_VERSION=0.2.4 # Update with the current istio version
      DEBURL=http://gcsweb.istio.io/gcs/istio-release/releases/${ISTIO_VERSION}/deb
      curl -L ${DEBURL}/istio-agent-release.deb > istio-agent-release.deb
      curl -L ${DEBURL}/istio-auth-node-agent-release.deb > istio-auth-node-agent-release.deb
      curl -L ${DEBURL}/istio-proxy-release.deb > istio-proxy-release.deb

      dpkg -i ${ISTIO_STAGING}/istio-proxy-envoy.deb
      dpkg -i ${ISTIO_STAGING}/istio-agent.deb
      dpkg -i ${ISTIO_STAGING}/istio-auth-node-agent.deb

      # TODO: This will be replaced with an 'apt-get' command once the repositories are setup.

      systemctl start istio
      systemctl start istio-auth-node-agent
  ```


After setup, the machine should be able to access services running in the k8s cluster
or other mesh expansion machines.

```bash
   # Assuming you install bookinfo in 'bookinfo' namespace
   vm$ curl productpage.bookinfo.svc.cluster.local:9080
   ... html content ...

```

## Running services on a mesh expansion machine

* Configure the sidecar to intercept the port. This is configured in /var/lib/istio/envoy/sidecar.env,
using the ISTIO_INBOUND_PORTS environment variable.

  Example (on the VM running the service):

   ```bash

   echo "ISTIO_INBOUND_PORTS=27017,3306,8080" > /var/lib/istio/envoy/sidecar.env
   systemctl restart istio
   ```

* Manually configure a selector-less service and endpoints. The 'selector-less' service is used for
services that are not backed by Kubernetes pods.

   Example, on a machine with permissions to modify k8s services:
   ```bash

   # istioctl register servicename machine-ip portname:port
   istioctl -n onprem register mysql 1.2.3.4 3306
   istioctl -n onprem register svc1 1.2.3.4 http:7000
   ```

After the setup, k8s pods and other mesh expansions should be able to access the
services running on the machine.