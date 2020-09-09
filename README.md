
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
 1. [API and SSH Key Preparation](./ssh_keys.md#sshkey)
    1. [Set up an Oracle Cloud Infrastructure API Signing Key](./ssh_keys.md#ociapikey)
    1. [Prepare vThunder SSH keys](./ssh_keys.md#a10sshkey)
 1. [Configure Oracle Cloud](./oci_config.md#configoci)
    1. [Create Virtual Cloud Network (VCN)](./oci_config#create_vcn)
       1. [Create VCN](./oci_config#createvcn)
       1. [Modify Management and Server subnets](./oci_config.md#modifymgmtsvrnet)
    1. [Create Public NETWORK](./oci_config.md#createpublicnet)
    1. [Modify VCN Security Policy](./oci_config.md#modifysecpol)   
    1. [Oracle Cloud Infrastructure CLI A10 configuration file](./oci_config.md#ociconfigfile)]
    1. [Add A10 Floating IP to OCI](./oci_config.md#ocifloating)
 1. [Create a10 vThunder instances](./deplooy_a10.md#creaatea10instance)
 1. [Create Primary ADC instance](./deplooy_a10.md#priadc)
    - [ATTACH VNICs to the ADC](./deplooy_a10.md#attachprivnic)
 1. [Create Secondary ADC instance](./deplooy_a10.md#secadc)
    - [ATTACH VNICs to the ADC](./deplooy_a10.md#attachsecvnic)
 1. [Primary vThunder - Configuration](./config_a10.md#primary-vthunder-configuration)   
    1. [DNS and Hostname Configuration](./config_a10.md#dns-and-hostname-configuration)   
    1. [Network Configuration](./config_a10.md#network-configuration)   
 1. [Secondary vThunder - Configuration](./config_a10.md#secondary-vthunder-configuration)   
    1. [Configure DNS and Hostname](./config_a10.md#configure-dns-and-hostname)   
    1. [Network Configuration](./config_a10.md#network-configuration)   
 1. [Redundancy Configuration](./config_a10.md#redundancy-configuration)   
    1. [Import API Private Key and Cloud Config File](./config_a10.md#import-api-private-key-and-cloud-config-file)   
    1. [Copy SSH Key to Primary vThunderADC](./config_a10.md#copy-ssh-key-to-primary-vthunderadc)   
    1. [High Availability Configuration](./config_a10.md#high-availability-configuration)   
    1. [Add Floating IP to Oracle Cloud](#add-floating-ip-to-oracle-cloud)   
