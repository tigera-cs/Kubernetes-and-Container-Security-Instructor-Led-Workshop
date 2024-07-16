# In this lab

* [Overview](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/waf.md#overview)
* [Implement Calico Cloud East-West web Application Firewall](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/waf.md#implement-calico-cloud-east-west-web-application-firewall)



### Overview

Calico workload-centric Web Application Firewall (WAF) protects workloads from a variety of application layer attacks originating from within the cluster such as SQL injection. Given that attacks on apps are the leading cause of breaches, organizations need to secure the HTTP traffic inside their cluster.

Historically, web application firewalls (WAFs) were deployed at the edge of your cluster to filter incoming traffic. Calico workload-based WAF solution takes a unique, cloud-native approach to web security by allowing companies to implement zero-trust rules for workloads inside their cluster.

WAF is deployed in the cluster along with Envoy DaemonSet. Calico Cloud proxies selected service traffic through Envoy, checking HTTP requests using Coraza with OWASP CoreRuleSet v4.X modified for kubernetes workloads. To review the rules deployed with the WAF, see [Ruleset files](https://github.com/coreruleset/coreruleset).

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
_____________________________________

2. Deploy sample apps and a pod that we will use to simulate `a lateral` `HTTP` attacks:

```
kubectl apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/release/v0.7.0/release/kubernetes-manifests.yaml
kubectl create ns red
kubectl run -n red red --image=wbitt/network-multitool
```
_____________________________________

3. It's time to enable WAF from UI `Threat Defence > Web Application Firewall > Configure Web Application Firewall` or from CLI:

- From UI: In `Threat Defence > Web Application Firewall > Configure Web Application Firewall`, select `cartservice` and `frontend`. Watch the GIF or play the demo below by clicking on the image:

[![WAF Enable](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/img/WAF_Enable.gif)](https://app.arcade.software/share/gDb3I8lwd7fLYA2o4sWw)

- From CLI:
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

**NOTE**: *To fully enable WAF from UI, you must select the service/services you want to protect and then click `Confirm Selection`. In this lab, we are going to enable WAF for the `Frontend` service.*
_____________________________________

4. Get the IP of the `Frontend` service, which we will use as a target, and generate some traffic:

```
FRONTEND=$(kubectl get svc frontend -o jsonpath='{.spec.clusterIP}')
echo $FRONTEND
```

```
kubectl exec -it -n red red -- curl -I http://frontend.default.svc.cluster.local 
```

```
kubectl exec -it -n red red -- curl -I http://$FRONTEND/
```
```
kubectl exec -it -n red red -- curl -I http://frontend.default/cart?artist=0+div+1+union%23foo*%2F*bar%0D%0Aselect%23foo%0D%0A1%2C2%2Ccurrent_user
```
_____________________________________

5. All these 3 commands should return this output:

```
HTTP/1.1 200 OK
Set-Cookie: shop_session-id=e88f8cb8-63ba-460e-bde4-55d508133912; Max-Age=172800
Date: Mon, 04 Dec 2023 16:03:23 GMT
Content-Type: text/html; charset=utf-8
```

However, while the first one will not generate any Security Event, the other 2 will:

- `kubectl exec -it -n red red -- curl -I http://$FRONTEND/` will trigger the `Host header is a numeric IP address` `ID`
- `kubectl exec -it -n red red -- curl -I http://frontend.default/cart?artist=0+div+1+union%23foo*%2F*bar%0D%0Aselect%23foo%0D%0A1%2C2%2Ccurrent_user` will trigger 2 different `IDs`:
    - `SQL Injection Attack Detected via libinjection`
    - `Remote Command Execution: Windows Command Injection`

Check the events and alerts generated for this traffic:

**Service Graph**

[![WAF Service Graph](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/img/WAF_Service_Graph.gif)](https://app.arcade.software/share/2u2AjfSzqE63hUAhnSAL)

**Security Events**

[![WAF Security Events](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/img/WAF_Security_Events.gif)](https://app.arcade.software/share/RzyuxjF5PSTWH5g9CL3D)

**Activity > Alerts**

[![WAF Activity Alerts](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/img/WAF_Activity_Alerts.gif)](https://app.arcade.software/share/RjHInEkLPoMG9P58FOGq)
_____________________________________

6. By default, WAF will not block a request even if it has matching rule violations. The rule engine is set to `DetectionOnly`. You can configure to block traffic instead with an `HTTP 403 Forbidden` response status code when the combined matched rules scores exceed a certain threshold.

To do so:
- Edit the WAF ConfigMap:
```
kubectl edit cm -n tigera-operator modsecurity-ruleset
```
- Search for `SecRuleEngine` using `/`
- Press `i` to edit the resource
- Change it from `DetectionOnly` to `On`, as shown below:

**BEFORE**

![WAF Before](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/img/WAF_before.png)

**AFTER**

![WAF Before](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/img/WAF_after.png)

- Press `ESC`, then type `:wq!` and press `ENTER` to save the configuration.
_____________________________________

7. Repeat step 4.

You will see that, while the first 2 commands are still returning `200 OK` the third one is returning `403 Forbidden`:

```
HTTP/1.1 403 Forbidden
date: Mon, 04 Dec 2023 16:45:45 GMT
server: envoy
transfer-encoding: chunked
```

Basically:
- The first command was and still is legitimate
- the second command is suspicious, but the score is not reaching the threshold
- The third command is exeeding the threshold, triggering the block from the WAF rule.
_____________________________________

8. Check again `Security Events` and `Activity > Alerts` and you hould see that the WAF is now blocking the simulated SQL Injection attack:

![WAF Block](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/img/WAF_block.png)
_____________________________________

9. **OPTIONAL**: Change the default score threshold

- Edit the WAF ConfigMap:
```
kubectl edit cm -n tigera-operator modsecurity-ruleset
```

- Change the default score threshold to 1 by adding this to the configuration, between `t:none,\` and `setvar:'tx.allowed_request_content_type=|appli...`:

```
        setvar:tx.inbound_anomaly_score_threshold=1,\
        setvar:tx.outbound_anomaly_score_threshold=20,\
```

**BEFORE**

```
    SecAction \
        "id:900220,\
        phase:1,\
        nolog,\
        pass,\
        t:none,\
        setvar:'tx.allowed_request_content_type=|application/x-www-form-urlencoded| |multipart/form-data| |multipart/related| |text/xml| |application/xml| |application/soap+xml| |application/json| |application/cloudevents+json| |application/cloudevents-batch+json| |application/grpc| |application/grpc+proto| |application/grpc+json| |application/octet-stream|'"
```

**AFTER**

```
    SecAction \
        "id:900220,\
        phase:1,\
        nolog,\
        pass,\
        t:none,\
        setvar:tx.inbound_anomaly_score_threshold=1,\
        setvar:tx.outbound_anomaly_score_threshold=20,\
        setvar:'tx.allowed_request_content_type=|application/x-www-form-urlencoded| |multipart/form-data| |multipart/related| |text/xml| |application/xml| |application/soap+xml| |application/json| |application/cloudevents+json| |application/cloudevents-batch+json| |application/grpc| |application/grpc+proto| |application/grpc+json| |application/octet-stream|'"
```

![WAF Before Score](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/5.%20Chapter%202%20-%20Cluster%20%26%20Pod%20-%20WAF/img/WAF_after_score.png)

- Press `ESC`, then type `:wq!` and press `ENTER` to save the configuration.

10. Repeat step 4.

You will see that, also the second command is returning `403 Forbidden`:

```
HTTP/1.1 403 Forbidden
date: Mon, 04 Dec 2023 16:45:45 GMT
server: envoy
transfer-encoding: chunked
```

Basically:
- The first command was and still is legitimate.
- the second command is suspicious, but the score is exceeding the new threshold, triggering the block from the WAF rule.
- The third command is exeeding the threshold, triggering the block from the WAF rule.

11. Finally, clean up the resources that were deployed for the purpose of this lab.

```
kubectl delete applicationlayers.operator.tigera.io tigera-secure
```

<!--
kubectl apply -f - <<EOF
apiVersion: operator.tigera.io/v1
kind: ApplicationLayer
metadata:
  name: tigera-secure
spec:
  webApplicationFirewall: Disabled
EOF
-->

```
kubectl delete -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/release/v0.7.0/release/kubernetes-manifests.yaml
```

```
kubectl delete ns red
```
_____________________________________

> **Congratulations! You have completed `5. Chapter 2 - Cluster & Pod - WAF` lab.**
