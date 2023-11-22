# In this lab

* [Overview](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/8.%20Chapter%203%20-%20Runtime%20-%20Threat%20Feed%20%26%20DGA/threat_feed_dga.md#overview)
* [Implement Calico Cloud Threat Feed](https://github.com/tigera-cs/Kubernetes-and-Container-Security-Instructor-Led-Workshop/blob/main/8.%20Chapter%203%20-%20Runtime%20-%20Threat%20Feed%20%26%20DGA/threat_feed_dga.md#implement-calico-cloud-threat-feed)



### Overview

Calico Cloud Image Assurances helps you identify vulnerabilities in container images that you deploy to Kubernetes clusters. Vulnerabilities are known flaws in libraries and packages used by applications that attackers can exploit and cause harm.

Image Assurance is based on the Common Vulnerabilities and Exposures (CVE) system, which provides a catalog of publicly-known security vulnerabilities and exposures. Known vulnerabilities are identified by a unique CVE ID based on the year it was reported (for example, CVE-2021-44228).

When you enable Image Assurance, the daemonSet "tigera-image-assurance-crawdad" is created in the "tigera-image-assurance" namespace

Scanned image content includes:
- Libraries and content (for example, python, ruby gems, jars and go)
- Packages (OS and non-OS)
- Image layer

In this lab, we will enable Image Assurance, scan images automatically and manually, and investigate on scanned and running images.
______________________________________________________________________________________________________________________________________________________________________

### Implement Calico Cloud Image Assurance


kubectl edit imageassurances default

Set the clusterScanner field to Enabled and save the file.

The cluster scanner is deployed as a container inside the tigera-image-assurance-crawdad daemonset.

Verify that a new container with name, cluster-scanner is created inside the daemonset.

CVE-2021-44228

kubectl run log4j --image  quay.io/jsabo/log4shell-vulnerable-app

That’s it. Images will start scanning. For help, see Understand scan results in Manager UI.

Manual scanning

sudo curl -Lo tigera-scanner https://installer.calicocloud.io/tigera-scanner/v3.18.0-1.1-1/image-assurance-scanner-cli-linux-amd64

sudo chmod +x ./tigera-scanner

sudo docker pull quay.io/jsabo/log4shell-vulnerable-app:latest

sudo ./tigera-scanner scan quay.io/jsabo/log4shell-vulnerable-app:latest

sudo ./tigera-scanner scan quay.io/jsabo/log4shell-vulnerable-app:latest --apiurl .... --token ....

> **Congratulations! You have completed `6. Chapter 2 - Custer & Pod - Calico Cloud Image Assurance` lab.**