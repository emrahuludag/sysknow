
![](./img/Ansible-and-Openshift.png? ':size=50%')

# Automated Health Report for OpenShift Cluster with Ansible

This Ansible playbook generates a single HTML report to monitor the health status of an OpenShift (IPI) cluster. The playbook checks the status of key components within the cluster, providing system administrators with a quick overview of the cluster's health. This allows administrators to detect and address potential issues in the OpenShift environment promptly.

### Features

* Generates a stand-alone HTML file for OpenShift cluster health
* Requires oc, jq, and python-jmespath packages on Ansible nodes
* Light-weight and provides table filtering functionality, allowing users to filter and view specific data within tables
* OC commands are run directly on the cluster, and data processing on the Ansible node
* Modular design, making it easy to add new report components
* Available Modules:
 	* oc-logout: Logs out from OpenShift to ensure session security
	* get-token: Retrieves the access token for API requests
	* get-nodes: Collects node information, including health and resource usage
	* get-co: Gathers ClusterOperator status for component health
	* get-resource: Checks available and used resources across nodes
	* get-node-conditions: Reports on node conditions and possible issues
	* get-pods-distribution: Displays distribution of pods across nodes
	* get-etcd-stats: Collects ETCD pods status
	* get-update-status: Checks for cluster update progress and status
	* get-deployments: Gathers deployment statuses in namespaces. Single running pod are highlighted
	* get-ssl: Validates SSL certificates and expiration dates
	* get-alert: Retrieves active alerts within the cluster
	* get-pod-health: Checks health and status of individual pods
	* get-overcommitted: Identifies overcommitted resources
	* get-pv: Monitors Persistent Volume status and availability
	* get-pod-status: Collects overall pod status across the cluster
	* get-pod-usage: Checks resource usage metrics of pods
	* get-ns: Lists namespaces and their statuses
	* get-route: Retrieves route configurations and access points
	* get-total-resource: Provides a summary of total cluster resources
	* get-operators: Reports on Operator statuses and health
	* get-users-group: Lists users and groups with cluster access
	* get-machine: Gathers machine information in the cluster
	* get-daemonset: Checks DaemonSet statuses
	* get-apps: Retrieves application statuses and configurations
	* send-mail: Sends the generated report via email.
	* send-teams-notifications: Sends the alert notifications via teams and mail

### How to usage

Clone or copy this repository on the ansible node

```bash
git clone https://github.com/kxr/ocpcr.git
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

# Sample Report

[Click OpenShift Full Sample Report](https://github.com/emrahuludag/sysknow/raw/main/docs/openshift/img/openshift-sample-report.html)


![](./img/ocp-report-sample01.png? ':size=80%')
![](./img/ocp-report-sample02.png? ':size=80%')
![](./img/ocp-report-sample03.png? ':size=80%')
![](./img/ocp-report-sample04.png? ':size=80%')
![](./img/ocp-report-sample05.png? ':size=80%')
![](./img/ocp-report-sample06.png? ':size=80%')
![](./img/ocp-report-sample07.png? ':size=20%')

