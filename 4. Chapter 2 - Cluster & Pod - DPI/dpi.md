# In this lab

* [Overview](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/4.%20Chapter%202%20-%20Cluster%20&%20Pod%20-%20DPI/dpi.md#overview)
* [Implement Calico Cloud Deep Packet Inspection](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/4.%20Chapter%202%20-%20Cluster%20&%20Pod%20-%20DPI/dpi.md#implement-calico-cloud-deep-packet-inspection)



### Overview

Security teams need to run DPI quickly in response to unusual network traffic in clusters so they can identify potential threats. Also, it is critical to run DPI on select workloads (not all) to efficiently make use of cluster resources and minimize the impact of false positives. Calico Cloud provides an easy way to perform DPI using Snort community rules. You can disable DPI at any time, selectively configure for namespaces and endpoints, and alerts are generated in the Alerts dashboard in Manager UI.

For each deep packet inspection resource (DeepPacketInspection), Calico Cloud creates a live network monitor that inspects the header and payload information of packets that match the Snort community rules. Whenever malicious activities are suspected, an alert is automatically added to the Alerts page in the Calico Cloud Manager.

Calico Cloud DPI uses AF_PACKET, a Linux socket that allows an application to receive and send raw packets. It is commonly used for troubleshooting (like tcdump and Wireshark), but also for network intrusion detection. For details, see AF_Packet.

After finshing this lab, you should gain a good understanding of how to deploy Calico Cloud Deep Packet Inspection and investigate on malitious activity.

______________________________________________________________________________________________________________________________________________________________________

### Implement Calico Cloud Deep Packet Inspection

1. Edit the FelixConfiguration to add the new field that controls the period, in seconds, at which Felix exports the flow logs to Elasticsearch:
 
```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"flowLogsFlushInterval":"30s"}}'
```

2. Create namespaces `red`and `blue` which we will be using for this test:

```
kubectl create ns red
kubectl create ns blue
```

3. Create a few pods which we'll be using to simulate a suspicious activity:

```
kubectl run -n red red1 --image=wbitt/network-multitool --labels='app=red1'
kubectl run -n red red2 --image=wbitt/network-multitool --labels='app=red2'
kubectl run -n blue blue1 --image=wbitt/network-multitool --labels='app=blue1'
kubectl run -n blue blue2 --image=wbitt/network-multitool --labels='app=blue2'
```

4. Create a YAML file containing the DeepPacketInspection resource, only for the namespace `red` and apply it to the cluster:

```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: DeepPacketInspection
metadata:
  name: red-dpi-all
  namespace: red
spec:
  selector: all()
EOF
```

This will result in Container threat detection running on all nodes in the managed cluster to detect malware and suspicious processes.

5. Adjust the CPU and RAM used for performing deep packet inspection by updating the component resource in IntrusionDetection. The following example configures deep packet inspection to use a maximum of 1 CPU and 1GB RAM:

```
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: IntrusionDetection
metadata:
  name: tigera-secure
spec:
  componentResources:
    - componentName: DeepPacketInspection
      resourceRequirements:
        limits:
          cpu: '1'
          memory: 1Gi
        requests:
          cpu: 100m
          memory: 100Mi
EOF
```

6. Verify that the DPI resource was created:

```
kubectl get deeppacketinspection -n red
```

```
NAME          CREATED AT
red-dpi-all   2023-11-01T17:02:06Z
```

7. Take note of IPs of `blue` and `red` pods:

```
kubectl get pods -A -owide | grep 'red1\|red2\|blue1\|blue2'
```
In my case, this is what I got:

```
blue                         blue1                                                                1/1     Running     0              6s     10.48.116.156   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
blue                         blue2                                                                1/1     Running     0              4s     10.48.116.157   ip-10-0-1-31.ca-central-1.compute.internal   <none>           <none>
red                          red1                                                                 1/1     Running     0              6s     10.48.127.211   ip-10-0-1-30.ca-central-1.compute.internal   <none>           <none>
red                          red2                                                                 1/1     Running     0              6s     10.48.127.212   ip-10-0-1-30.ca-central-1.compute.internal   <none>           <none>
```

8. Generate some traffic which will trigger the DPI resource (make sure to replace IP addresses with the ones you got in your lab):

From red1 to red2 (Command 1)
```
kubectl exec -it -n red red1 -- sh -c "curl http://10.48.127.212:80 -H 'User-Agent: Mozilla/4.0' -XPOST --data-raw 'smk=1234'" > /dev/null 2>&1
```
From red1 to blue1 (Command 2)
```
kubectl exec -it -n red red1 -- sh -c "curl http://10.48.116.156:80 -H 'User-Agent: Mozilla/4.0' -XPOST --data-raw 'smk=1234'" > /dev/null 2>&1
```
From blue2 to red2 (Command 3)
```
kubectl exec -it -n blue blue2 -- sh -c "curl http://10.48.127.212:80 -H 'User-Agent: Mozilla/4.0' -XPOST --data-raw 'smk=1234'" > /dev/null 2>&1
```
From blue2 to blue1 (Command 4)
```
kubectl exec -it -n blue blue2 -- sh -c "curl http://10.48.116.156:80 -H 'User-Agent: Mozilla/4.0' -XPOST --data-raw 'smk=1234'" > /dev/null 2>&1
```

9. Wait 30 seconds and, from `Service Graph > Default`, double-click on the `red` namespace. You should see 4 alerts. Click on each arrow to investigate on these alerts and you will see that:

- `Command 1` generated 2 alerts: 1 for `red1` pod and 1 for `red2` pod (source & destination)
- `Command 2` generated 1 alert for pod `red1` (source)
- `Command 3` generated 1 alert for pod `red2` (destination)
- `Command 4` didn't generate any alert

Play the demo below by clicking on the image:

[![Service Graph Simulation](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/4.%20Chapter%202%20-%20Cluster%20&%20Pod%20-%20DPI/DPI_Service_Graph.png)](https://app.arcade.software/share/a3b3TQCPRQeURNIBplvJ){:target="_blank"}

Alerts are also visible in Activity > Alerts. Play the demo below by clicking on the image:

[![Activity > Alerts Simulation](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/4.%20Chapter%202%20-%20Cluster%20&%20Pod%20-%20DPI/DPI_Alerts.png)](https://app.arcade.software/share/efXWMLTRGkRdqWCTVdTB){:target="_blank"}

10. Clean up the resources that were deployed for the purpose of this lab.

```
kubectl delete ns red
kubectl delete ns blue
```

> **Congratulations! You have completed `4. Chapter 2 - Cluster & Pod - DPI` lab.**