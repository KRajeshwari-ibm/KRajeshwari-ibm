---
title: TARGET
tabs:
  [
    "Overview",
    # "Network Connectivity",
    "Runbooks",
    # "Configurations",
    # "Documentations",
    "Known Issues",
    "Changed Request",
    "Patch Management"
  ]
---

# TARGET RUN-BOOK

Target Corporation is an American retail corporation that operates a chain of discount department stores and hypermarkets, headquartered in Minneapolis, Minnesota. It is the seventh-largest retailer in the United States, and a component of the S&P 500 Index.[3] The company is one of the largest American-owned private employers in the United States.

## Click [https://cloud.ibm.com/docs/mas-ms?topic=mas-ms-migration-from-maximo-saas-flex-or-on-premise]


import jsonData from '/root/data/acom_runbook_issues.json';


<div>
  <table>
    <thead>
      <tr>
        <th>ID</th>
        <th>Title</th>
        <th>Labels</th>
      </tr>
    </thead>
    <tbody>
      {jsonData.map((item, index) => (
        <tr key={index}>
          <td><a href={item.url}>{item.number}</a></td>
          <td>{item.title}</td>
          <td>{item.labels}</td>
        </tr>
      ))}
    </tbody>
  </table>
</div>



## High-Level Steps
| S.NO     | Description   |
| -------- | -------------- |
| 0 | [**Environment Information and Configuration**](#0.-Configuration-and-environment-information) |
| 0.1 | ### Step 0.1: Verify Database version

1. **Database Configuration**: Run a Configurations details check.
   ```bash
   db2level > level.out
   db2licm -l > licm.out
   db2 get dbm cfg > dbmcfg.out
   ```
  ```bash
   db2 connect to DBNAME
   db2 get db cfg for DBNAME > dbcfg.out
    ```

```bash
   db2 "SELECT 'SELECT COUNT() AS row_count FROM ' || TABSCHEMA || '.' || TABNAME || ';'FROM SYSCAT.TABLES WHERE TYPE = 'T' " > tablecount.out
   db2 -tvf tablecount.out -z tablecount.out1
    ```
  ```bash
db2 "SELECT COUNT() AS TABLE_COUNT FROM SYSCAT.TABLES WHERE TYPE = 'T' AND TABSCHEMA NOT LIKE 'SYS%'"
       ```
| 1 | [**Pre-Req Database Indexes and relations check between Source database and Target database) |
| 1.1 | [&emsp; IBM](#1.1-ibm) |
| 1.2 | [&emsp; ACOM](#1.2-acom) |
| 2 | [**Request Access**](#2.-request-access) |
| 3 | [**PROD ACOM Environment**](#3.-prod-acom-environment) |
| 3.1 | [&emsp; ACOM Production CP4D install on ROSA](#3.1-acom-production-cp4d-install-on-rosa) |
| 3.2 | [&emsp; ACOM Customizations](#3.2-acom-customizations) |
| 3.3 | [&emsp; Additional Artifacts Requested by ACOM](#3.3-additional-artifacts-requested-by-acom) |
| 3.4 | [&emsp; Accessing PROD ACOM ROSA and CP4D Portals ](#3.4-accessing-prod-acom-rosa-and-cp4d-portals) |
| 4 | [**NON-PROD ACOM Environment**](#4-nonprod-acom-environment) |
| 4.1 | [&emsp; ACOM NON-Production CP4D install on ROSA](#4.1-acom-non-production-cp4d-install-on-rosa) |
| 4.2 | [&emsp; ACOM Customizations](#4.2-acom-customizations) |
| 4.3 | [&emsp; Additional Artifacts Requested by ACOM](#4.3-additional-artifacts-requested-by-acom) |
| 4.4 | [&emsp; Accessing NON-PROD ACOM ROSA and CP4D Portals ](#4.4-accessing-nonprod-acom-rosa-and-cp4d-portals) |
| 5 | [**Monitoring**](#5-monitoring)|
| 5.1 | [&emsp; Custom Healthcheck Scripts](#5.1-custom-healthcheck-scripts)|
| 5.2 | [&emsp; Slack](#5.2-slack)|
| 5.3 | [&emsp; PagerDuty](#5.3-pagerduty)|
| 5.4 | [&emsp; AlertManager in ROSA](#5.4-alertmanager-in-rosa)|
| 6 | [**Logging**](#6-logging)|
| 7 | [**Backup and Restore**](#7-backup-and-restore)|
| 7.1 | [&emsp; OADP Backup Setup Steps](#7.1-oadp-backup-setup-steps)|
| 7.2 | [&emsp; Discussions specific to Backup](#7.2-discussions-specific-to-backup)|
| 7.3 | [&emsp; AWS EFS Backup](#7.3-aws-efs-backup)|
| 8 | [**Log Tickets - Red Hat, AWS and SalesForce**](#8-log-tickets---red-hat-aws-and-salesforce) |
| 8.1 | [&emsp; Red Hat and AWS Cases](#8.1-red-hat-and-aws-cases) |
| 8.2 | [&emsp; SalesForce CP4D Skill Cases](#8.2-salesforce-cp4d-skill-cases) |
| 9 | [**Debugging and Diagnostics**](#9-debugging-and-diagnostics)|
| &emsp; 9.1 | [&emsp; Troubleshooting Common Issues](https://github.ibm.com/analytics-cloud-managed-services/iacs-devops/wiki/asml-monitoring#15-common-alerts-causes-and-resolutions)%7C
| 10 | [**ACOM specific Requirements**](#10-acom-specific-requirements)
| 10.1 |[&emsp; License Service and how to collect a snapshot](#10.1-license-service-and-how-to-collect-a-snapshot)
| 10.2 |[&emsp; To retrieve a License Report Snapshot **(do not provide to acom any other license report - only a snapshot)**](#102-to-retrieve-a-license-report-snapshot--do-not-provide-to-acom-any-other-license-report---only-a-snapshot)
| 11 | [**Technical Tips**](#11.0-technical-tips) |
| 11.3 | [&emsp; Add Custom CP4D URL in OpenShift](#11.3-add-custom-cp4d-url-in-openshift) |


## 0. Install and Environment Information

https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3830

https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3576#issue-33644682

### 0.1  Software Architecture : High Level Network Diagram

For Each ACOM env there are **two VPCs** in the AWS Tokyo Region:
1. IBM VPC
1. ACOM VPC

VPCs are attached to the ACOM **Transit Gateway** (https://docs.aws.amazon.com/vpc/latest/tgw/tgw-transit-gateways.html) 

**IBM VPC contains:**
* ROSA Cluster single zone
* Bastion Node

**ACOM VPC contains:**
* Databases
* Data Centers
* AWS Route53 Private Hosted zone for the custom private domain and record that matches custom CP4D Console Route

![image](https://media.github.ibm.com/user/83177/files/d01d5d84-8c17-4d52-bb43-1713888ed70c)


### 0.2 Software Stack and Versions
- OCP 4.12.16
- CP4D 4.6.6

### 0.3 CP4D Components and Versions:

**To retrieve the current versions of installed components**:
https://CP4D_URL/zen/#/versionDetails

![image](https://media.github.ibm.com/user/83177/files/5ae79b39-fe96-494f-9e18-97f22912b7b9)

## 1 AWS Resources
### 1.1 IBM
_* Production Account - 612500463914_ 
_* Region: Tokyo (ap-northeast-1)_
_* Single Zone_

| IBM AWS Resource| Production | Non-Production |
| ----------- | ----------- |--------|
| VPC | vpc-068db7223137815c9 |vpc-03f85e7e603d24ccb|
| CIDR | 10.175.60.0/24 | 10.176.60.0/24 |
| Security Group | sg-0ad0bcd123fc88663 | sg-0b50d55c5c2fc63ee |
| EFS | fs-0918817fa6462aafe (acom-prod-efs) | fs-0f71e089989327a57 (acom-nonprod-efs) |

### 1.2 ACOM

| ACOM AWS Resource| Production | Non-Production |
| ----------- | ----------- |--------|
| VPC | N/A | N/A |
| CIDR | 10.165.0.0/16 | 10.166.0.0/16 |
| Route53 Hosted Zone Domain | icp4d.prod.adaptor.backyard.acom.co.jp | icp4d.dev.adaptor.backyard.acom.co.jp |
| Route53 Hosted Zone Record | zen-cpd-zen | zen-cpd-zen |


Hosted zone setup: https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3602#issuecomment-58828299

## 2 Request Access
To Request access to one of ACOM environments:

1. Create a 'Request Access - ACOM Production (or Non-Production) environment for {yourname}' Git and specify:
&emsp; * which ACOM Environment you require access to.
&emsp; * include a request for your SOS account to be added to an ACOM bastion node (acom-bastion or acom-nonprod-bastion). 
&emsp; * if you need access to a cluster in the Red Hat HybridCloud Platform, include a request access and a justification.

2. After the request was approved and access granted, find credentials for the cluster and CP4D in [1Password Vault](https://ibm.ent.1password.com/vaults/sn47dds7cxxatzw3vgxk4vxhma/allitems/cm2xopyk32b35efat5xo64m7si)

3. If you require access to the AWS IACS Production account, log a separate Git 'Request Access - AWS IACS Production Account for {yourname}'

## 3 PROD ACOM Environment
As of end of June 28, 2023<p></p>
ACOM Production Install is still in the 'Initial' Production configuration.  The install performed with [CPDeployer](https://ibm.github.io/cloud-pak-deployer/). <p></p>
Number of worker nodes: 9 <p></p>

The Full Production install has 2 additional nodes with the total of 11 worker nodes.<p></p>

**Components Installed in Production-Initial:**
1. Control Plane
2. Operators
3. Foundational Services
4. Watson Query - backup
5. Data Management Console - backup
6. DataStage - backup
7. Watson Pipelines - backup
8. Watson Knowledge Catalog - each components has its own backup strategy
9. Common Database Services
10. Analytics Engine Powered by Apache Spark - backup
11. Watson Studio - backup
12. SPSS Modeler - backup
13. Scheduling Services
14. Jupyter Notebook Server with Python + Jupyter Lab - backup
15. Common Core Services
16. Watson Machine Learning

### 3.1 ACOM Production CP4D install on ROSA

* ROSA OpenShift Console: https://console-openshift-console.apps.acom-prod.jglp.p1.openshiftapps.com
* Cloud Pak for Data: https://cpd-zen-46.apps.acom-prod.jglp.p1.openshiftapps.com/zen 
* Red Hat Hybrid Cloud Platform: https://console.redhat.com/openshift 
* AWS IACS Production Account (612500463914): https://ap-northeast-1.console.aws.amazon.com/ec2/home?region=ap-northeast-1#Home

### 3.2 ACOM Customizations
1. CP4D DataStage configuration:
* [Configuring memory and heap size in DataStage](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=resources-configuring-memory-heap-size-in-datastage)
* [Creating custom NLS maps in DataStage](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=nls-creating-custom-maps)
&emsp;* [Git with Instructions for NLS Setup in PROD](https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3919)
2. CP4D DataStage NLS files provided by ACOM: 
* - ibm-1390_P110-2003_acom.ucm
* - ibm-16684_P110-2003_acom.ucm
* - ibm-1390_P110-2003_acom.cnv
* - ibm-16684_P110-2003_acom.cnv

Location https://ibm.ent.box.com/folder/207191631071?s=zwdetdb29hwfpndqu5vady1b6hrl0pc2 (Owner jmccarey@us.ibm.com)

4. Custom URL for the CP4D Console 
Current (as of Jun 28, 2023) - CP4D console URL: https://zen-cpd-zen.icp4d.prod.adaptor.patio.acom.co.jp
To be changed to - https://zen-cpd-zen.icp4d.prod.adaptor.backyard.acom.co.jp  - after configured and tested in NON-PROD
**Note**: the custom URL is not accessible in the browser, it is tested from a bastion node.

5. Backup email notification Format:

&emsp;&emsp;Backup start time:     <sub>_(in UTC or JST)_</sub>
&emsp;&emsp;Backup stop time:     <sub>_(in UTC or JST)_</sub>
&emsp;&emsp;Duration:       <sub>_(in format as '1h 45min')_</sub>
&emsp;&emsp;Status: **Success** <sub>_or_</sub> **Failed - {short explanation}** <sub>_or_</sub> **Incomplete - more time required**

&emsp;&emsp;https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3605#issuecomment-59969509

6. Update of cluster components or nodes email notification Format:

```ruby
1.  Work time (Start - Completion time):
2.  Evidence before addition (number of nodes):
Evidence after addition (number of nodes):
3.  Evidence before installation (service list):
Evidence after installation (service list):
```
&emsp;&emsp;https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3956#issuecomment-59972432


### 3.3 Additional Artifacts Requested by ACOM
* Licensing Service snapshot in monthly TAM updates to ACOM: see [ACOM Specific Requirements](https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-DEVOPS/wiki/ACOM---Runbook/#10-acom-specific-requirements).
* Access Control List review confirmation - expected as a quarterly Ticket from ACOM 
  ** MUST be forwarded to @Nikhil Sahni to action [Git-3885](https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3885)
  ** After completed by Nikhil, provide the confirmation to the customer.
* ROSA Platform Audit logs - expected as an occasional request from ACOM for internal audits. Should be requested via an SF ticket.
* Potential requests for additional Backups - not more than 2-3 per/month 


### 3.4 Accessing PROD ACOM ROSA and CP4D Portals

Access to environment is only allowed via VPN.

**To Connect:**
&emsp; 1.  Connect to mgmt-035-jump with your SSO account.
&emsp; 2.  From mgmt-035-jump `ssh` to the production bastion node

&emsp;  `ssh acom-bastion`

Output:
&emsp;  `[<yourSSOUser>@ip-10-175-60-36 ~]$`


<p>&#x1f4a1; **Important:** check the box name or IP to ensure you logged on to the right ACOM environment - PROD: ip-10-175-60-94
</p> 

## 4 NON-PROD ACOM Environment
The install performed with cpd-cli tool {TODO link}. <p></p>
Number of worker nodes: 9 <p></p>

**Components Installed in NoN-Production (the same components as in Production-Initial): **
1. Control Plane
2. Operators
3. Foundational Services
4. Watson Query - backup
5. Data Management Console - backup
6. DataStage - backup
7. Watson Pipelines - backup
8. Watson Knowledge Catalog - each components has its own backup strategy
9. Common Database Services
10. Analytics Engine Powered by Apache Spark - backup
11. Watson Studio - backup
12. SPSS Modeler - backup
13. Scheduling Services
14. Jupyter Notebook Server with Python + Jupyter Lab - backup
15. Common Core Services

### 4.1 ACOM NON-Production CP4D install on ROSA

* ROSA OpenShift Console: https://console-openshift-console.apps.acom-nonprod.rx8k.p1.openshiftapps.com
* Cloud Pak for Data: https://cpd-zen-46.apps.acom-prod.jglp.p1.openshiftapps.com/zen 
* Red Hat Hybrid Cloud Platform: https://console.redhat.com/openshift 
* AWS IACS Production Account (612500463914): https://ap-northeast-1.console.aws.amazon.com/ec2/home?region=ap-northeast-1#Home

### 4.2 ACOM Customizations

1. CP4D DataStage configuration:
* [Configuring memory and heap size in DataStage](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=resources-configuring-memory-heap-size-in-datastage)
* [Creating custom NLS maps in DataStage](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=nls-creating-custom-maps)
   ** [NLS Setup in PROD](https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3919)
2. CP4D DataStage NLS files provided by ACOM: 
* - ibm-1390_P110-2003_acom.ucm
* - ibm-16684_P110-2003_acom.ucm
* - ibm-1390_P110-2003_acom.cnv
* - ibm-16684_P110-2003_acom.cnv

Location https://ibm.ent.box.com/folder/207191631071?s=zwdetdb29hwfpndqu5vady1b6hrl0pc2 (Owner jmccarey@us.ibm.com)

4. Custom URL for the CP4D Console 
Current (as of Jun 28, 2023) - CP4D Custom Console URL: https://zen-cpd-zen.icp4d.prod.adaptor.patio.acom.co.jp
To be changed to - https://zen-cpd-zen.icp4d.prod.adaptor.backyard.acom.co.jp  - change first in NON-PROD, then in PROD

5. Backup email notification Format:

&emsp;&emsp;Backup start time:     <sub>_(in UTC or JST)_</sub>
&emsp;&emsp;Backup stop time:     <sub>_(in UTC or JST)_</sub>
&emsp;&emsp;Duration:       <sub>_(in format as '1h 45min')_</sub>
&emsp;&emsp;Status: **Success** <sub>_or_</sub> **Failed - {short explanation}** <sub>_or_</sub> **Incomplete - more time required**

&emsp;&emsp;https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3605#issuecomment-59969509

6. Update of cluster components or nodes email notification Format:

```ruby
1.  Work time (Start - Completion time):
2.  Evidence before addition (number of nodes):
Evidence after addition (number of nodes):
3.  Evidence before installation (service list):
Evidence after installation (service list):
```
&emsp;&emsp;https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3956#issuecomment-59972432



### 4.3 Additional Artifacts Requested by ACOM
 -- ?? Need to be discussed with ACOM (if same as PROD)

### 4.4 Accessing NON-PROD ACOM ROSA and CP4D Portals
**To Connect:**
&emsp; 1.  Connect to mgmt-035-jump with your SSO account.
&emsp; 2.  From mgmt-035-jump `ssh` to the production bastion node

&emsp;  `ssh acom-bastion`

Output:
&emsp;  `[<yourSSOUser>@ip-10-176-60-94 ~]$`


<p>&#x1f4a1; **Important:** check the box name or IP to ensure you logged on to the right ACOM environment - NON-PROD: ip-10-176-60-94</p>

## 5 Monitoring
**ACOM Production and NoN-Production environments are monitored with:** 
1. SOS Compliance Portal
2. Custom Healthchecks and Alerts in Slack and PagerDuty
3. AlertManager in ROSA with custom Alert Rules for CP4D projects - zen-46 and ibm-common-services
4. CP4D Alerts forwarding from zen-watchdog to Slack (https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=alerts-forwarding-slack)

![image](https://media.github.ibm.com/user/83177/files/e51be219-6dc8-495e-afac-68f6195e4a43)

### 5.1 Custom Healthcheck Scripts

The following metrics are currently being checked in 5 min intervals:
1. CP4D Portal endpoint URL.
2. Custom (ACOM) Cp4D Portal URL.
3. All Service Instances endpoint URLs (DV, DMC, DataStage).
3. All Service Instances status and pod states.
4. Services status - an overall count and a status of services that are not in Normal state.
5. Number of pods in Error states in CP4D namespaces: zen-46 and ibm-common-services

### 5.2 Slack

**PROD:** 
[#iacs_acom_prod_alerts](https://ibm-analytics.slack.com/archives/C054YKUHCVA)

![image](https://media.github.ibm.com/user/83177/files/89f5dd37-754c-4642-9263-d366c2775369)

**NON-PROD (in progress):** 
[#iacs_acom_non-prod_alerts](https://ibm-analytics.slack.com/archives/C05DBGZDTGV)

### 5.3 PagerDuty
Dedicates Service for ACOM in PagerDuty:
[IACS_ACOM_PriorityCallHandling_Ankur](https://ibm.pagerduty.com/service-directory/P264WNW)

**PagerDuty Alerts are currently enabled for (as of Jun 28):**

- CP4D Portal Endpoint
- Service Instances Endpoints
- Cluster's Nodes down
- CLUSTER OPERATORs degraded

#### 5.3.1 How to Enable/Disable PD Alerts
1. Login to NON-PROD bastion node of ACOM environment as 'ec2-user' or PROD bastion node of ACOM environment as 'root' user (see login command in [1Password Vault](https://ibm.ent.1password.com/vaults/sn47dds7cxxatzw3vgxk4vxhma/allitems/cm2xopyk32b35efat5xo64m7si)).
2. Edit crontab
`crontab -e` 

&emsp; **To Disable all PD Alerts set:**

&emsp; `PD=enable`

&emsp; **To Enable all PD Alerts set:**

&emsp; `PD=disable`

&emsp; **To Enabled all PD Alerts _EXCEPT Service Instance URLs_ use the following settings:**

&emsp; `PD=enable`
&emsp; `PD_INSTANCE=disable`

### 5.4 AlertManager in ROSA

Since the Main AlertManager is used by the Red Hat SRE team we cannot add any custom Notification Receivers.<p></p>
Instead the secondary AlertManager for user-workloads is setup to monitor custom namespaces: zen-46, ibm-common-services.

Custom Alerts creation and configuration is currently in Progress..

## 6 Logging

There are three types of Logs to be collected in an ACOM environment:
**1. System Audit Logs**
These are the OpenShift Platform Audit logs that are forwarded to the SOS Compliance QRadar instance.
**2. CP4D Application Logs**
These are application logs from application pods.
Can be collected either:
* in the CP4D Portal -> Support -> Diagnostics 
OR
* with cpd-cli tool

**3. CP4 Audit Logs**
https://www.ibm.com/docs/en/cloud-paks/cp-data/4.0?topic=considerations-auditing-cloud-pak-data

These logs likely to be setup shortly when ACOM provides an SIEM solution (Splunk, LogDNA, QRadar).

## 7 Backup and Restore

### 7.1 OADP Backup Setup Steps

* Refer to [this documentation for OADP Backup Setup on AWS](https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Wiki-Runbooks/wiki/OADP-Backup-Setup-on-AWS)

* Refer to [this documentation for OADP Backup Setup on AWS for ACOM](https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Wiki-Runbooks/wiki/OADP-Backup-Setup-on-AWS---ACOM)


### 7.2 Discussions specific to Backup
https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3605

### 7.3 AWS EFS Backup

[Setting up EFS backup in AWS](https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Wiki-Runbooks/wiki/Elastic-File-Storage-Backup-in-Amazon-Web-Services)

#### ACOM NON-PROD EFS Backup Setup

![image](https://media.github.ibm.com/user/16908/files/38c6cbde-a1bc-4ca8-b55f-4f20db3ec3ec)

![image](https://media.github.ibm.com/user/16908/files/414390f1-988f-4b89-b8a5-79d0c82605d5)

![image](https://media.github.ibm.com/user/16908/files/1cf08e7b-7912-4c17-9b27-83cc2813a34a)

![image](https://media.github.ibm.com/user/16908/files/045eed0e-37e9-41bc-8882-76477783b085)

![image](https://media.github.ibm.com/user/16908/files/ac967744-33b4-45e4-b97f-eb78534452ae)

#### ACOM Prod EFS Backup Setup 

![image](https://media.github.ibm.com/user/16908/files/ad8aabe0-381f-4587-8613-c3d3912de5d6)

![image](https://media.github.ibm.com/user/16908/files/235096f8-6d2a-4989-a382-73a510278a42)
![image](https://media.github.ibm.com/user/16908/files/1b953c6f-e03a-4b83-9739-b479111092b1)
![image](https://media.github.ibm.com/user/16908/files/f9d2f2a6-2765-4f92-8d7a-15d9ff438776)

## 8 Log Tickets - Red Hat, AWS and SalesForce

### 8.1 Red Hat and AWS Cases

**Red Hat tickets can be logged 2 ways:**
* **PREMIUM SUPPORT** - AWS Support case with '**Red Hat OpenShift Service on AWS (ROSA)**' Service topic.
* **STANDARD SUPPORT** - Red Hat Support Case 
&emsp;&emsp; - either from inside of the OpenShift cluster console (the cluster ID will be pre-populated in the case)
&emsp;&emsp;or
&emsp;&emsp; - in https://access.redhat.com/support. To find a cluster ID, login to [Red Hat Hybrid Cloud Platform](https://console.redhat.com/openshift), search for a cluster by name: 'acom-prod'  or 'acom-nonprod', copy cluster ID from the 'Overview' page.


ℹ️ **1. Using Red Hat Account** (provided by IBM) - **STANDARD SUPPORT**
via Red Hat Support Portal: https://access.redhat.com/support 
You must be logged on to Red Hat with IBM RH account # 6585020: 

![image](https://media.github.ibm.com/user/83177/files/4e410798-66b8-46a8-9ef4-e213772574c8)

To check your account number (IBM Wide account): 
* Login to https://console.redhat.com/openshift with the Red Hat account provided to you by IBM.
* Check that account number the same as on the screenshot above.

ℹ️ **2. Using AWS IACS Production Account** as an AWS ROSA case - **PREMIUM SUPPORT**
The AWS Support team will contact Red Hat for the cluster issues.

To open a case:
* Login to the AWS IACS Production account
* On the AWS Portal, at the top right corner, click on **?**, then Support Center<p></p>
![image](https://media.github.ibm.com/user/83177/files/4ea36575-afae-4d36-bec8-2898d7309d39)
* Pick 'Technical' and 'Create case'<p></p>
![image](https://media.github.ibm.com/user/83177/files/f9f51d0b-8f6e-4a43-8acd-18642a29b95b)
* Choose 'Technical' again and from the 'Service' drop-down list pick the **'Red Hat OpenShift Service on AWS (ROSA)'**.<p></p>

&emsp;&emsp; ![image](https://media.github.ibm.com/user/83177/files/8cae125f-cd60-4e27-8406-55789f191b1f)

In the 'Severity' drop-down list, the highest priority is **Business-critical system down**. 
It supports the quickest response time from the AWS Support team. <p></p>
![image](https://media.github.ibm.com/user/83177/files/8ba0acb4-36f8-4324-916b-8a39c429dfbd)

### 8.2 SalesForce CP4D Skill Cases

ℹ️ **1. Sales Force tickets** for Japan customers, by default, fall under the Japan/Australian CP4D Support team.
If you log a ticket during the EST timezone and need a quick attention from the North America CP4D Support team, tag Support Manager and Team Leads - see "**SalesForce tickets logging (on ACOM behalf)**" in [Git-3485](https://github.ibm.com/Analytics-Cloud-Managed-Services/IACS-Issue-Tracking/issues/3485)

![image](https://media.github.ibm.com/user/83177/files/c016a613-0887-4429-9c55-4beace1738bf)

## 9 Debugging and Diagnostics

## 10 ACOM specific Requirements
### 10.1 License Service and how to collect a snapshot

To generate a Licensing report:
1. Retrieve a Licensing Service URL
2. Obtain a Licensing Service API Token

To retrieve a Licensing Service URL:
* In the OpenShift Console, 'Networking'->'Routes', pick 'ibm-common-services' namespace and copy a value of the ibm-licensing-service-instance route.

additional information in the [Retrieve License Service Data](https://www.ibm.com/docs/en/cloud-paks/foundational-services/3.23?topic=pcfls-apis#ls_url) documentation.

![image](https://media.github.ibm.com/user/83177/files/e6d7e934-78f5-4b02-9e97-690678ffc337)

**To obtain a Licensing Service API Token:**
* Copy a value of the ibm-licensing-token secret in the 'ibm-common-services' namespace.

additional information in the [Obtaining and Updating an API token](https://www.ibm.com/docs/en/cloud-paks/foundational-services/3.23?topic=authentication-license-service-api-token) documentation.

### 10.2 To retrieve a License Report Snapshot  (DO NOT PROVIDE to ACOM ANY OTHER LICENSE report - only a SNAPSHOT)
#### 10.2.1 Get Snapshot in the Browser
* Logon to OpenShift Console
* Run a Licensing Service URL in the browser (ex: https://ibm-licensing-service-instance-ibm-common-services.apps.acom-nonprod.rx8k.p1.openshiftapps.com)
* Enter an API token when prompted (ex: yNpmLeDdvF2ao4XFYGallveC)
* Download "Snapshot" zip

<p>&#x1f4a1; Note: if the License service keeps prompting for an API token (although the token is correct), refresh the application by deleting the pod:</p>
`oc delete pods -l app=ibm-licensing-service-instance -n ibm-common-services`

![image](https://media.github.ibm.com/user/83177/files/5d998421-c16e-45e3-9a2e-3e133033c238) 

#### 10.2. Get Snapshot from a command prompt
**Example for NON-PROD:**
`oc login https://api.acom-nonprod.rx8k.p1.openshiftapps.com:6443 --username cluster-admin --password {pwd}`

`curl -X GET https://ibm-licensing-service-instance-ibm-common-services.apps.acom-nonprod.rx8k.p1.openshiftapps.com/snapshot?token=yNpmLeDdvF2ao4XFYGallveC -k --output licensing-audit-report-Jul12_2023.zip`

## 11.0 Technical Tips

### 11.1 How to check ELB (AWS Elastic Load Balancer) attributes and IdleTimeout
Run aws elb CLI commands: **describe-load-balancers** and **describe-load-balancer-attributes** 

### NON-PROD

```ruby
export VPC_ID=vpc-03f85e7e603d24ccb

LOAD_BALANCER=`aws elb describe-load-balancers --output text | grep $VPC_ID | awk '{ print $5 }' | cut -d- -f1 | xargs`
for lbs in ${LOAD_BALANCER[@]}; do
aws elb describe-load-balancer-attributes  --load-balancer-name $lbs
done
```
### PROD

```ruby
export VPC_ID=vpc-068db7223137815c9

LOAD_BALANCER=`aws elb describe-load-balancers --output text | grep $VPC_ID | awk '{ print $5 }' | cut -d- -f1 | xargs`
for lbs in ${LOAD_BALANCER[@]}; do
aws elb describe-load-balancer-attributes  --load-balancer-name $lbs
done
```

### Output shows the IdleTimeout in the list of ELB attributes:
```json
    "LoadBalancerAttributes": {
        "CrossZoneLoadBalancing": {
            "Enabled": false
        },
        "AccessLog": {
            "Enabled": false
        },
        "ConnectionDraining": {
            "Enabled": false,
            "Timeout": 300
        },
        "ConnectionSettings": {
            "IdleTimeout": 1800
        },
        "AdditionalAttributes": [
            {
                "Key": "elb.http.desyncmitigationmode",
                "Value": "defensive"
            }
        ]
    }
```
### 11.2 Turn off Host Header Injection Check
If the Custom CP4D URL reports a connection problem: 
![image](https://media.github.ibm.com/user/83177/files/af875d54-030c-49ee-9754-f275850dd8b2)

Perform [Turning off the host header injection check](https://www.ibm.com/docs/en/cloud-paks/cp-data/4.6.x?topic=environment-turning-off-host-header-injection-check) and verify that it resolved the connection issue.

**BEFORE:**
```ruby
[root@ip-10-175-60-36 ~]# oc get cm product-configmap -oyaml | grep INJECT
  HOST_INJECTION_CHECK_ENABLED: "true"
```


**Action Taken:**
Turn off the host header injection check
```ruby
[root@ip-10-175-60-36 ~]# oc patch configmap product-configmap --type=merge --patch '{"data": {"HOST_INJECTION_CHECK_ENABLED":"false"}}'
configmap/product-configmap patched
```
**Reload nginx pod:**
```ruby
[root@ip-10-175-60-36 ~]# oc rollout restart deployment/ibm-nginx
deployment.apps/ibm-nginx restarted
```
**After:**
```ruby
[root@ip-10-175-60-36 ~]# oc get cm product-configmap -oyaml |grep INJECT
  HOST_INJECTION_CHECK_ENABLED: "false"
```

```ruby

[root@ip-10-175-60-36 ~]# curl -k https://zen-cpd-zen.icp4d.prod.adaptor.patio.acom.co.jp 
<html>
<head><title>303 See Other</title></head>
<body>
<center><h1>303 See Other</h1></center>
<hr><center>openresty</center>
</body>
</html>
[root@ip-10-175-60-36 ~]# 
```

### 11.3 Add Custom CP4D URL in OpenShift

1. In the OpenShift Portal->Networking Create a new Route
2. Set Route Name (ex: acom-cpd-new)
3. Add Hostname - name of the new URL but without 'https' (ex: zen-cpd-zen.icp4d.prod.adaptor.backyard.acom.co.jp)
4. Leave the Path setting as is ('**/**')
5. Pick from the Services drop-down list: **ibm-nginx-svc**
6. Pick Port: **443 -> 8443 (TCP)**
7. Set TLS Termination to '**Re-encrypt**'
8. Set Insecure traffic to '**Redirect**'
9. Tick '**Secure Route**' box and add certificates provided by the client
a. **Public end certificate** add to the Certificates section
b. **Private key** to 'Private key' section (the key must be decrypted)
c. **Root CA certificate or certificate chain** that includes a root certificate to 'CA certificate' section
d. **Destination certificate** (retrieve it from the default cpd route) add to 'Destination CA certificate' section
10. Complete route creation and make sure it is '**Accepted**' by OpenShift

![image](https://media.github.ibm.com/user/83177/files/d79cf984-7a81-4e1a-92e4-002ff5e22eac)

11. Copy '**Router canonical hostname**'  (ex: router-default.apps.acom-prod.jglp.p1.openshiftapps.com) and 
send it to the client for setting up a record in their Hosted Zone on AWS. 
The Router canonical hostname is used as a CNAME in the HZ record:

![image](https://media.github.ibm.com/user/83177/files/ec06cda7-9c5a-4188-8849-290f16caf01f)

12. The Router canonical hostname is used as a CNAME in the HZ record, for example:

![image](https://media.github.ibm.com/user/83177/files/7e6a9368-c056-4c19-9fcf-daf56279196e)

13. To access a customer Hosted Zones record for successfully navigating to the new CP4D URL: 
a. The customer must authorize our (IBM) VPC to submit an AssociateVPCWithHostedZone request to associate the VPC with a specified hosted zone that was created by a customer AWS account:
```ruby
aws route53 create-vpc-association-authorization --hosted-zone-id Z04189482882OFGWDY8PV --vpc VPCRegion=ap-northeast-1,VPCId=vpc-068db7223137815c9 --region ap-northeast-1
```
&emsp; &emsp; b. Then on IBM side run a `associate-vpc-with-hosted-zone` command to associate our Amazon VPC with a customer's private hosted zone:
```ruby
aws route53 associate-vpc-with-hosted-zone --hosted-zone-id Z04189482882OFGWDY8PV --vpc VPCRegion=ap-northeast-1,VPCId=vpc-068db7223137815c9
```
14. To check that the hosted zone was associated with VPC successfully run `aws route53 list-hosted-zone-by-vpc` command:
```ruby
aws route53 list-hosted-zones-by-vpc --vpc-id vpc-068db7223137815c9 --vpc-region ap-northeast-1
```
The output should contain the hosted zone id from the previous `associate-vpc-with-hosted-zone` command:
```ruby
{
    "HostedZoneSummaries": [
        {
            "HostedZoneId": "Z04189482882OFGWDY8PV",
            "Name": "adaptor.backyard.acom.co.jp.",
            "Owner": {
                "OwningAccount": "126752972853"
            }
        },
        {
            "HostedZoneId": "Z01581712FHG87Z44ALU3",
            "Name": "icp4d.prod.adaptor.backyard.acom.co.jp.",
            "Owner": {
                "OwningAccount": "126752972853"
            }
        },
    ],
    "MaxItems": "100"
}
```
15. Ask a customer to test the new custom CP4D URL.

## 12.0 In Progress 

1. Integration of OpenShift Console logon with IBM Verify
2. Named users for CP4D
3. Custom Alerts configuration 


TODO:
### 0.2 TODO - Install



