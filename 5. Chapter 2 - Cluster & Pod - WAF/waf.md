# In this lab

* [Overview](LINK)
* [Implement Calico Cloud East-West web Application Firewall](LINK)



### Overview

Calico workload-centric Web Application Firewall (WAF) protects workloads from a variety of application layer attacks originating from within the cluster such as SQL injection. Given that attacks on apps are the leading cause of breaches, organizations need to secure the HTTP traffic inside their cluster.

Historically, web application firewalls (WAFs) were deployed at the edge of your cluster to filter incoming traffic. Calico workload-based WAF solution takes a unique, cloud-native approach to web security by allowing companies to implement zero-trust rules for workloads inside their cluster.

WAF is deployed in the cluster along with Envoy DaemonSet. Calico Cloud proxies selected service traffic through Envoy, checking HTTP requests using the industry-standard ModSecurity with OWASP CoreRuleSet v3.3.5 modified for kubernetes workloads. To review the rules deployed with the WAF, see [Ruleset files](https://github.com/tigera/operator/tree/master/pkg/render/applicationlayer/modsec-core-ruleset).

You simply enable WAF in Manager UI, and determine the services that you want to enable for WAF protection. By default WAF is set to DetectionOnly so no traffic will be denied until you are ready to turn on blocking mode.

Every request that WAF finds an issue with, will result in a Security Event being created for you to review in the UI, regardless of whether the traffic was allowed or denied. This can greatly help in tuning later.

If you configure WAF in blocking mode, WAF will use something called anomaly scoring mode to determine if a request is allowed with 200 OK or denied 403 Forbidden.

This works by matching a single HTTP request against all the configured WAF rules. Each rule has a score and WAF adds all the matched rule scores together, and compares it to the overall anomaly threshold score (100 by default). If the score is under the threshold the request is allowed and if the score is over the threshold the request is denied. Our WAF starts in detection mode only and with a high default scoring threshold so is safe to turn on and then fine-tune the WAF for your specific needs in your cluster.

After finishing this lab, you should gain a good understanding of how to deploy Calico Cloud WAF and investigate on malitious activity.

______________________________________________________________________________________________________________________________________________________________________

### Implement Calico Cloud East-West web Application Firewall

1. Edit the FelixConfiguration to add the new field that controls the period, in seconds, at which Felix exports the flow logs to Elasticsearch:
 
```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"flowLogsFlushInterval":"30s"}}'
```

2. Deploy sample apps and a pod that we will use to simulate `east-west` `HTTP` attacks:

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/release/v0.7.0/release/kubernetes-manifests.yaml
kubectl create ns red
kubectl run -n red red --image=wbitt/network-multitool
```

This will result in Container threat detection running on all nodes in the managed cluster to detect malware and suspicious processes.


3. It's time to enable WAF from UI `Threat Defence > Web Application Firewall > Configure Web Application Firewall` or from CLI:

```
kubectl patch felixconfiguration default --type='merge' -p '{"spec":{"policySyncPathPrefix":"/var/run/nodeagent"}}'
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: ApplicationLayer
metadata:
  name: tigera-secure
spec:
  webApplicationFirewall: Enabled
EOF
kubectl annotate svc frontend -n default --overwrite projectcalico.org/l7-logging=true
```

**NOTE**: *To fully enable WAF from UI, you need to select the service/services you want to protect and then click `Confirm Selection`. In this lab, we are going to enable WAF for the `Frontend` service.*



6. Finally, clean up the resources that were deployed for the purpose of this lab.



> **Congratulations! You have completed `5. Chapter 2 - Cluster & Pod - WAF` lab.**
