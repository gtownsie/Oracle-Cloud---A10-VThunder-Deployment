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
API keys are required to perform the VRRP-A failover process in an Oracle Cloud Infrastructure deployment. vThunder supports unicast-based VRRP-A to provide redundancy when an active vThunder goes down for any reason. In the Oracle Cloud environment, a public IP address is assigned for a VIP as a secondary IP on the uplink / gateway facing interface. The secondary public IP address(es) have to be moved from the failed vThunder to a new active vThunder when the failover is triggered. The A10 vThunder in the Oracle Cloud implements this workflow and can automatically move the VIP address(es) and other floating IP addresses to the new active vThunder using API-based Oracle functions.
The following files need to be prepared before starting the vThunder configuration.

* API key pair to create API signing key. For example,
  * Private key: oci_api_key.pem (RSA 2K key, PEM format)
  * Public key: oci_api_key_pub.pem

## Set up an Oracle Cloud Infrastructure API Signing Key for Use with Oracle Functions
Before using Oracle Functions, you have to set up an Oracle Cloud Infrastructure API signing key.

The instructions in this topic assume:

  * you are using Linux
  * you are following Oracle's recommendation to provide a passphrase to encrypt the private key

To set up an API signing key:

1. Log in to a Linux workstation.
1. In a terminal window, confirm that the `~/.oci` directory does not already exist. For example, by entering:
`ls  ~/.oci`
1. Assuming the `~/.oci` directory does not already exist, create it. For example, by entering:
`mkdir ~/.oci`
1. Generate a private key encrypted with a passphrase that you provide by entering:
`$ openssl genrsa -out ~/.oci/<private-key-file-name>.pem -aes128 2048`
where `<private-key-file-name>` is a name of your choice for the private key file (for example, `john_api_key_private.pem`).

For example:
```
$ openssl genrsa -out ~/.oci/john_api_key_private.pem -aes128 2048
'Generating RSA private key, 2048 bit long modulus'
....+++
....................................................................+++
e is 65537 (0x10001)
Enter pass phrase for /Users/johndoe/.oci/john_api_key_private.pem:
```
When prompted, enter a passphrase to encrypt the private key file. 
* Be sure to make a note of the passphrase you enter, as you will need it later.  
When prompted, re-enter the passphrase to confirm it.

Confirm that the private key file has been created in the directory you specified. For example, by entering:
```
$ ls -l ~/.oci/john_api_key_private.pem

-rw-r--r-- 1 johndoe staff 1766 Jul 14 00:24 /Users/johndoe/.oci/john_api_key_private.pem
```
Change permissions on the file to ensure that only you can read it. For example, by entering:
```
$ chmod go-rwx ~/.oci/john_api_key_private.pem
```
Generate a public key (in the same location as the private key file) by entering:
```
$ openssl rsa -pubout -in ~/.oci/<private-key-file-name>.pem -out ~/.oci/<public-key-file-name>.pem
```
where:

`<private-key-file-name>` is what you specified earlier as the name of the private key file (for example, `john_api_key_private.pem`)
`<public-key-file-name>` is a name of your choice for the public key file (for example, `john_api_key_public.pem`)

For example:
```
$ openssl rsa -pubout -in ~/.oci/john_api_key_private.pem -out ~/.oci/john_api_key_public.pem

Enter pass phrase for `/Users/johndoe/.oci/john_api_key_private.pem`:
```
When prompted, enter the same passphrase you previously entered to encrypt the private key file.
Confirm that the public key file has been created in the directory you specified. For example, by entering:
````
$ ls -l ~/.oci/
-rw------- 1 johndoe staff 1766 Jul 14 00:24 john_api_key_private.pem
-rw-r--r-- 1 johndoe staff 451 Jul 14 00:55 john_api_key_public.pem
```
Copy the contents of the public key file you just created. For example, by entering:
```
$ cat ~/.oci/john_api_key_public.pem | pbcopy
```
Having created the API key pair, upload the public key value to Oracle Cloud Infrastructure:

Log in to the Console as the Oracle Cloud Infrastructure user who will be using Oracle Functions to create and deploy functions.

1. In the top-right corner of the Console, open the **Profile** menu (User menu icon) and then
1. Click **User Settings** to view the details.
1. On the **API Keys** page, click Add **Public Key**.

Paste the public key's value into the window and click Add.

The key is uploaded and its fingerprint is displayed (for example, `d1:b2:32:53:d3:5f:cf:68:2d:6f:8b:5f:77:8f:07:13`).

Note the fingerprint value. You'll use the fingerprint in a subsequent configuration task.
