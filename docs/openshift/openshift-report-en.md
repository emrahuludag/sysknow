
![](./img/Ansible-and-Openshift.png? ':size=50%')

# Automated OpenShift Cluster Health Report

Automation has become a fundamental way to enhance efficiency, reduce human errors, and accelerate processes within IT infrastructure today. By automating repetitive tasks and complex workflows, operations can run smoothly without manual intervention, allowing IT teams to focus on more strategic, value-generating work. Furthermore, automation provides scalability and consistency essential for keeping up with rapidly growing digital demands, ensuring the security and continuity of systems.

For OpenShift environments located across different remote locations, the ansible automation solution I developed offers a centralized way to monitor and report on the health status of all clusters regularly. This automation immediately sends notifications via Microsoft Teams and email when an issue is detected, enabling infra and DevOps teams to respond quickly. Consequently, the reliability and performance of OpenShift infrastructure across multiple locations are monitored and managed more effectively, delivering greater resilience and efficiency.

> [!NOTE|style:flat]
> This playbook tested with 4.11.x, 4.12.x, 4.13.x and 4.14.38 Openshift (IPI) version.
> Some modules may work for UPI clusters.

### Features

* Generates a stand-alone HTML file for OpenShift cluster health
* Requires oc, jq, and python-jmespath packages on Ansible nodes
* Light-weight and provides table filtering functionality, allowing users to filter and view specific data within tables
* OC commands are run directly on the Ansible node
* Modular design, making it easy to add new components
* Available Modules:
 	* oc-logout: Logs out from OpenShift to ensure session security
	* get-token: Retrieves the access token for API requests
	* get-nodes: Collects node information, including health and resource usage
	* get-co: Gathers ClusterOperator status for component health
	* get-resource: Checks available and used resources across nodes
	* get-node-conditions: Reports on node conditions and possible memory, cpu, kubelet services issues
	* get-pods-distribution: Displays distribution of pods across nodes
	* get-etcd-stats: Collects ETCD pods status
	* get-update-status: Checks for cluster update progress and status
	* get-deployments: Gathers deployment statuses in namespaces. Single running pod are highlighted
	* get-ssl: Validates SSL certificates and expiration dates
	* get-alert: Retrieves active alerts within the cluster
	* get-pod-health: Checks health and status of individual pods
	* get-overcommitted: Get overcommitted  node status
	* get-pv: Monitors Persistent Volume status and availability
	* get-pod-status: Collects overall pod status across the cluster
	* get-pod-usage: Checks resource usage metrics of pods
	* get-ns: Lists namespaces and their statuses
	* get-route: Route lists
	* get-total-resource: Summary of total cluster resources
	* get-operators: Cluster Operator statuses and health
	* get-users-group: Lists users and groups
	* get-machine: Gathers machine information
	* get-daemonset: Checks DaemonSet statuses
	* get-apps: Retrieves application statuses and configurations
	* send-mail: Sends the generated report via email.

### How to use
Install packages

```bash
yum install python3-jmespath jq
```

Install collections   community.general, ansible.posix, ansible.utils


```bash
ansible-galaxy install -r collections/requirements.yml
```

Clone or copy this repository on the ansible node

```bash
git clone https://github.com/emrahuludag/openshift-report.git
```

Edit variables;

```bash
vi ./vars/main.yaml 
```

```bash
site_name: "TRIST"
cluster_name: "TR-DEV"

#
# Authorized user
#
ocp_user: "user"
ocp_pass: "pass"
ocp_url: "https://api.exampledomain.local:6443"

#
# Mail Relay Configuration
#
mail_host: "1.1.1.1"
mail_port: "25"
mail_sender: "cluster-report@exampledomain.local (Cluster Overview Automation)"
mail_to: "emrahuludag@gmail.com, devops@exampledomain.local"

#
# Console and App UI Certificate Checks
#
web_console: "console-openshift-console.apps.exampledomain.local"
app_ui: "console-openshift-console.apps.exampledomain.local"

```

And run the ocp4-report.yml playbook.

```bash
ansible-playbook ocp4-report.yml
```


### Schedule Report


Create script daily.sh

```bash
#!/bin/sh
/usr/bin/ansible-playbook /openshift-report/ocp4_report.yml
```

Add cron job 
Script will run 07:00 AM 

```bash
0 7 * * * /openshift-report/daily.sh
```


### Sample Report

Right click to save [OpenShift Full Sample Report](https://github.com/emrahuludag/sysknow/raw/main/docs/openshift/img/openshift-sample-report.html)

![](./img/ocp-report-sample01.png? ':size=80%')
![](./img/ocp-report-sample02.png? ':size=80%')
![](./img/ocp-report-sample03.png? ':size=80%')
![](./img/ocp-report-sample04.png? ':size=80%')
![](./img/ocp-report-sample05.png? ':size=80%')

![](./img/ocp-report-sample06.png? ':size=80%')

![](./img/ocp-report-sample07.png? ':size=20%')



