# Synchronization Tool for Sentinel and JIRA
Author: Thijs Lecomte

## Overview
This tool will synchronize incidents between Azure Sentinel and JIRA Service Management using the following tools:
* Azure Logic Apps
* Azure Functions
* Automation for JIRA
* Azure Sentinel Automation Rules
* Azure Key Vault

This tool will do the following:
* Create an incident in JIRA when an incident is created in Sentinel
* Sync the assigned user from JIRA to Sentinel
* Sync the status from JIRA to Sentinel
* Add the URL to the JIRA incident as a comment in Sentinel
* Sync public comments from JIRA to Sentinel

![Overview](Images/Solution%20overview.png)

[Blog post with more background information](https://thecollective.eu/blog/setting-up-a-bidirectional-sync-between-sentinel-and-jira/)

## Implementation
To implement this solution, a few different steps need to be done:
1. Create necessary Service Principals
2. JIRA Configuration
   1. Custom fields
   2. Deploy Automation for JIRA rules (used for sync from JIRA to Azure Sentinel)
3. Deploy the Key Vault and add secrets
4. Deploy Azure Logic Apps (4) through ARM deployment
5. Deploy Azure Function for comment sychronization and add the Powershell code (check the Functions)
6. Create Sentinel Automation Rule

## 1. Create Service Principals
The tool requires a service principal for authentication to different services:
* Authentication to Azure Sentinel
* Authentication to Key Vault to retrieve secrets

### Azure Sentinel Service Principal
This Service Principal needs to have at least Azure Sentinel Operator permissions and it used in the following:
This Service Principal needs to have the 'Secret - Get' Permission on the Key Vault holding all the secrets.
* All Logic Apps
* Comment Sync Function

## 2. Azure Proxy
JIRA does not like long URLs being used for webhooks. For this reason, you may need to set up a azure proxy. To do this, create a Function App running windows, and then once it is set up, you can add a proxy that acts as a shortened URL that points to your sentinel webhook trigger. To do this, click on the Proxy button in the menu of the Function App, and add a proxy pointing to the long azure URL.

## 3. JIRA Configuration
### JIRA Custom Fields
#### Introduction
A lot of the Sentinel specific information is stored inside of Custom Fields in JIRA which need to be created.
This document contains an overview of the different custom fields that are used in the Logic Apps.
All Logic Apps need to be updated with the correct ID's of the fields.

#### Custom Fields overview

| **Field Name** | **Field ID** | **Field Type**|
| --- | --- | --- |
| Organizations | customfield_10002 | Built-in |
| Sentinel Incident URL | customfield_10144 | Url Field |
| Incident ID | customfield_10145 | Text Field (Single line) |
| Closure Comment | customfield_10146 | Text Field (Multiline) |
| Closure Reason | customfield_10047 | Select List (Single choice) |
| Tenant Name | customfield_10149 | Select List (Single Choice) |
| Created At | customfield_10154 | Date Time Picker |
| Att&ck Tactics | customfield_10055 | Select List (Multiple choices) |
| Affected User | customfield_10058 | Text Field (Multiline) |
| Subscription ID | customfield_10162 | Text Field (Singline) |
| Sentinel Resource Group | customfield_10169 | Text Field (Singline) |
| Sentinel Workspace Name | customfield_10070 | Text Field (Singline) |
| Sentinel Workspace ID | customfield_10172 | Text Field (Singline) |
| Sentinel Incident ID | customfield_10172 | Text Field (Singline) |
| Sentinel Incident ARM ID | customfield_10175 | Text Field (Singline) |

The Att&ck Tactics list contains all Sentinel Tactics.
The Closure Reason contains all valid Sentinel Closure Reasons


### JIRA Webhook
In order to synchronize changes from JIRA to Sentinel, A webhook was created that sends updates back to sentinel.

Under administration and system, there is an option to add webhooks. Add a webhook that captures the "Issue: updated" and "Comment: created, updated, deleted" events, and sends them to sentinel via the proxy.


## 4. Deploy Key Vault
The Key Vault is used to store three secrets:
* One for the AAD Service Principal
* One for the Azure Sentinel Service Principal
* One for the JIRA Secret

Create a new Key Vault or deploy an existing one and add the different secrets to it.
The names will need to be provided when deploying the ARM Templates from the Logic Apps.

Edit the Access Policy and assign 'Secret - Get' permissions to the Key Vault Service Principal.

## 5. Deploy Logic Apps
The solution consists of four different Logic Apps:
* Sync Incidents from Sentinel to JIRA (Sync-Incidents.json)
* Sync status from JIRA to Sentinel (Sync-Status.json)

This will enable a two way synchronization between JIRA and Sentinel. Not all Logic Apps are mandatory, you can deploy the ones your organization needs.

Each Logic App can be deployed using the provided ARM templates.
After deploying the Logic Apps, you can copy the HTTP trigger URLs and paste them in the JIRA Automation Rules.

### Custom Fields Configuration
A lot of JIRA Custom fields are used within these Logic Apps. It's important to create these custom fields in your own JIRA environment and change the correct ID's in the Logic Apps.
For more information about the different custom fields used, please check the JIRA Configuration.

### Sync Incidents from Sentinel to JIRA 

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fsanjay900%2FAzure-Sentinel%2Fmaster%2FPlaybooks%2FSync-IncidentsWithJIRA%2FPlaybooks%2FSync-Incidents.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fsanjay900%2FAzure%2FAzure-Sentinel%2Fmaster%2FPlaybooks%2FSync-IncidentsWithJIRA%2FPlaybooks%2FSync-Incidents.json)

