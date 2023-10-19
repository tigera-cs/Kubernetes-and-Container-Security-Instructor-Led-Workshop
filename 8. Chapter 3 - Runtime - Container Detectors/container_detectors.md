# In this lab

* [Overview](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/tree/main/3.%20Chapter%201%20-%20Perimeter%20-%20Egress%20Gateway#overview)
* [Implement Calico Enterprise Egress Gateway](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/tree/main/3.%20Chapter%201%20-%20Perimeter%20-%20Egress%20Gateway#implement-calico-enterprise-egress-gateway)



### Overview

In Kubernetes, Ingress traffic refers to any traffic that is initiated from outside the cluster to the services running inside the Kubernetes cluster. Egress traffic is just the opposite, any traffic that is initiated from the pods from within the cluster to the IP endpoints that are located outside the cluster. Kubernetes provides a native ingress resource to manage and control Ingress traffic. However, there is no native Kubernetes Egress resource. So when pods need to connect to an endpoint outside the cluster, they do so using their own IP addresses by default. Considering that an application could be implemented through one or more pods and the fact that pods in Kubernetes are ephemeral, it is almost impossible to identify and control the Kubernetes egress traffic from outside the cluster as the IP addresses are constantly changing. Egress gateways (EGWs) are pods that act as gateways for traffic leaving the cluster from certain client pods. The primary function of egress gateway is to configure the egress gateway client to have a particular and persistent source IP address when connecting to services outside the Kubernetes cluster.

To simplify the installation of th lab, we will be using the bastion instead of a pfSense, for the BGP peering and Egress Gateway configuration.


After finshing this lab, you should gain a good understanding of how to deploy Calico Enterprise Egress Gateway and establish a permanent identity for the traffic that is leaving the cluster.

______________________________________________________________________________________________________________________________________________________________________

### Implement Calico Enterprise Egress Gateway

1. Deploy the needed BGPConfiguration and BGPPeer so we route our traffic to the `bastion` host through the egress gateway.

```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  nodeMeshMaxRestartTime: 120s
  asNumber: 64512
  serviceClusterIPs:
    - cidr: 10.49.0.0/16
  listenPort: 179
  bindMode: NodeIP
  communities:
  - name: bgp-large-community
    value: 64512:120
  prefixAdvertisements:
    - cidr: 10.10.10.0/31
      communities:
        - bgp-large-community
        - 64512:120
---
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: my-global-peer
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
EOF

```

2. Bastion host is simulating our upstream router/firewall and we should have BGP sessions established to all the cluster nodes using the above configurations. Run the following command on the bastion node to validate the bgp sessions from the bastion node to the cluster nodes.

```
watch sudo birdc show protocols
```

Wait for all sessions to be established.

```
BIRD 1.6.8 ready.
name     proto    table    state  since       info
direct1  Direct   master   up     23:42:33    
kernel1  Kernel   master   up     23:42:33    
device1  Device   master   up     23:42:33    
control1 BGP      master   up     23:42:35    Established   
worker1  BGP      master   up     23:42:36    Established   
worker2  BGP      master   up     23:42:35    Established
```
We must also set the bird config on the bastion to accept advertisements of the egress gateway network.

```
sudo vi /etc/bird/bird.conf
```
In this file add the 10.10.10.0/31 network here:

```
# Import filter
filter rt_import {
                        if (net ~ 10.48.2.0/24) then accept;
                        if (net ~ 10.49.0.0/16) then accept;
                        if (net ~ 10.50.0.0/24) then accept;
                        if (net ~ 10.10.10.0/31) then accept; <-------
                        reject;
        }
```
And restart bird

```
sudo systemctl restart bird
```

3. Enable egress gateway support by patching FelixConfiguration to support egress gateway both per namespace and per pod.

```
kubectl patch felixconfiguration.p default --type='merge' -p '{"spec":{"egressIPSupport":"EnabledPerNamespaceOrPerPod"}}'
    
```
4. Egress gateways require the Policy Sync API to be enabled in `felixconfiguration` to implement symmetric routing. Run the following command to enable this configuration cluster-wide.


28. Clean up the resources that were deployed for the purpose of this lab.

```
```
kubectl delete deployments egress-gateway

```

> **Congratulations! You have completed `8. Chapter 3 - Runtime - Container Detectors` lab.**