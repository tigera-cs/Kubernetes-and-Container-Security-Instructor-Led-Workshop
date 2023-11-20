# In this lab

* [Overview](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/8.%20Chapter%203%20-%20Runtime%20-%20Threat%20Feed%20%26%20DGA/threat_feed_dga.md#overview)
* [Implement Calico Cloud Threat Feed](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/8.%20Chapter%203%20-%20Runtime%20-%20Threat%20Feed%20%26%20DGA/threat_feed_dga.md#implement-calico-cloud-threat-feed)



### Overview

Firewalls often detect and block traffic associated with known bad actors, but don’t have granular visibility into which pod is infected. Calico’s threat feed can pinpoint and report on the exact source of malicious traffic.

Calico Cloud ingests threat feeds that identify IP addresses for known bad actors, such as botnets. Any traffic to those IPs is automatically blocked and generates an alert. All the workloads in the cluster will be protected with a block-alienvault-ipthreatfeed security policy, which is enabled to block egress and ingress traffic.

Calico’s threat feed deployment consists of 3 main components: a threat feed resource, a network set resource, and a global network policy.

1. The threat feed resource pulls updates automatically on a daily basis. The threat feed(s) must be available using HTTP(S), and return a newline-separated list of IP addresses or prefixes in CIDR notation.
2. A network set resource (NetworkSet) represents an arbitrary set of IP subnetworks/CIDRs, and gets updated periodically by the threat feeds. The metadata for the network set includes a set of labels. These labels are used to create the network policy to block the traffic.
3. A global network policy blocks traffic to any of the suspicious IPs. Notice that the suspicious IPs are not defined in the network policy manifest; instead, Calico uses a selector (in our case 'feed == "training-ip-threatfeed"'), which refers to the NetworkSet.

______________________________________________________________________________________________________________________________________________________________________

### Implement Calico Cloud Threat Feed

1. As the first step, we turn down the aggregation of flow logs sent to Elasticsearch to get more useful results from threat feed searches:

```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"flowLogsFileAggregationKindForAllowed":1}}'
```

2. Now, to configure Threat Feed, we need to deploy the GlobalThreatFeed resource for an IP list, which will also create a GlobalNetworkSet for the IPs included in the list. The IP list that we are going to use for this test is 'https://installer.calicocloud.io/feeds/v1/ips' and we can deploy the 2 resources, with this command:

```
cat << EOF | kubectl apply -f -
kind: GlobalThreatFeed
apiVersion: projectcalico.org/v3
metadata:
  name: training.ip.threatfeed
spec:
  content: IPSet
  mode: Enabled
  description: AlienVault IP Block List
  feedType: Builtin
  globalNetworkSet:
    labels:
      feed: training-ip-threatfeed
  pull:
    http:
      format: {}
      url: 'https://installer.calicocloud.io/feeds/v1/ips'
EOF
```

NOTE: 

This is the part of the manifest which creates the Global Network Set from the IP list of the Global Threat Feed resource:
```
  globalNetworkSet:
    labels:
      feed: training-ip-threatfeed
```

3. Let's verify from Calico Cloud UI that the Global Threat Feed list and the Global Network Set were created.

- Global Threat Feed list: Threat Defense > Threat Feeds:

![global-threat-feed-list](img/1.global-threat-feed-list.gif)

- Global Network Set: Network Sets:

![global-network-set](img/2.global-network-set.gif)

4. The GlobalThreatFeed resource will only detect suspicoius activity and generate alerts. To block these attempts, we need to create a Global Network Policy which will refers to the Global Network Set, using the label 'feed == 'training-ip-threatfeed'. You can deploy the 'tigera-security' tier and the 'block-alienvault-ipthreatfeed' policy using this command:

```
cat << EOF | kubectl apply -f -
---
apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: tigera-security
spec:
  order: 300
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
    name: tigera-security.block-training-ipthreatfeed
spec:
  tier: tigera-security
  selector: all()
  namespaceSelector: ''
  serviceAccountSelector: ''
  egress:
  - action: Deny
    source: {}
    destination:
      selector: feed == "training-ip-threatfeed"
  - action: Pass
    source: {}
    destination: {}
  types:
  - Egress
EOF
```
kubectl create ns blue
kubectl create ns red
kubectl create ns yellow
kubectl create ns brown
kubectl run -n blue blue --image=wbitt/network-multitool --labels=app=blue
kubectl run -n red red --image=wbitt/network-multitool --labels=app=red
kubectl run -n yellow yellow --image=wbitt/network-multitool --labels=app=blue
kubectl run -n brown brown --image=wbitt/network-multitool --labels=app=red

while read -r ip; do timeout 1 curl -I "http://$ip"; done < list


> **Congratulations! You have completed `8. Chapter 1 - Perimeter - Calico Enterprise Threat Feed` lab.**