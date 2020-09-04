
# DEPLOYMENT SCENARIO
For this deployment example,a web application service with a pair if vThunder VM's are deployed in one region using two available domains for redundancy in Oracle Cloud Infrastructure (OCI).

![Deployment Scenario](./images/Deployment_Senario.png)
_Figure 1:_ Example deployment topology and network information

# DEPLOYMENT PREREQUISITES
To deploy vThunder ADC for a business application running in OCI, the user needs the following:
* Oracle Cloud Infrastructure accounts and access information
* vThunder ADC (image available in the Oracle Cloud Marketplace)
* vThunder License
* SSH key pair for SSH and console access to vThunder
* Oracle API Public and Private Key pair

# CONFIGURATION STEPS OVERVIEW
The high-level configuration steps of this example deployment are as follows:
1. [Prepare Oracle API Public and Private Keys and A10 Keys for login and HA](./ssh_keys.md)
1. [Oracle Cloud configuration (OCI)](./oci_config.md)
   1. [Create Virtual Cloud Network (VCN)](./oci_config.md#creaatevcn)
   1. Configure Management and Server subnets
   1. Create Public network
   1. Modify VCN Security Policy
1. [Deploy two vThunder ADC instances](./deploy_a10.md)
1. [Configure vThunder ADC](./config_a10.md)
  1. General and interfaces
  1. High-availability (VRRP-A and failover) configuration
  1. SLB virtual service (VIP) configuration

---
1. [[API and SSH Key Preparation](./ssh_keys.md#sshkey)](#api-and-ssh-key-preparation)   
     1. [[Set up an Oracle Cloud Infrastructure API Signing Key](./ssh_keys.md#ociapikey)](#set-up-an-oracle-cloud-infrastructure-api-signing-key)   
     1. [[Prepare vThunder SSH keys](./ssh_keys.md#a10sshkey)](#prepare-vthunder-ssh-keys)   
1. [[Configure Oracle Cloud](./oci_config#configoci)](#configure-oracle-cloud)   
  1. [[Create Virtual Cloud Network (VCN)](./oci_config#create_vcn)](#create-virtual-cloud-network-vcn)   
     1. [[Create VCN](./oci_config#createvcn)](#create-vcn)   
     1. [[Modify Management and Server subnets](./oci_config#modifymgmtsvrnet)](#modify-management-and-server-subnets)   
  1. [[Create Public NETWORK](./oci_config#createpublicnet)](#create-public-network)   
  1. [[Modify VCN Security Policy](./oci_config#modifysecpol)](#modify-vcn-security-policy)   
  1. [[Oracle Cloud Infrastructure CLI A10 configuration file](./oci_config#ociconfigfile)](#oracle-cloud-infrastructure-cli-a10-configuration-file)   
  1. [[Add A10 Floating IP to OCI](./oci_config#ocifloating)](#add-a10-floating-ip-to-oci)   
1. [[Create a10 vThunder instances](./deplooy_a10#creaatea10instance)](#create-a10-vthunder-instances)   
   1. [[Create Primary ADC instance](./deplooy_a10#priadc)](#create-primary-adc-instance)   
      - [[ATTACH VNICs to the ADC](./deplooy_a10#attachprivnic)](#attach-vnics-to-the-adc)   
   1. [[Create Secondary ADC instance](./deplooy_a10#secadc)](#create-secondary-adc-instance)   
      - [[ATTACH VNICs to the ADC](./deplooy_a10#attachsecvnic)](#attach-vnics-to-the-adc)  
1. [[A10 vThunder Configuration](./config_a10#configvthunder)](#a10-vthunder-configuration)   
   1. [[Primary vThunder - Configuration](./config_a10#configpri)](#primary-vthunder-configuration)   
      1. [[DNS and Hostname Configuration](./config_a10#configpridnshost)](#dns-and-hostname-configuration)   
      1. [[Network Configuration](./config_a10#configprinetwork)](#network-configuration)   
   1. [[Secondary vThunder - Configuration](./config_a10#configsec)](#secondary-vthunder-configuration)   
      1. [[Configure DNS and Hostname](./config_a10#configsecdnshost)](#configure-dns-and-hostname)   
      1. [[Network Configuration](./config_a10#configsecnetwork)](#network-configuration)   
   1. [[Redundancy Configuration](./config_a10#redundancy)](#redundancy-configuration)   
      1. [[Import API Private Key and Cloud Config File](./config_a10#redundancyconfig)](#import-api-private-key-and-cloud-config-file)   
      1. [[Copy SSH Key to Primary vThunderADC](./config_a10#redundancykey)](#copy-ssh-key-to-primary-vthunderadc)   
      1. [[High Availability (VRRP-A) Configuration (HA)](./config_a10#configha)](#high-availability-vrrp-a-configuration-ha)   
      1. [[Add Floating IP to Oracle Cloud](./config_a10#configocifloat)](#add-floating-ip-to-oracle-cloud)   
