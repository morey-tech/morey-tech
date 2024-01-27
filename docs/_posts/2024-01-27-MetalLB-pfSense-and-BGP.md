---
title: "It's like Magic - MetalLB, pfSense, and BGP"
date: 2024-01-27 00:00:00 -0000
excerpt_separator: "<!--more-->"
categories:
  - Technical Blog
tags:
  - homelab
  - maas
  - bare-metal
  - kubernetes
header:
  image: /assets/images/Xnapper-2024-01-27-16.37.24.png
---

[Services in Kubenetes](https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/) can be exposed via a load balancer with an external IP address when given the `type: LoadBalancer`. When using an IaaS platform (e.g. AWS, GCP, DigitalOcean, Civo), they provide the load balancer implementation, typically using a controller in the cluster that creates the resource outside of the cluster (e.g. [aws-load-balancer-controller](https://github.com/kubernetes-sigs/aws-load-balancer-controller)).

However, Kubernetes does not implement network load balancers for bare-metal clusters. So you are left with "NodePort" and "externalIPs" services to route traffic into the clusters. Both of these are insufficient for production use. 

MetalLB offers a network load balancer implementation that integrates with standard network equipment (e.g. pfSense) so that external services on bare-metal clusters "just work". Combined with [pfSense](https://pfsense.org/) and [BGP](https://www.cloudflare.com/en-ca/learning/security/glossary/what-is-bgp/), it becomes like magic. Especially to me, who has a solid understanding of networking principles but not so much of complex routing.

Each time a Service is configured to receive an external IP address (external to the cluster network, not necessarily exposed to the internet) from MetalLB, it will be immediately (within milliseconds) known by pfSense and accessible to the rest of the network.

## The Setup

### Prerequisites

I'll assume that pfSense is already installed and configured with the bare minimum configuration. Basically, pfSense should be the router for the LAN network that the Kubernetes nodes are connected to. And that you have a (bare-metal) Kubernetes cluster with [a CNI compatible with MetalLB](https://metallb.universe.tf/installation/network-addons/).

### My Environment

In my case, I have a bare-metal Kubernetes cluster comprised of 3 nodes, running MicroK8s with Calico as the CNI. With pfSense running in a VM under Proxmox, hosted on a bare-metal host dedicated to networking services, acting as the network's primary router (and default gateway).

I'll reference the following attributes in the configuration throughout the setup steps.

| Attribute | Value |
| --- | --- |
| VRF Name | default |
| pfSense ASN | 65100 |
| pfSense Router ID | 192.168.3.1 |
| BGP network | 10.8.0.0/16 |
| MetalLB ASN | 65101 |
| Neighbor IP Addresses | 192.168.3.15<br>192.168.3.16<br>192.168.3.17<br>192.168.3.18 |

### pfSense

pfSense implements BGP using [the (FRR) project](https://frrouting.org), which is a free and open-source Internet routing protocol suite for Linux. It implements not only BGP but also OSPF, RIP, IS-IS, PIM, LDP, BFD, Babel, PBR, OpenFabric and VRRP, with alpha support for EIGRP and NHRP.

- Install the FRR package on pfSense.
    - Go to "System / Package Manager / Available Packages".
    - Search for the package "frr."
    - Click "+ Install".
    
    ![Xnapper-2024-01-27-15.10.13.png](/assets/images/Xnapper-2024-01-27-15.10.13.png)
    
    ![Xnapper-2024-01-27-15.11.18.png](/assets/images/Xnapper-2024-01-27-15.11.18.png)
    
- Enable the FRR service.
    - Go to "Services / FRR / Global Settings".
    - Click the "Enable FRR" checkbox.
    - Enter a "Master Password" (this can be generated and isn't used later).
    - Click "Save" at the bottom of the page.
    
    ![Xnapper-2024-01-27-11.41.28.png](/assets/images/Xnapper-2024-01-27-11.41.28.png)
    
- Add a Route Map to tell pfSense to accept any routes being sent to it from other routers.
    - Go to "Services / FRR / Global Settings".
    - Set the "Name" to "Permit-Any".
    - Set the "Description to "Match any route".
    - Set the "Action" to "Permit".
    - Set the "Sequence" to "100".
    - Click "Save" at the bottom of the page.
    
    ![Xnapper-2024-01-27-14.51.21.png](/assets/images/Xnapper-2024-01-27-14.51.21.png)
    
- Enable BGP and set the ASN.
    - Go to "Services / FRR / BGP / BGP".
    - Click the "Enable BGP Routing" checkbox.
    - Set the "Local AS" to a valid private AS number for pfSense (e.g. `64500`).
    - Set the "Router ID" to the gateway IP assigned to pfSense on the LAN network for the Kubernetes nodes (e.g. `192.168.3.1`).
    - Click "Save" at the bottom of the page.
    
    ![Xnapper-2024-01-27-13.09.53.png](/assets/images/Xnapper-2024-01-27-13.09.53.png)
    
    - Under "Advanced", check the box for "Disable eBGP Require Policy".
        - Normally, eBGP requires filter policies to configure trust when ISPs need to peer with other ISPs. This doesn't have to be disabled, but then you would need to configure a policy to allow the routes.
        - Click "Save" at the bottom of the page.
        
        ![Xnapper-2024-01-27-13.17.04.png](/assets/images/Xnapper-2024-01-27-13.17.04.png)
        
- Set up the BGP neighbours (Kubernetes nodes) in a peer group.
    - Go to "Services / FRR / BGP / Edit / Neighbors".
    - Create top-level neighbour that will become the peer group.
        - Set the "Name/Address" for the group (e.g. "rubrik-metallb).
        - Set the "Description" to what the peer group will contain.
        - Set the "Remote AS" to the desired ASN for MetalLB (e.g. "64501").
        - Click "Save" at the bottom of the page.
        
        ![Xnapper-2024-01-27-14.39.19.png](/assets/images/Xnapper-2024-01-27-14.39.19.png)
        
    - Then create a neighbor for each Kubernetes node that MetalLB is running on.
        - Set the "Name/Address" to the IP address of the Kubernetes worker node.
        - Set the "Description" to the hostname of the node.
        - Set the "Peer Group" to the top-level neighbor ("rubrik-metallb" in my case).
        - Set the "Remote AS" to the desired ASN for MetalLB (e.g. "64501").
        - Click "Save" at the bottom of the page.
        
        ![Xnapper-2024-01-27-14.50.17.png](/assets/images/Xnapper-2024-01-27-14.50.17.png)
        

### MetalLB (Kubernetes)

- Install MetalLB onto your Kubernetes cluster.
    - You can find the complete installation instructions here: [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)
    - In my environment, I'm using Kustomize to reference the manifests from the MetalLB repo. And Argo CD to deploy the manifests into the cluster from Git.
    - The Argo CD Application looks like:
        
        ```jsx
        apiVersion: argoproj.io/v1alpha1
        kind: Application
        metadata:
          name: metallb
          namespace: argocd
        spec:
          project: default
          source:
            path: environments/rubrik/system/metallb
            repoURL: 'https://github.com/morey-tech/homelab.git'
            targetRevision: HEAD
          destination:
            namespace: metallb-system
            server: 'https://kubernetes.default.svc'
          ignoreDifferences:
            - group: apiextensions.k8s.io
              jsonPointers:
                - /spec/conversion/webhook/clientConfig/caBundle
              kind: CustomResourceDefinition
          syncPolicy:
            automated: {}
            syncOptions:
              - allowEmpty=true
              - CreateNamespace=true
        ```
        
    - The `kustomization.yaml` referenced by the Application, at `environments/rubrik/system/metallb` in my `morey-tech/homelab` repo, looks like:
        
        ```jsx
        namespace: metallb-system
        resources:
          - github.com/metallb/metallb/config/native?ref=v0.13.12
          - config.yaml
        ```
        
        - It installs version `v0.13.12` of MetalLB.
        - `config.yaml` contains the configuration for MetalLB which I'll cover next.
- Configure MetalLB to advertise the address pool and peer with pfSense.
    - Define the IP address pool.
        - The `addresses` can be any non-overlapping subnet and doesn't have to be related at all to existing subnets. Thanks to the magic of BGP, pfSense will automatically know how to route requests for IP addresses in this subnet to the Kubernetes nodes.
        
        ```yaml
        apiVersion: metallb.io/v1beta1
        kind: IPAddressPool
        metadata:
          name: rubrik-address-pool
          namespace: metallb-system
        spec:
          addresses:
          - 10.8.0.0/16
        ```
        
    - Define the BGP advertisement for the IP address pool.
        
        ```yaml
        apiVersion: metallb.io/v1beta1
        kind: BGPAdvertisement
        metadata:
          name: bgp-advertisement
        spec:
          ipAddressPools:
          - rubrik-address-pool
          # Advertise each route as a /32 (i.e. single IP address).
          aggregationLength: 32
        ```
        
    - Configure MetalLB to peer with pfSense.
        - Set `myASN` to the ASN of the MetalLB (e.g. `64501`).
        - Set `peerASN` to the ASN of pfSense (e.g. `64500`).
        - Set the `peerAddress` to the "Router ID" used in pfSense (e.g. the gateway IP on the lan network for the Kubernetes nodes).
        
        ```yaml
        apiVersion: metallb.io/v1beta2
        kind: BGPPeer
        metadata:
          name: pfsense-peer
        spec:
          myASN: 64501
          # ASN of pfSense.
          peerASN: 64500
          peerAddress: 192.168.3.1
        ```
        

## Put it to the test

- Create a Service for MetalLB to assign an IP address to from the pool.
    - The `metallb.universe.tf/address-pool` annotation is how MetalLB knows to assign an IP address and from what pool.
    
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: guestbook-ui
      namespace: example
      annotations:
        metallb.universe.tf/address-pool: rubrik-address-pool
    spec:
      type: LoadBalancer
      ports:
      - port: 80
        targetPort: 80
      selector:
        app: guestbook-ui
    ```
    
- Once created, the status will update with the IP address assigned by MetalLB
    
    ```yaml
    status:
      loadBalancer:
        ingress:
          - ip: 10.8.0.0
    ```
    
- Back in pfSense you can check the routes created for the service.
    - Go to "Services / FRR / Status / BGP".
    - Under "BGP Routes" it'll show the IP address for the service and the IP addresses of the Kubernetes advertising that service IP.
    
    ![Xnapper-2024-01-27-15.46.35.png](/assets/images/Xnapper-2024-01-27-15.46.35.png)
    

## Conclusion

It's now dead simple to get a routable IP address for any service in my bare-metal Kubernetes cluster. By simply adding the annotation to a Service of `type: LoadBalancer`, MetalLB will assign an IP address from the pool and advertise it using BGP to pfSense. At which point, I can reach the service from outside of the cluster, anywhere else on the network.

![Xnapper-2024-01-27-16.37.24.png](/assets/images/Xnapper-2024-01-27-16.37.24.png)

Referencing IP addresses when trying to access services from Kubernetes is not very user friendly, and not very tolerant to the dynamic nature of the IP assignments. My next step is to automated the create of DNS records for these services using a combination of [`external-dns`](https://github.com/kubernetes-sigs/external-dns) and [`k8s_gateway`](https://github.com/ori-edge/k8s_gateway) to resolve external IPs from outside of Kubernetes.

# References

- [https://github.com/noahburrell0/k8s/blob/main/configs/setup/metallb/configs.yaml](https://github.com/noahburrell0/k8s/blob/main/configs/setup/metallb/configs.yaml)
- [https://docs.netgate.com/tnsr/en/latest/dynamicrouting/bgp/required-info.html](https://docs.netgate.com/tnsr/en/latest/dynamicrouting/bgp/required-info.html)
- [https://www.youtube.com/watch?v=jXG8fuJ-fUI](https://www.youtube.com/watch?v=jXG8fuJ-fUI)
- [https://blog.perf3ct.tech/setting-up-metallb-in-bgp-mode-with-pfsense/](https://blog.perf3ct.tech/setting-up-metallb-in-bgp-mode-with-pfsense/)
- [https://metallb.universe.tf/](https://metallb.universe.tf/)