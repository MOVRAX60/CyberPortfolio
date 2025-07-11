---
title: "Sentinel Monitoring and Alerting with SQL Injections"
url: "/projects/cloud/grad-project/"
description: "Azure"
cover : "/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a5de6192.jpg"
---

# Contents {#contents .contents-heading style="margin-top: 0in"}
- Introduction
- Resources
- Diagram
- Billing Alert 
- Networking	
- Configuration	
- Virtual Network	
- Subnet	
- Virtual Machine Provisioning	
- Bastion	
- Database provisioning	
- Basics	
- Networking	
- Review + Create	
- Monitoring Alert	
- Action Group	
- Monitoring Alert Cont.	
- WAF Policy	
- Application Gateway Configuration	
- Log Analytics Configuration	
- Sentinel Configuration	
- Data Connectors	
- WAF	
- Virtual Machine	
- Sentinel Workbook templates	
- DVWA Installation	
- SQL Injection	
- Sentinel Workbook	
- Sentinel Incident	
- WAF Prevention	
- Remarks	
- Resources	
- Azure Resources

# Introduction {#introduction .western style="page-break-before: always"}

One of my main focuses for this project was the idea of deploying a
cloud SIEM solution. Sentinel is Azure\'s SIEM solution with a full
stack of security resources to enable security and monitoring of Cloud
and On-Premises deployments. Sentinel uses a series of data connectors
to gather data and log analytics workspace to ingest and visualize data.
The main service Sentinel uses the Log Analytics Workspace. This
workspace holds the data gathered from the endpoints in the network so
Sentinel can review, and flag logs as instructed. Most Azure services
can use Microsoft Defender for Cloud, which feeds data to Log Analytics.
To generate logs, Sentinel alerts, and Sentinel incidents, I need a
system that will be exploitable. To accomplish this, I am using an
Ubuntu VM running a Linux Apache PHP Stack and a MySQL Azure Database;
with these two services, I will be able to host DVWA or \"Damn
Vulnerable Web Application,\" which is filled with various web
application vulnerabilities. And the last key component of this project
is the Azure Web Application Firewall in detection mode because
prevention will immediately stop the exploitation of endpoints. I aim to
build a test environment and exploit the underlying database, which will
Send Alerts from the WAF, VM, and DB; through these alerts, I can
automate the creation of an incident.

# Resources {#resources .western}

Azure Sentinel

Data Connector: Microsoft Web Application Firewall

Data Connector: Syslog -- web server

Ubuntu Virtual Machine

Azure MySQL Database

Log analytics workspace

Windows Defender for Cloud

Azure Web Application Gateway

Azure Web Application Firewall policies

Azure Bastion

# Diagram {#diagram .western style="margin-top: 0in; page-break-before: always"}

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a5de6192.jpg)


# Billing Alert {#billing-alert .western}

My estimate for this project is that it will come in around 100 dollars,
so I wanted to set my billing alert to an actual cost instead of a
forecast because these services will be expensive, but I will not
utilize them long-term. I set my forecast as actual and for 80%
utilization because most of the security services are utilization based;
the cost will remain relatively low until I start generating logs from
the WAF and DB through.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_ba59b0c7.jpg)


![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a6d3b616.jpg)

# Networking {#networking .western}

## Configuration {#configuration .western style="margin-left: 0.5in"}

