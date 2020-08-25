# DEPLOYMENT SCENARIO
For this deployment example,a web application service with a pair if vThunder VM's are deployed in one region using two available domains for redundancy in Oracle Cloud Infrastructure.

![Deployment Scenario](./images/Deployment_Senario.png)
_Figure 7:_ Example deployment topology and network information

#DEPLOYMENT PREREQUISITES
To deploy vThunder ADC for a business application running in Oracle Cloud Infrastructure, the user needs the following:
* Oracle Cloud Infrastructure accounts and access information
* vThunder ADC (image available in the Oracle Cloud Marketplace)
* vThunder License Prepare appropriate license
* SSH key pair for SSH and console access to vThunder and other Linux VM instances hosted in Oracle Cloud.
* Create a pair of Oracle API Public and Private Keys

# CONFIGURATION STEPS OVERVIEW
The high-level configuration steps of this example deployment are as follows:
1. Prepare API keys (used for HA failover operation)
1. Prepair Oracle API Public and Private Keys
1. Define and set VCN and subnets in Oracle Cloud
1. Install two vThunder ADC instances
1. Configure vThunder ADC
   1. General and interfaces
   1. High-availability (VRRP-A and failover) configuration
   1. SLB virtual service (VIP) configuration
# API KEYS AND CONFIG FILE PREPARATION
API keys are required to perform the VRRP-A failover process in an Oracle Cloud Infrastructure deployment. vThunder supports
unicast-based VRRP-A to provide redundancy when an active vThunder goes down for any reason. In the Oracle Cloud environment,
a public IP address is assigned for a VIP as a secondary IP on the uplink / gateway facing interface. The secondary public IP
address(es) have to be moved from the failed vThunder to a new active vThunder when the failover is triggered. The A10 vThunder in
the Oracle Cloud implements this workflow and can automatically move the VIP address(es) and other floating IP addresses to the
new active vThunder using API-based Oracle functions.
The following files need to be prepared before starting the vThunder configuration.
* API key pair to create API signing key. For example,
  * Private key: oci_api_key.pem (RSA 2K key, PEM format)
  * Public key: oci_api_key_pub.pem
