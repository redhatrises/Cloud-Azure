# Azure Virtual Desktop Deployment with Microsoft Intune

This guide provides instructions for deploying Azure Virtual Desktop (AVD) with Microsoft Intune for automated deployment of the CrowdStrike Falcon Sensor to session hosts.

## Prerequisites

- **Azure Subscription** with appropriate permissions to set up and configure Intune as well as Azure Virtual Desktop infrastructure
- **Azure Virtual Network** properly configured to allow the CrowdStrike Falcon Sensor to communicate with the CrowdStrike cloud
- **Microsoft Entra ID** configured for automatic MDM enrollment

## Setup Steps

### 1. Configure Azure Virtual Desktop Infrastructure

Configure your Azure Virtual Desktop infrastructure as desired by your organization.

See the official Microsoft documentation: [Azure Virtual Desktop Documentation](https://learn.microsoft.com/en-us/azure/virtual-desktop/)

### 2. Deploy CrowdStrike Falcon Sensor via Intune

Follow the detailed steps in the [Installing the CrowdStrike Falcon Sensor using Microsoft Intune](https://github.com/CrowdStrike/Cloud-Azure/tree/main/intune) guide to create and deploy the CrowdStrike Falcon Sensor Intune package to your AVD session hosts.

### 3. Enroll Azure Virtual Desktop Hosts in Intune

Ensure that Azure Virtual Desktop hosts are enrolled in Intune management, either through automatic enrollment or manual configuration.

See the official Microsoft documentation: [Azure Virtual Desktop Management](https://learn.microsoft.com/en-us/azure/virtual-desktop/management)