```commandline
az network vnet create -n SenVirtNet -g AzureSentinelProj2022 --address-prefixes 172.16.0.0/20

#Creating Virtual Network
az network vnet subnet create -g AzureSentinelProj2022 --vnet-name SenVirtNet \
    -n AppGateBoundary --address-prefixes 172.16.1.0/24
az network vnet subnet create -g AzureSentinelProj2022 --vnet-name SenVirtNet \
    -n WebSrvBoundary --address-prefixes 172.16.2.0/24
az network vnet subnet create -g AzureSentinelProj2022 --vnet-name SenVirtNet \
    -n DatBasBoundary --address-prefixes 172.16.3.0/24

#Creating Subnets for each boundary
az network nsg create -g AzureSentinelProj2022 --name AppGateNSG
az network nsg create -g AzureSentinelProj2022 --name WebSrvNSG
az network nsg create -g AzureSentinelProj2022 --name DatBaseNSG

#Creating the respective NSGs to control traffic flow
az network vnet subnet update -g AzureSentinelProj2022 -n AppGateBoundary \
        --vnet-name SenVirtNet --network-security-group AppGateNSG
az network vnet subnet update -g AzureSentinelProj2022 -n WebSrvBoundary \
        --vnet-name SenVirtNet --network-security-group WebSrvNSG
az network vnet subnet update -g AzureSentinelProj2022 -n DatBasBoundary \
        --vnet-name SenVirtNet --network-security-group DatBasNSG

az network nsg rule create -g AzureSentinelProj2022 -n DatBasBoundary \
    -n WebsrvSQLTraffic --priority 100 \
  --source-address-prefixes 172.16.2.0/24 --source-port-ranges '*'\
  --destination-address-prefixes 172.16.3.0/24 --destination-port-ranges 3306 --access allow
#Attaching Traffic from the web server network to the database
az network nsg rule create -g AzureSentinelProj2022 -n AppGateBoundary \
    -n WebsrvSQLTraffic --priority 100 \
  --source-address-prefixes 'MY-IP' --source-port-ranges 80 \
  --destination-address-prefixes '*' --destination-port-ranges 80 --access allow
# Allowing Application gateway to pass web traffic to webserver
az network nsg rule create -g AzureSentinelProj2022 -n AppGateBoundary \
    -n AppGateV2Rule --priority 101 \
  --source-address-prefixes '*' --source-port-ranges '*'\
  --destination-address-prefixes '*' --destination-port-ranges 65200-65535 --access allow
#Adding required ports for AppGate to communicate with Azure Services
az network nsg rule create -g AzureSentinelProj2022 -n AppGateBoundary \
    -n WebsrvSQLTraffic --priority 100 \
  --source-address-prefixes 172.16.1.0/24 --source-port-ranges 80 \
  --destination-address-prefixes 172.16.2.0/24 --destination-port-ranges 80 --access allow
#Allowing http traffic from the application gate to the webserver
```

## Virtual Network {#virtual-network .western style="margin-left: 0.5in"}

The SenVirtNet network uses a 172.16.x.x/20 addressing scheme. This will
give the endpoints more than enough space to exist in their subnets.
Each will be configured with its own traffic rules to ensure that the DB
is not accessible to anything other than it is intended to and that the
web server is only accessible via the application gateway and WAF. And
that the only route from the public IP is to the web application
gateway. Because of the vulnerable nature of using DVWA, I\'ve chosen to
disable the HTTP access until all services are correctly configured.

## Subnet {#subnet .western style="margin-left: 0.5in"}

AppGateBoundary - 172.16.1.0/24

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_5c35c6d7.jpg)

WebSrvBoundary - 172.16.2.0/24

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_baeedc72.jpg)

DatBasBoundary - 172.16.3.0/24

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_33279036.jpg)

# Virtual Machine Provisioning {#virtual-machine-provisioning .western}

I used an Ubuntu 20.04 server to hold the LAMP stack. This web server will exist in the WebSrvBoundary, and access will be enabled for the webserver to communicate with the database NSG and the Application gateway NSG. This web application is relatively small and does not need many resources to run a webpage.

```commandline
az vm create --name SenVirtWebSrv01 -g AzureSentinelProj2022 \
--image Canonical:0001-com-ubuntu-server-focal:20_04-lts:least --size Standard_B2s \
--vnet-name SenVirtNet --Subnet WebSrvBoundary --nsg WebSrvNSG \
--public-ip-address "" --private-ip-address 172.16.2.5 \
--authentication-type ssh --admin-username senvirtadmin --generate-ssh-keys
```


After running this command, we can see that SenVirtWebSrv01 has been created and is running.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_878b563d.jpg)

Once the webserver is active, I use the following CLI command to install the LAMP stack.

