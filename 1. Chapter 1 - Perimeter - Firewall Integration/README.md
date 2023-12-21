# In this lab

### Overview

The traditional security solutions like firewalls often lack native awareness of Kubernetes workloads. They were not originally designed to comprehend or manage the dynamic and diverse nature of containerized environments.

Calico addresses this limitation by integrating seamlessly with traditional security solutions like Fortinet and AWS Security Groups. This integration provides a sophisticated and fine-grained control mechanism that allows for enhanced security and management capabilities.

With Calico integrated into Fortinet and AWS Security Groups, it becomes possible to exert precise control over access to AWS resources. Additionally, this integration enables Fortinet to gain visibility into Kubernetes workloads. As a result, Fortinet becomes capable of managing the egress traffic from individual workloads, granting greater control and oversight.

One of the notable advantages is the ability to create and enforce Calico Security Policies directly through the Fortimanager interface. This streamlined approach simplifies policy creation and management, offering an intuitive interface for defining and implementing security policies across the Kubernetes environment.

### Extend Kubernetes to Fortinet firewall devices - Fortigate Integration

<a href="https://drive.google.com/file/d/1NDphqsYbhhLo2ojYITA07GlisbzkCWbk/view" target="_blank">Demo</a>

**Yaml files for reference:**

Config map to define the integration tier and the address selection:
```
kubectl -n tigera-firewall-controller create configmap  tigera-firewall-controller \
--from-literal=tigera.firewall.policy.selector="projectcalico.org/tier == '<INTEGRATION_TIER>'" \
--from-literal=tigera.firewall.addressSelection="<POD|NODE>"
```

Config map to add the Fortigate/Fortimanager information:
```
kubectl apply -f - <<EOF
kind: ConfigMap
apiVersion: v1
metadata:
  name: tigera-firewall-controller-configs
  namespace: tigera-firewall-controller
data:
  # FortiGate device information
  tigera.firewall.fortigate: |
    - name: <FORTIGATE_NAME>
      ip: <FORTIGATE_ip>
      apikey: 
        secretKeyRef:
          name: <SECRET_NAME>
          key: <SECRET_key>
  tigera.firewall.fortimgr: |
EOF
```

Secret to store the Fortigate user's API key:
```
kubectl create secret generic <SECRET_NAME> \
-n tigera-firewall-controller \
--from-literal=<SECRET_KEY>=<FORTIGATE_USER_API_KEY>
```

Secret to tigera-firewall-controller pull the images from Tigera:
```
kubectl create secret generic tigera-pull-secret \
--from-file=.dockerconfigjson=<CONFIG.JSON> \
--type=kubernetes.io/dockerconfigjson -n tigera-firewall-controller
```


### Extend FortiManager firewall policies to Kubernetes - Fortimanager Integration

<a href="https://drive.google.com/file/d/1NH0ySu62rjCf_j9LfKcZMZDarsdAB_Li/view" target="_blank">Demo</a>

**Yaml files for reference:**

Tier creation:
```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: Tier
metadata:
  name: fortinet
spec:
  order: 200
EOF
```

Tigera Fortimanager integration config map:
```
kubectl apply -f - <<EOF
# Configuration of Tigera Fortimanager Integration Controller
kind: ConfigMap
apiVersion: v1
metadata:
  name: tigera-fortimanager-controller-configs
  namespace: tigera-firewall-controller
data:
  tigera.firewall.fortimanager-policies: |
    - name: <FORTIMANAGER_NAME>
      ip: <FORTIMANAGER_IP>
      username: <FORTIMANAGER_USER>
      adom: root
      packagename: <FORTIMANAGER_PACKAGE_NAME>
      tier: <CALICO_INTEGRATION_TIER>
      password:
        secretKeyRef:
          name: <SECRET_NAME>
          key: <SECRET_KEY>
EOF
```

Secret to store the Fortimanager user's password:
```
kubectl create secret generic <SECRET_NAME> \
-n tigera-firewall-controller \
--from-literal=<SECRET_KEY>=<FORTIMANAGER_USER_PASSWORD>
```

Secret to tigera-firewall-controller pull the images from Tigera:
```
kubectl create secret generic tigera-pull-secret \
--from-file=.dockerconfigjson=<CONFIG.JSON> \
--type=kubernetes.io/dockerconfigjson -n tigera-firewall-controller
```

Global Network Set creation:
```
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  labels:
    tigera.io/address-group: allow-google
  name: gnp-multitool2-google
spec:
  allowedEgressDomains:
  - '*.google.com'
EOF
```

### AWS Security Group Integration

