# In this lab

* [Overview](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/7.%20Chapter%203%20-%20Runtime%20-%20Container%20Detectors/container_detectors.md#overview)
* [Implement Calico Cloud Container Threat Detection](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/7.%20Chapter%203%20-%20Runtime%20-%20Container%20Detectors/container_detectors.md#implement-calico-cloud-container-threat-detection)



### Overview

Calico Cloud provides a threat detection engine that analyzes observed file and process activity to detect known malicious and suspicious activity.

As part of these threat detection capabilities, Calico Cloud maintains a database of malware file hashes. This database consists of SHA256, SHA1, and MD5 hashes of executable file contents that are known to be malicious. Whenever a program is launched in a Calico Cloud cluster, malware detection generates an alert in the Alerts dashboard if the program's hash matches one that is known to be malicious.

Our threat detection engine also monitors activity within the containers running in your clusters to detect suspicious behavior and generate corresponding alerts. The threat detection engine monitors the following types of suspicious activity within containers:

Access to sensitive system files and directories
Defense evasion
Discovery
Execution
Persistence
Privilege escalation


After finishing this lab, you should gain a good understanding of how to deploy Calico Cloud Container Threat Detection and investigate on malitious activity.

______________________________________________________________________________________________________________________________________________________________________

### Implement Calico Cloud Container Threat Detection

1. Edit the FelixConfiguration to add the new field that controls the period, in seconds, at which Felix exports the flow logs to Elasticsearch:
 
```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"flowLogsFlushInterval":"30s"}}'
```

2. Container threat detection is disabled by default. Let's enable it using kubectl:

```
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: RuntimeSecurity
metadata:
  name: default
EOF
```

This will result in Container threat detection running on all nodes in the managed cluster to detect malware and suspicious processes.


3. If a malicious or suspicious program is run within the cluster, it will be reported on the Alerts page of the Calico Cloud UI. We can simulate attacks deploying these 2 pods:

- Pod "outbound-connection-to-miner", which simulates outbound connections to miner pools on common ports:

```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: outbound-connection-to-miner
  labels:
    app: ubuntu
spec:
  containers:
  - image: ghcr.io/lightspin-tech/light-k8s-attack-simulations/k8s-attack-simulation:latest
    command: ["src/shell-outbound-connection-to-miner.sh"]
    imagePullPolicy: Always
    name: simulation
EOF
```

- Pod "evil-pod", which simulates a malware pod:

```
kubectl run evil-pod --image quay.io/tigera/runtime-security-test
```

4. Target the endpoint to generate malware activity:

```
kubectl exec -it pod/evil-pod -- curl -XPOST localhost/bad
```
5. Let's wait a minute and then check Service Graph. A red alert is shown on the Default namespace and, double-clicking on it, we can see details about the attacks. Play the demo below by clicking on the image:

[![Service Graph Simulation](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/7.%20Chapter%203%20-%20Runtime%20-%20Container%20Detectors/Screenshot%202023-10-20%20at%2009.36.40.png)](https://app.arcade.software/share/CD8sOEot4tj3bZXUzsVd)

Alerts are also visible in Activity > Alerts. Play the demo below by clicking on the image:

[![Activity > Alerts Simulation](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/7.%20Chapter%203%20-%20Runtime%20-%20Container%20Detectors/Alerts.png)](https://app.arcade.software/share/6mmVE7eGn48uBzffAom6)

In Threat Defense > Security Events, you get a more user-friendly overview of the malitious event. Play the demo below by clicking on the image:

[![Security Events Simulation](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/7.%20Chapter%203%20-%20Runtime%20-%20Container%20Detectors/Security_events.png)](https://app.arcade.software/share/Eeg8MUG9Xzb8YKQldTtu)

6. Clean up the resources that were deployed for the purpose of this lab.

```
kubectl delete pod outbound-connection-to-miner
```

```
kubectl delete pod evil-pod
```

> **Congratulations! You have completed `7. Chapter 3 - Runtime - Container Detectors` lab.**