```commandline
az vm run-command invoke \
  -g AzureSentinalProj2022
  -n SenVirtWebSrv01 \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get install -y lamp-server"
# This script will run the install packages for a LAMP stack
```

Now that the LAMP stack has been installed. Using the bastion, I can verify that the Stack has been installed and that the web server is running as intended.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_4584d0ff.jpg)

Checking Version on installed applications

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_bedd8680.jpg)

Using curl to verify default Apache page is active

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_58a1bfd4.jpg)

Now that the ubuntu VM has been configured and verified, I can move to the Database creation.

# Bastion {#bastion .western}

Because of the NSG configuration, I opted to use a bastion, and the process for building one is very straightforward.

We can create a bastion by opening the connection tab on the webserver VM and selecting bastion, which brings us to the bastion page; because one has not been configured before, we are presented with this screen.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_6178a15d.jpg)

We\'ll use the Create Bastion using defaults to provide all the necessary functionality.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_acb2d73b.jpg)

And then, after a few minutes, the bastion is available, and we can open it from the connection tab in the VM.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_54f73045.jpg)

Using the keys generated from the VM creation, we can now use the bastion to log into the webserver.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_88412467.jpg)



# Database provisioning {#database-provisioning .western}

Azure Free accounts have 750 hours of MySQL flexible Server. By
utilizing this, I can spin up a small database and use it as a back end
for this website. The database for this instance does not need to be
large DVWA only requires less than 1 GB, so 20GB is more than suitable.
I used the following settings to launch this server.

## Basics {#basics .western style="margin-left: 0.5in"}

Subscription: Azure subscription 1

Resource group: Azure SentinelProj2022

Server Name: senvirtdb

Region: East US

MySQL: Version 5.7

Workload Type: Dev/Hobby

Compute+ Storage Burstable B1ms

Availability zone: No preference

## Networking {#networking-1 .western style="margin-left: 0.5in"}

Connectivity Method: Private Access (VNet Integration)

Subscription: Azure subscription 1

Virtual Network: SenVirtNet

Subnet: DatBasBoundary (172.16.3.0/24)

Private DNS Integration: Azure subscription 1

Private DNS zone: (New) senvirtdb.private.mysql.database.azure.com

## Review + Create {#review-create .western style="margin-left: 0.5in"}

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_c7aea5aa.png)

Once the database server was created and running, I could go to
databases and create the DVWA database to house the DVWA installation
data.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_389e1b40.png)

Now the database will be created, which is reflected in the database
window below.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_94183a2e.png)

Once the database has been created from the Web server, I can test that
the rules have been configured correctly and that the database has been
created. From the Web server, I connect to the database and then list
the databases.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_f001aaf9.png)

As we can see, the webserver can connect to the database through the NSG
rules.

# Monitoring Alert {#monitoring-alert .western}

I decided to use an alert focused on the database for my monitoring
alert. I started with the threshold of 300 as the baseline was 150, so
twice the request will signify that there is an issue on the database.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_df332b84.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_fe8cc61e.jpg)

##  Action Group {#action-group .western style="margin-left: 2in"}



![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_d4f9b542.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_c739247d.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_d7b002c2.jpg)

Now that the action group has been created, I can finish the monitoring
rule.

## Monitoring Alert Cont. {#monitoring-alert-cont. .western}


![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_13629bab.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_35ccc523.jpg)



![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_8f693f0b.jpg)

Once the SQL injection has exploited the database, we can see that the
database request has exceeded the threshold, and I have received an
email notification.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_41d5d1e1.jpg)

Following the link in the email, we can see this spike reflected in the
azure portal.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_f8156f81.jpg)



# WAF Policy {#waf-policy .western}

To create an application gateway within the WAF V2 tier, a WAF policy
needs to be created. This is available from the Web Application Firewall
Policies. This is referred to as a front door WAF policy. The
configuration is straightforward for this policy; however, the Policy
Mode needs to be set to detection to allow reporting to Sentinel while
allowing exploitation of the database. This level of visibility is
needed to produce a detailed incident report.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_5e165930.png)