This Logic App will create a new incident in JIRA when an incident in Sentinel is created.
It uses the 'Incident Trigger' from Sentinel and is triggered by an Automation Rule (see Sentinel Configuration).
This Logic App does the following:
* Retrieve all the incident information
* Retrieve the Tactics in this incident
* Retrieve the details from the Account Entity
* Check which customer this incident is coming from (to add the right organization in JIRA)
* Create JIRA ticket through an API call
* Add a link to the JIRA ticket on the sentinel ticket

It uses two connections:
* One connection to Sentinel through a Service Principal (to be configured when deploying the Logic App)
* One connection to a Key Vault to retrieve the JIRA API Key (also configured when deploying the Logic App)

In order to correlate the right add the incident in JIRA to the correct Organization, we use a switch and determine the correct organization based on the originate Subscription ID.
Add a case per customer and add the right Subscription ID, Customer name and Organization ID.
If you do not use organizations in JIRA, you can remove the switch.
![Switch](Images/Azure%20-%20Switch%20Organization.png)

### Sync status from JIRA to Sentinel

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fsanjay900%2FAzure-Sentinel%2Fmaster%2FPlaybooks%2FSync-IncidentsWithJIRA%2FPlaybooks%2FSync-Status.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fsanjay900%2FAzure%2FAzure-Sentinel%2Fmaster%2FPlaybooks%2FSync-IncidentsWithJIRA%2FPlaybooks%2FSync-Status.json)

This Logic App will change the status in Sentinel when the status has been changed in JIRA. This also includes the assignee and comments.
It uses an HTTP trigger which is triggered from a JIRA Automation Rule.
It's important you use the same closure reason in JIRA as the ones in Sentinel, otherwise the sync will fail.

It uses one connections:
* One connection to Sentinel through a Service Principal (to be configured when deploying the Logic App)

## 6. Deploy Azure Function

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fsanjay900%2FAzure-Sentinel%2Fmaster%2FPlaybooks%2FSync-IncidentsWithJIRA%2FPlaybooks%2FSync-Status.json)
[![Deploy to Azure Gov](https://aka.ms/deploytoazuregovbutton)](https://portal.azure.us/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fsanjay900%2FAzure%2FAzure-Sentinel%2Fmaster%2FPlaybooks%2FSync-IncidentsWithJIRA%2FPlaybooks%2FSync-Status.json)

To sync incident comments from JIRA to Azure Sentinel an Azure Function is used. This Function App contains one Powershell Function.
There are two types of comments in JIRA: internal and public comments. This script will only sync the public comments, so that customers don't have access to the internal ones.

The code for this function can be found [in this repository](Functions/Sync-Comment.ps1).
This Function uses a managed identity to authenticate to the Key Vault.

Deploy a new Powershell Azure Function app and create one Function with an HTTP trigger.
Paste the code from the PS1 file in the Function and change the following variables:
* jiraUser => Used for authentication to JIRA
* jiraSecretURL => The URL to the JIRA Secret in our Key Vault
* sentinelSecretURL => The URL to the Sentinel Secret in our Key Vault
* tenantID => Tenant ID for the Sentinel Service Principals
* clientID => Client ID for the Sentinel Service Principal

Enable the Managed Identity (Identity => System Assigned).
Provide 'Secret - Get' permissions to the Function app in the Access Policies of the Key Vault.

## 7. Sentinel Automation Rule
In order to trigger the Logic App that creates incidents in JIRA, we use an Automation Rule in Sentinel.
This will enable us to trigger a Logic App (Playbook) each time an incident is created.

Navigate to your Sentinel workspace and choose 'Automation', after this click 'Create Rule'.
Provide a name for the rule and for conditions choose 'Analytics Rule Name contains All'. This will trigger out Logic app each time an incident is created.
If you only want to sync certain incidents, choose the right condition.

For actions, choose 'Run Playbook' and select the 'Sync-Incidents' Playbook.
![Automation Rule](Images/Sentinel%20-%20Automation%20Rule.png)

## Conclusion
After this solution, you are able to work on Azure Sentinel incidents while staying in your trusted ITSM tool.
This tool should take care of any previous synchronization issues between the JIRA and Sentinel.

For any issues with the tool or requested improvements, feel free to create an issue on Github or contact me through [Twitter](https://twitter.com/thijslecomte).