Azure gives multiple policies for rule sets; these sets are based on the
OWASP top 10 rules and cover the top web application vulnerabilities,
these rules are an excellent basis for web app security, but this
project has two focuses, SQLI (REQUEST-942-APPLICATION-ATTACK-SQLI) and
PHP (REQUEST-933-APPLICATION-ATTACK-PHP)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_f6b4a528.jpg)

Nothing for custom rules or association (this is handled later) so we
can move to review and create

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_6bd38e8e.jpg)

Now that the WAF policy has been created, we can apply it to the
application gateway and WAF.



# Application Gateway Configuration {#application-gateway-configuration .western}

To configure a Web [A]{style="text-transform: uppercase"}pplication
Firewall, the Virtual network needs to have an Application Gateway. The
Application gateway and its accompanying WAF will be located in the
AppGateBoundary, making the web server available to the internet and
helping feed logs to the Sentinel environment.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_e9a3f2a3.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_6e54ee23.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_e0594f94.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_df319821.jpg)

To complete the configuration, a routing rule will need to be created to
send traffic from the WAF to the webserver

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_2c682b45.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_be0a753.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_e5cab149.jpg)

Now that the routing rules have been configured, we have the front end
(our public IP), routing rules (connection from the public to the
backend), and our backend pool. Now that our configuration has been
completed, our web server should now be open to the internet

By going to the public IP of 20.163.243.165, We can see that the default
Apache page is properly configured

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_3a05a5c4.png)



And if we look at the AppGateway/WAF, we can see these requests made to
the gateway

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_99385d59.png)

# Log Analytics Configuration {#log-analytics-configuration .western}

[]{#_MON_1725050224} The primary data source for Sentinel is the Log
Analytics platform; Azure Sentinel can not be configured before Log
Analytics has been enabled. Log Analytics can be configured with the
following command.

```commandline
az monitor log-analytics workspace create -g AzureSentinelProj2022 -n SenNetLAW
```

Log Analytics has now been created

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_4695a2a9.png)


# Sentinel Configuration {#sentinel-configuration .western}

Once Log analytics has been configured, it is possible to create a
sentinel deployment in this virtual network. Going to the Sentinel page,
we will create a Microsoft Sentinel Instance.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_e609b19e.png)

We need to attach a log analytics platform to Sentinel; previously, I
had created SenNetLAW for this deployment of Sentinel.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_bf8e4e6d.png)

Now that log analytics has been selected, the Sentinel deployment has
been completed.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_db996d70.png)

In its current state, the sentinel platform is empty. We need to add
data connectors to the environment to populate the SIEM.


## Data Connectors {#data-connectors .western}

Azure has a wide variety of data connectors built into the platform that
covers everything in the Azure environment. They also offer a content
hub for services and devices such as darktrace or Palo Alto firewalls.
For this project, I will need two separate data connectors, One for the
WAF and one for the server. Data connectors can be found under the
configuration tab on the left.

### WAF {#waf .western}

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_9295711e.png)



Microsoft offers a prebuilt data connector for their WAF solution, which
is called \"Azure Web Application Firewall\" We can connect this data
connector by going to the Open Connector Page

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_2775dd27.jpg)

Each data connector has a different configuration method, but the page
will have a walkthrough for these steps.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a5936ee5.png)

The Application Gateway and WAF policy must have these values configured
for this project.

From here, I will go to the DVWAAppGate and diagnostic settings

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_b9c96ee1.png)

Then add the diagnostic setting

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_5efc3e4a.png)

From here, I Will name the diagnostic setting name as AppGateLogs Select
allLogs and send to log analytics workspace SenNetLAW and save

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_c2ff4541.png)

Now we can see that the AppGate logs have been configured

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_c82cea1b.png)

Now we need to do the same for the WAF Policy

We can find the diagnostic settings under activity log\> export activity


![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_3f02fb95.png)

And we will repeat the same process with the policy by clicking add
diagnostic



![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_7317e3e2.png)

And then selecting the following options in diagnostic settings, I
excluded autoscale because the logs won\'t be relevant for this
deployment. And then we want to save this.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_2a2b173d.png)

Now we can see WAFPolicyLogs in the diagnostic settings

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a3f2de2c.png)

Now we can click back to the WAF data connector to finish the connection

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_7cb1bbd0.png)

From here, we will want to add a Workbook and some templates

For the workbook, I\'ll use the Microsoft Web Application Firewall (WAF)
-- Azure WAF workbook

We can set this up by going to \"go to workbooks gallery\" on the right
side.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_df62dd7d.png)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_49f0a4ba.png)

From here we can view the template to see how the workbook looks

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_1ab45fd2.png)

And if we like the workbook, we can go back and hit save to add this to
our dashboard.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_da022e7b.png)

And the last piece we want to add is the analytics templates.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_8136aea6.png)

I will be using the SQLi Detection Template because that is the focus of
this project, So we click to create a rule, and this will bring us to
the wizard

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_7d0e562c.png)

Set rule logic allows us to modify the configuration and triggers for
the analytics template. This standard configuration will be sufficient.
The Wizard does allow us to run a check on the data that is currently
available in log analytics

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_450e4153.png)


For incident settings, we can turn on Alert grouping to allow Sentinel to connect alerts to start forming a timeline of events

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_8a1fc450.png)

I am not creating an automated response for this because I want to review this incident manually. And I will be aware of it because I am the one executing it. We can move on to the review

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_63359b37.png)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_d5130fa5.png)

Now that the WAF is sending Logs to Sentinel, we can see this data populate and move on to the next connector.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_69908e83.png)

### Virtual Machine {#virtual-machine .western style="margin-left: 0.5in"}

To send logs to Sentinel from a Linux-based server, we will need to use the Syslog data connector

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_5a51b246.png)

Azure makes these easy because the connector can be deployed as an agent on the existing server, reducing the time spent on the configuration. From here, we can click the \"Install agent on azure Linux virtual machine\" and then download and install!

[](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_6ccd253e.png)

We select our VM from this menu

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_8778a0b2.png)

Then we can click connect on the VM\'s page

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_e7d6a580.png)

The agent will begin to run on the selected virtual machine

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_5a0fd21b.png)

Now that we can see that the machine is connected to Syslog, we can go
back and finish the configuration

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_28e0a38.jpg)

Now that the Syslog agent is connected, we can set the logging for the VM

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_e343cc65.png)

We go to open workspace agent and then change the facility name to
Syslog and click apply

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_77820b28.png)

We will go to Next Steps and add our Workbook

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_f3b937e9.png)

And our Virtual machine will start sending logs to Sentinel

# Sentinel Workbook templates {#sentinel-workbook-templates .western}

Playbooks are dashboards specifically designed to show common
information users look for in each service, they are customizable, but
for this project, I want to keep them as the default because they will
pull the information I need.

Now that the data connectors have been configured on the WAF and VM, we
can look at the selected workbooks. If we go to workbooks on the
sentinel dashboard, we will see the active connections.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_de58334.jpg)

And from here, we can open the Syslog playbook and see some data being
populated

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_cbda665d.jpg)

And looking at the WAF workbook, we\'ll see some requests by IP to the
webserver.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_4f763d6b.jpg)


# DVWA Installation {#dvwa-installation .western}



DVWA is a web application built for web application penetration testing.
It is easy to configure and available on GitHub. I choose this web
application because of its ease of deployment, ease of use, and
exploitability. To install DVWA, the server needs a LAMP stack that was
installed previously. Using the bastion in the network, I connect to the
VM, and I can begin the installation

I will disable MySQL.service because it is not
needed on this server, pull a copy of DVWA, and set the default website
to DVWA in Apache.

```commandline
sudo systemctl stop mysql.service | sudo systemctl disable mysql.service

git clone https://github.com/digininja/DVWA.git

echo '<?php phpinfo();?>' | sudo tee -a /var/www/html/phpinfo.php > /dev/null

cd DVWA
cp config/config.inc.php.dist config/config.inc.php
cd
sudo mv DVWA /var/www/html
cd /var/www/html/DVWA

sudo nano /etc/apache2/site-available/000-default.conf

    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/DVWA

sudo nano ./config/config.inc.php

$_DVWA['DB_server'] = 'senvirtdb.mysql.database.azure.com';
$_DVWA['DB_port'] = '3306';
$_DVWA['DB_user'] = 'dvwa';
$_DVWA['DB_password'] = 'p@ssw0rd';
$_DVWA['DB_database'] = 'dvwa';

# Default security level
# Default value for the security level with each session.
# The default is 'impossible'. You may wish to set this to either 'low','medium','high'
$_DVWA['default_security_level'] = 'medium';


sudo systemctl restart apache2
systemctl status apache2
```
Now that the configuration has been completed, we can load the webpage

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_34b5fee9.png)

We now need to initialize the database by clicking create/reset the
database

And we can see the database has been created and the main webpage is
available

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_29cba07c.jpg)

# SQL Injection {#sql-injection .western}

To generate Alerts and even incidents in Sentinel. I need something that
will allow me to exploit the web application and the underlying
database. I picked a relatively small webserver with the hope that it
will send alerts to Sentinel as well regarding service health. While the
networks have been segregated, the web application is still vulnerable
and enumerating and dumping the database should not pose much of a
challenge. By using automated database exploitation, I will be able to
generate a lot of noise from the WAF and the database.

I picked SQLMap to execute the SQL Injection because it\'s a standard
tool for database security testing across many database types. Its ease
of use also helps because it will Enumerate, Exploit, and Report with
limited input. I will also use OWASP Zap to pull a session cookie for
SQLMap to bypass the login screen and begin executing SQL queries on the
database.

The attack starts at 10:15 PM Est on 9/24/2022

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_7f6d61d2.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_8b831799.jpg)

 Now that the cookie has been captured from the
webserver, we can build our SQLMap command and begin enumerating the
database by running the following command and setting the wizard option
to allow SQLMap to automate the attack.

```commandline
sqlmap -u "http://20.163.243.165/vulnerabilities/sqli/" --proxy=http://127.0.0.1:8080
--cookie="PHPSESSID=ueufmufdimuqm9l72lf6on1lkg;security=medium"
--data="id=1&Submnit=Submit""
--wizard
```


![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_ebc1acf.jpg)


![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_60ec0488.jpg)
 
Now that SQLMap has found a vulnerability, we can
see the database is DVWA and scan for tables by running the following.

```commandline
sqlmap -u "http://20.163.243.165/vulnerabilities/sqli/" --proxy=http://127.0.0.1:8080
--cookie="PHPSESSID=ueufmufdimuqm9l72lf6on1lkg;security=medium"
--data="id=1&Submnit=Submit""
--wizard -D dvwa --tables
```
![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_5ea5af31.jpg)

We can see that the DVWA database contains the
table guestbook and users. We want to find users and passwords, so
we\'ll run the same command with the table users and search for columns

```commandline
sqlmap -u "http://20.163.243.165/vulnerabilities/sqli/" --proxy=http://127.0.0.1:8080
--cookie="PHPSESSID=ueufmufdimuqm9l72lf6on1lkg;security=medium"
--data="id=1&Submnit=Submit""
--wizard -D dvwa -T users --columns
```

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_117904ff.jpg)

We can see that the table users contains the
passwords and users. So we\'ll want to dump the users\' table and crack
the passwords by running the following command

```commandline
sqlmap -u "http://20.163.243.165/vulnerabilities/sqli/" --proxy=http://127.0.0.1:8080
--cookie="PHPSESSID=ueufmufdimuqm9l72lf6on1lkg;security=medium"
--data="id=1&Submnit=Submit""
--wizard -D dvwa -T users -dump
```

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_3a081753.jpg)

The DVWA table users have been dumped, and the passwords have been
cracked, revealing that the password for user gordonb is abc123. To
verify this, we can log in to the DVWA website

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_fd29c122.jpg)

And here, we can see that logging in with gordonb was successful.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_5c2c671e.jpg)

The attack ends at 10:23 EST on 9/24/2022

Now that this SQL attack has been successfully executed, we can now view
the attacks in Sentinel


# Sentinel Workbook {#sentinel-workbook .western}

Because a successful SQL Injection has been run against the web
application, the WAF playbooks will begin to populate with data. This
data is broken into several categories by default.



![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_b9890ac5.jpg)

The default dashboard at the beginning will show the basic time this
attack was executed and the type of events the filters have caught.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a9462501.jpg)

Scrolling down further, we can see detailed logs of the traffic that has
flowed through the WAF that matched a pattern in the OWASP 3.2 rules,
and we can see events that are attack specific.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a36359b2.jpg)

In these charts, we can filter logs from the WAF that are matched to an
IP address. And we can see what rule these logs are attached to.


# Sentinel Incident {#sentinel-incident .western}

The playbooks configured show the message\'s detailed views for further
analysis. But now that a SQL injection attack has occurred, the
analytics rule configured earlier will trigger on the same event logs as
seen above. This will now generate an incident.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_3201e202.jpg)


If we open the incidents, we can see that Sentinel has created an
incident in our queue, and we can open this and start looking at the
details.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_83ec35d0.jpg)

Below we can see that the incident has been populated with any
information that Sentinel has deemed related. There are 570 events
currently tied to this incident

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_af3b8e88.jpg)

We can also look at the Rules that have triggered the event and related
information such as IP addresses

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_caf41f1b.jpg)

Sentinel also can visualize the attacks and help show the connections
between events.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_e757d75b.jpg)

Using the information from the incident, we can start to look at this
attack through our workbooks and attempt to resolve this issue.


## WAF Prevention {#waf-prevention .western}

Because this is a vulnerable web application, this won\'t be expressly covered, but we can see the mitigation of turning the WAF from Detection to Prevention and rerun our attack.

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_d4e1e7dd.jpg)

We need to change the WAF policy by clicking change to prevention in the WAF policy

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_ce7c2e7.jpg)

Now that we can see the WAF is in prevention mode and we will rerun the attack

```commandline
sqlmap -u "http://20.163.243.165/vulnerabilities/sqli/" --proxy=http://127.0.0.1:8080
--cookie="PHPSESSID=ueufmufdimuqm9l72lf6on1lkg;security=medium"
--data="id=1&Submnit=Submit""
--wizard 
```
And as we can see, the attack fails immediately

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_972d1caf.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_16c2740d.png)

and if we check the workbook for the WAF, we will see that it has blocked an attempted SQL injection

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a58f3469.jpg)

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_a2e487b5.png)


# Remarks {#remarks .western}


Overall, this was a great project to explore the Sentinel with; the
ability to provision resources within a resource group and then assign a
log workplace and Sentinel Deployment helps keep the information
concise. One of the significant benefits of usage-based services is that
the assets can be grouped and monitored and fed into a global view;
However, I didn\'t do anything of this nature; I think the potential is
an excellent resource for building within large environments. Using the
Azure CLI was very helpful for creating the Networking and the VM; I did
attempt to use the CLI for creating the SQL Server and Database;
however, I was unable to build the database server correctly after
multiple attempts, so I decided to use the web interface to build these.
The application gateway and WAF policy applied were enjoyable to work
with. The Application gateway and WAF fell into the same categories as
these have many options that need to be configured for the service to be
set up correctly. The use of OWASP top 10 was nice to see in action. The
one thing that caught my attention, however, was that the default
workbook for the WAF data connector does not pick up generated alerts.
To have the analytics rules generate an incident, they need to be
tweaked to pick up the alerts; I only did these for the SQL injection
rules, but I imagine this might be an issue across the OWASP rules that
the WAF references.



# Resources {#resources-1 .western}

SQLMap

[<https://sqlmap.org/>]{.underline}

DVWA

[<https://github.com/digininja/DVWA>]{.underline}

## Azure Resources {#azure-resources .western style="margin-left: 0.5in"}

![](/images/gradprojimport/FA2022_CC031_ProjAzure_Gagnon_Thomas_VM_AzureMySQL_Bastion_LogAnalytics_Sentinel_AppGate_WAF_html_be06e2ad.png)