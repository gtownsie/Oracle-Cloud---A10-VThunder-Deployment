
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
1. Prepare Oracle API Public and Private Keys
1. Prepare API keys (used for HA failover operation)
1. Define and set VCN, subnets, and security rules in Oracle Cloud
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
```
$ openssl genrsa -out ~/.oci/<private-key-file-name>.pem -aes128 2048
```
where `<private-key-file-name>` is a name of your choice for the private key file (for example, `oci_api_key.pem`).

For example:
```
$ openssl genrsa -out ~/.oci/oci_api_key.pem -aes128 2048
'Generating RSA private key, 2048 bit long modulus'
....+++
....................................................................+++
e is 65537 (0x10001)
Enter pass phrase for /Users/johndoe/.oci/oci_api_key.pem:
```
When prompted, enter a passphrase to encrypt the private key file.
* Be sure to make a note of the passphrase you enter, as you will need it later.
When prompted, re-enter the passphrase to confirm it.

Confirm that the private key file has been created in the directory you specified. For example, by entering:
```
$ ls -l ~/.oci/oci_api_key.pem

-rw-r--r-- 1 johndoe staff 1766 Jul 14 00:24 /Users/johndoe/.oci/oci_api_key.pem
```
Change permissions on the file to ensure that only you can read it. For example, by entering:
```
$ chmod go-rwx ~/.oci/oci_api_key.pem
```
Generate a public key (in the same location as the private key file) by entering:
```
$ openssl rsa -pubout -in ~/.oci/<private-key-file-name>.pem -out ~/.oci/<public-key-file-name>.pem
```
where:

`<private-key-file-name>` is what you specified earlier as the name of the private key file (for example, `oci_api_key.pem`)
`<public-key-file-name>` is a name of your choice for the public key file (for example, `oci_api_key_pub.pem`)

For example:
```
$ openssl rsa -pubout -in ~/.oci/oci_api_key.pem -out ~/.oci/oci_api_key_pub.pem

Enter pass phrase for `/Users/johndoe/.oci/oci_api_key.pem`:
```
When prompted, enter the same passphrase you previously entered to encrypt the private key file.
Confirm that the public key file has been created in the directory you specified. For example, by entering:
```
$ ls -l ~/.oci/
-rw------- 1 johndoe staff 1766 Jul 14 00:24 oci_api_key.pem
-rw-r--r-- 1 johndoe staff 451 Jul 14 00:55 oci_api_key_pub.pem
```
Copy the contents of the public key file you just created. For example, by entering:
```
$ cat ~/.oci/oci_api_key_pub.pem | pbcopy
```
Having created the API key pair, upload the public key value to Oracle Cloud Infrastructure:

Log in to the Console as the Oracle Cloud Infrastructure user who will be using Oracle Functions to create and deploy functions.

1. In the top-right corner of the Console, open the **Profile** menu (User menu icon) and then
1. Click **User Settings** to view the details.
1. On the **API Keys** page, click Add **Public Key**.

Paste the public key's value from the `oci_api_key_pub.pem` into the window and click Add.

The key is uploaded and its fingerprint is displayed (for example, `d1:b2:32:53:d3:5f:cf:68:2d:6f:8b:5f:77:8f:07:13`).

> **Note the fingerprint value. You'll use the fingerprint in a subsequent configuration task.**

## Prepare SSH keys
During the vThunder deployment an SSH Key pair is required to allow SSH access to the instances after deployment.  The following steps will outline the process to generate the key pair using the `puttygen` utility.
1. Install putty on your workstation from the putty site:  https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
1. After the installation is complete run the `puttygen` utility
1. Select `Key` dropdown and verify that it is set to `SSH-2 RSA`
1. For the type of key to generate, select `RSA` with `2048` bits
1. Select the `Generate` button and move the mouse in the empty area until the key has been successfully generated.
![PuTTY Key Generator](./images/putty_generate.png)
1. Click the `Save public key` and save the file as `ssh_key.pub`
1. Click the `Save Private Key` and save the priate key as `ssh_key_priv.ppk` NOTE:  the .ppk file is used by Putty
1. Under the Key section, select the text in the box labeled `Public key for pasting into OpenSSH authorized_keys file`
   1. Right-Click the window and `select-all`
   </BR>![PuTTY OpenSSH authorized_keys](./images/putty_authorized_keys.png)
   1. Paste the text into a notepad document and save it in the same folder as the other keys.
   authorized_keys_notepad.png
   1. Save the file as `authorized_keys.pub`
   ![PuTTY OpenSSH Notepad](./images/authorized_keys_notepad.png)
1. Select the `Conversions` dropdown and select `export OpenSSH key` and save the file as `ssh_key` with no extension.



# Configure Oracle Cloud
## Create Virtual Cloud Network (VCN)
The next step is to create the VCN within Oracle Cloud.  Table 1 reflects the VCN network and the sub-networks contained within the VCN.

**Table 1:  Example VCN and Subnet Assignement**

Components|Name|Value|Notes
--------------|--------------|--------------|--------------
Region|US-West PHoenix||
Available Domains|PHX-AD-1, PHX-AD-2, PHX-AD-3||
VCN|VCN-a10demo|10.0.0.0/20| Main Network that contains subnets below
Subnet|Management|10.0.0.0/24|Public/Regional
 -|Public|10.0.1.0/24|Public/Regional
 -|Server|10.0.10.0/24|Private/Regional
 -|HA|10.0.3.0/24|Private/Regional

### Create VCN
1. Login to the Oracle Cloud web interfaces
1. Select the "hamburger" menu dropdown in the upper left corner, Select `Networking/Virtual Cloud Networks`
1. Click on the `Start VCN Wizard` and
![Create VCN](./images/vcn_wizard_start.png)
1. Fill out the form with the following parameters:
   1. Name:  `VCN_a10demo`
   1. VCN CIDR BLOCK: `10.0.0.0/20`
   1. Public Subnet CIDR Block:  `10.0.0.0/24` *NOTE: This will become the Management Network*
   1. Private Subnet CIDR Block:  `10.0.2.0/24` *NOTE: This will become the Server Network*
   ![Create VCN](./images/vcn_wizard_configuration.png)
1. Select `Next`
1. Select `Create`

### Modify Management and Server subnets
1. Go into the `VCN_a10dmo` configuration page
1. Select `Public Subnet-VCN_a10demo`
1. Select `Edit`
1. Change Name to `Management_Network` and `Save Changes`
   ![Create VCN](./images/vcn_modify_management.png)
1. Select `Pr Subnet-VCN_a10demo`
1. Select `Edit`
1. Change Name to `Server_Network` and `Save Changes`
   ![Create VCN](./images/vcn_modify_server.png)
1.  Choose `Default Security List for VCN_a10demo`
   ![add ingress rule](./images/add_ingress_rule.png)
1.  `add ingress rule` with the following settings:
   1.  Source type:  `CIDR`
   1.  Source CIDR:  `0.0.0.0/0`
   1.  IP Protocol:  `All Protocols`

   ![add ingress rule](./images/add_ingress_rule_1.png)
1. `Save Changes`

## Create Public NETWORK
1. Create Route Table by going into Route tables under resources and choosing `Create Route Table`
![Create VCN](./images/public_network_route_table.png)
1. Go the the 'Subnets' screen and select `Create Subnet`
1. Create the Public Network using the following configuration:
   1. Name:  `Public_Network`
   1. Subnet Type:  `Regional`
   1. CIDR BLOCK: `10.0.1.0/24`
   1. Route Table: `Default Route Table for VCN_a10demo`
   1. Subnet Access:  `Public Subnet`
   1. DHCP Options: `Default DHCP Options for VCN_a10demo`
   1. Security List: `Default Security List for VCN_a10demo`
   1. Choose `Create Subnet`
![Create VCN](./images/create_public_network.png)
Once completed the Subnets for the VCN will reflect the following:
![VCN Configuration ](./images/vcn_configuration.png)

### Modify VCN Security Policy
By default the VCN security policy only allows SSH, ICMP Type 3 code 4, and ICMP type 3 from the VCN main Net block (10.0.0.0/20).  *This policy also applies to device to device connectivity within the VCN subnets*.  For this lab the security policy is set to ANY/ANY all protocols.

> ***THIS IS NOT RECOMMENDED FOR A PRODUCTION ENVIRONMENT  ONCE THE CONFIGURATION IS COMPLETE PLEASE FOLLOW YOUR COMPANY STANDARDS FOR SECURITY POLICIES***

To modify the security policy, follow the following steps:
1.  From the VCN configuration screen, under Resources, select `Security Lists`
1.  Choose `Default Security List for VCN_a10demo`
![add ingress rule](./images/add_ingress_rule.png)
1.  `add ingress rule` with the following settings:
    1.  Source type:  `CIDR`
    1.  Source CIDR:  `0.0.0.0/0`
    1.  IP Protocol:  `All Protocols`
![add ingress rule](./images/add_ingress_rule_1.png)
1. `Save Changes`

## Oracle Cloud Infrastructure CLI configuration file (txt format)
When the vThunder devices are configured in a reduant pair, they must have the ability to communicate with OCI to move IP address beween the vThunderADC-1 and vThunderADC-2.  When assinging IP addresses to vnics, OCI only allows an IP address to be assigned to one instance and 1 vnic at a time.  When a failover occurs, the vThunder instances send a API call to OCI to remove the Virtal and floating IP address from the Primary to the secondary device.  The config file below provides the VThunder instances the credentials and information to communicate with OCI.  This configuration file will be used later in the implementation process.

To build the `config` document the following information is needed:

* User = User account OCID
* Fingerprint = Public API key fingerprint that was uploaded in the previous step.
* Key file = full path of private API key file on the vThunder
* Pass phrase = add pass phrase if the private key is generated with a pass phrase (optional)
* Tenancy = OCID of the tenancy in which the user will be creating and deploying functions.
* Region = Region identifier of the Oracle Cloud Infrastructure in which the user is deploying services.

Below is an example of a ‘config’ file below to be imported to the vThunder. See the section later in the document for the detailed procedures.
```
    [DEFAULT]
    user=ocid1.user.oc1..aaaaaaa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7s8t9u0
    fingerprint=1b:2c:3d:4e:5f:6g:7h:8i:9j:0k:1l:2m:3n:4o:5p:6q
    key_file=/a10data/cloud/oci_api_key.pem
    pass_phrase=
    tenancy=ocid1.tenancy.oc1..aaaaaaaagz11111111bbbbbbbb2222222cccccccc3333333333
    region=us-ashburn-1
```
1. To find the `User` account ocid, go to the profile page by clicking on the profile icon in the upper right corner of the OCI window.
![OCID](./images/ocid.png)
1. Use the `fingerprint` value that was collected in  the *Set up an Oracle Cloud Infrastructure API Signing Key for Use with Oracle Functions* section.
1. The Key file, leave the same as in the example.
1. To locate the tenancy value,  go to the tenancy page by clicking on the profile icon in the uppper right corner of the OCI window.
![Tenancy OCID](./images/tenancy-ocid.png)
1. To find the region name Click on the region dropdown at the top of the OCI page and choose Manage Regions.
1. Find the region name that is used for your deployment and copy the `Region identifier`
![Region](./images/region.png)

# Create a10 vThunder instances
Table 2: VTHUNDER ADC INSTANCE AND NETWORK CONFIGURATION SPECIFICATIONS

-|PRIMARY ADC|SECONDARY ADC|NOTES
---------------|---------------------|--------------------|---------------
Instance Name|vThunderADC-1|vThunderADC-2|
Availability Domain|AD1|AD2|
Instance Shape|VM.Standard 2.4|VM.Standard 2.4|Selected based on VNIC counts (4) required in this deployment
CONFIGURE NETWORKING
VCN Compartment|a10demo|a10demo
VCN|VCN-a10demo|VCN-a10demo
Subnet Compartment|a10demo|a10demo|
Subnet|Management_Network|Management_Network|For mgmt. interface
Public IP assignment|Yes|Yes

## Create Primary ADC instance
Next a the Secondary ADC instance is created using the following settings.

Name|IP Address|Floating IP
---------|---------|---------
Instance Name|vThunderADC-1|
Management Network|DHCP|
Public Network|10.0.1.11|10.0.1.10
Server Network|10.0.2.11|10.0.2.10

1. From the OCI screen, select the dropdown menu in the upper left corner
1. Select `Compute/Instances`
1. Click on `Create Instance` enter the name from Table 2, `vThunderADC-1`
1.Select `Change Image`
![Create Compute Instance](./images/cci.png)
1. Change to `Partner Images` tab and check `A10 vThunder Application Delivery Controller - BYOL`
![Select Image](./images/cci-image.png)
1. Since this is the `vThunderADC-1` select `AD 1` for the Availability Domain
1. Choose `Change Shape`
![Select Shape](./images/cci-ad-shape.png)
1. For `Instance Type` select `Virtual Machine`, `Shape Series` choose `Intel Skylake`, check the box next to `VM Standard 2.4`
![Select Instance Flavor](./images/cci-shape.png)
1. Continue to scroll down the CCI screen to the `Configure Networking` and change `Subnet` to  `Management_Network (Regional)` and validate that `ASSIGN A PUBLIC IP ADDRESS` is selected
![Configure Networking](./images/CCI-networking.png)
1. After the Network configuration, scroll do the `Add SSH Keys` section.  Select `Choose SSH key FILES` and select the file `authorized_keys.pub` file created in the SSH Key section
![Configure Networking](./images/cci-ssh.png)
1.  Choose `Create` to create the instance.

### ATTACH VNICs to the ADC
By default OCI will only deploy the instance with the Subnet defined in the Create Compute Instance Network Configuration.  The following steps will add the Public and Server networks to the Instance.

1. Select `vThunderADC-1` under the instances List
1. Scroll to the bottom of the page and select `Attached VNICs`
![Instance Config VNICs](./images/instance-config-vnics.png)
1. Choose `Create VNIC`
![Create VNICs](./images/vnic-create-1.png)
1. For name enter `Public_VNIC`,Select a Subnet choose `Public_Network`, Private IP address enter `10.0.1.11` and check 'ASSIGN A PUBLIC IP'
![Create Public VNICs](./images/vnic-public-1.png)
1. Save Changes
1. Select `Create VNIC`
1. For name enter `Server_VNIC`,Select a Subnet choose `Server_Network`, Private IP address enter `10.0.2.11` and check 'ASSIGN A PUBLIC IP'
![Create Server VNICs](./images/vnic-server-1.png)
1. At the top of the instance Details screen, select `Reboot` to restart the instance.  Once the instance restarts the remaining network interfaces are available to ACOS.

## Create Secondary ADC instance
Next a the Secondary ADC instance is created using the following settings.

Name|IP Address|Floating IP
---------|---------|---------
Instance Name|vThunderADC-2|
Management Network|DHCP|
Public Network|10.0.1.12|10.0.1.10
Server Network|10.0.2.12|10.0.2.10

1. From the OCI screen, select the dropdown menu in the upper left corner
1. Select `Compute/Instances`
1. Click on `Create Instance` enter the name from Table 2, `vThunderADC-2`
1.Select `Change Image`
![Create Compute Instance](./images/cci-2.png)
1. Change to `Partner Images` tab and check `A10 vThunder Application Delivery Controller - BYOL`
![Select Image](./images/cci-image.png)
1. Since this is the `vThunderADC-2` select `AD 2` for the Availability Domain
1. Choose `Change Shape`
![Select Shape](./images/cci-ad-shape-2.png)
1. For `Instance Type` select `Virtual Machine`, `Shape Series` choose `Intel Skylake`, check the box next to `VM Standard 2.4`.
![Select Instance Flavor](./images/cci-shape.png)
1. Continue to scroll down the CCI screen to the `Configure Networking` and change `Subnet` to  `Management_Network (Regional)` and validate that `ASSIGN A PUBLIC IP ADDRESS` is selected.
![Configure Networking](./images/CCI-networking.png)
1. After the Network configuration, scroll do the `Add SSH Keys` section.  Select `Choose SSH key FILES` and select the file `authorized_keys.pub` file created in the SSH Key section
![Configure Networking](./images/cci-ssh.png)
1.  Choose `Create` to create the instance.

### ATTACH VNICs to the ADC
By default OCI will only deploy the instance with the Subnet defined in the Create Compute Instance Network Configuration.  The following steps will add the Public and Server networks to the Instance.

1. Select `vThunderADC-2` under the instances List
1. Scroll to the bottom of the page and select `Attached VNICs`
![Instance Config VNICs](./images/instance-config-vnics.png)
1. Choose `Create VNIC`
![Create VNICs](./images/vnic-create-1.png)
1. For name enter `Public_VNIC`,Select a Subnet choose `Public_Network`, Private IP address enter `10.0.1.12` and check 'ASSIGN A PUBLIC IP'
![Create Public VNICs](./images/vnic-public-1.png)
1. Save Changes
1. Select `Create VNIC`
1. For name enter `Server_VNIC`,Select a Subnet choose `Server_Network`, Private IP address enter `10.0.2.12` and check 'ASSIGN A PUBLIC IP'
![Create Server VNICs](./images/vnic-server-1.png)
1. At the top of the instance Details screen, select `Reboot` to restart the instance.  Once the instance restarts the remaining network interfaces are available to ACOS.

# A10 vThunder CONFIGURATION
## Primary vThunder - Configure Network Interface
The next step is the congiruation of the data plane network interfaces, default gateway, DNS, and Hostname.

>***NOTE:  The Hostname MUST match the Instance name***

Name|IP Address|Floating IP
---------|---------|---------
Hostname Name|vThunderADC-1|
Management Network|DHCP|
Public Network|10.0.1.11|10.0.1.10
Server Network|10.0.2.11|10.0.2.10

### Configure DNS and Hostname
1. SSH into the public IP address of the vThunderADC-1 instances using the SSH keys created earlier in this document
1. Type `enable` and `config t` to go into configuration mode.
1. Add the DNS server by typing `ip dns primary 8.8.8.8`
1. Add the hostname by typing `hostname vThunderADC-1`
1.  Validate the configuration by issuing a `sh run` command, it should mirror the following (some lines redacted):
    ```
    vThunderADC-1(config)(NOLICENSE)#sh run
    !
    ip dns primary 8.8.8.8
    !
    hostname vThunderADC-1
    !

    ```
1. Validate that the vThunder recognizes the network interfaces by running the `sh interfaces brief`, below is a sample of the output, if only the management interface is shown issue a reboot command, the interfaces are recognized after the vnics are creaated and the instance is rebooted:
    ```
    vThunderADC-1(config)(NOLICENSE)#sh interfaces brief
    Port    Link  Dupl  Speed  Trunk Vlan MAC             IP Address          IPs  Name
    mgmt    Up    auto  auto   N/A   N/A  0000.1700.9792  10.0.0.2/24           1
    1       Disb  None  None   none  1    0200.1703.f7b9  0.0.0.0/0             0
    2       Disb  None  None   none  1    0200.1703.062e  0.0.0.0/0             0
    ```
1. Set the IP address of the Public Network by creating the following VLANS by issuing the following commands *NOTE: VLAN tagging is disabled, the vlan tags themselves are not use.  Any tag number can be used.  In this example the vlan tag was pulled from the `Attached VNICs` menu*:
    ```
    vlan 2125
    untagged ethernet 1
    router-interface ve 2125
    name "Public Network"
    exit
    !
    interface ve 2125
    ip address 10.0.1.11 /24
    exit
    !
    vlan 2126
    untagged ethernet 2
    router-interface ve 2126
    name "Server Network"
    exit
    !
    interface ve 2126
    ip address 10.0.2.11 /24
    exit
    wr mem
    ```
1.  Validate the configuration by running the 'sh interfaces brief' command again.  Below is an example of the output.
    ```
    vThunderADC-1(config)(NOLICENSE)#sh interfaces brief
    Port    Link  Dupl  Speed  Trunk Vlan MAC             IP Address          IPs  Name
    mgmt    Up    auto  auto   N/A   N/A  0000.1700.9792  10.0.0.2/24           1
    1       Disb  None  None   none  2125 0200.1703.f7b9  0.0.0.0/0             0
    2       Disb  None  None   none  2126 0200.1703.062e  0.0.0.0/0             0
    ve2125  Down  N/A   N/A    N/A   2125 0200.1703.f7b9  10.0.1.11/24          1
    ve2126  Down  N/A   N/A    N/A   2126 0200.1703.062e  10.0.2.11/24          1
    ```
1. Enable the ethernet interfaces by running the following commands:
    ```
    interface ethernet 1
    enable
    interface ethernet 2
    enable
    wr mem
    ```
1. To allow synchronization between the two vThunder instances SSH must be allowed on the interfaces.  SSH is the method which A10 synchronizes the configurations.
    ```
    enable-management service ssh
      ve 2125 to 2126
      exit-module
    !
    wr mem
    ```
1. Run the 'sh interfaces brief' command again and the interfaces should reflect the `UP` status
1. Create a default gateway `ip route 0.0.0.0 /0 10.0.1.1`

## Secondary vThunder - Configure Network Interface
The next step is the configuration of the data plane network interfaces, default gateway, DNS, and Hostname.

>***NOTE:  The Hostname MUST match the Instance name***

Name|IP Address|Floating IP
---------|---------|---------
Hostname Name|vThunderADC-2|
Management Network|DHCP|
Public Network|10.0.1.12|10.0.1.10
Server Network|10.0.2.12|10.0.2.10

### Configure DNS and Hostname
1. SSH into the public IP address of the vThunderADC-1 instances using the SSH keys created earlier in this document
1. Type `enable` and `config t` to go into configuration mode.
1. Add the DNS server by typing `ip dns primary 8.8.8.8`
1. Add the hostname by typing `hostname vThunderADC-2`
1.  Validate the configuration by issuing a `sh run` command, it should mirror the following (some lines redacted):
    ```
    vThunderADC-2(config)(NOLICENSE)#sh run
    !
    ip dns primary 8.8.8.8
    !
    hostname vThunderADC-2
    !
    end
    wr mem
    ```
1. Validate that the vThunder recognizes the network interfaces by running the `sh interfaces brief`, below is a sample of the output, if only the management interface is shown issue a reboot command, the interfaces are recognized after the vnics are creaated and the instance is rebooted:
    ```
    vThunderADC-2(config)(NOLICENSE)#sh interfaces brief
    Port    Link  Dupl  Speed  Trunk Vlan MAC             IP Address          IPs  Name
    mgmt    Up    auto  auto   N/A   N/A  0000.1700.9792  10.0.0.4/24           1
    1       Disb  None  None   none  1    0200.1703.f7b9  0.0.0.0/0             0
    2       Disb  None  None   none  1    0200.1703.062e  0.0.0.0/0             0
    ```
1. Set the IP address of the Public Network by creating the following VLANS by issuing the following commands *NOTE: VLAN tagging is disabled, the vlan tags themselves are not use.  Any tag number can be used.  In this example the vlan tag was pulled from the `Attached VNICs` menu*:
    ```
    vlan 2125
    untagged ethernet 1
    router-interface ve 2125
    name "Public Network"
    exit
    !
    interface ve 2125
    ip address 10.0.1.12 /24
    exit
    !
    vlan 2126
    untagged ethernet 2
    router-interface ve 2126
    name "Server Network"
    exit
    !
    interface ve 2126
    ip address 10.0.2.12 /24
    exit
    !
    wr mem
    ```
1.  Validate the configuration by running the 'sh interfaces brief' command again.  Below is an example of the output.
    ```
    vThunderADC-1(config)(NOLICENSE)#sh interfaces brief
    Port    Link  Dupl  Speed  Trunk Vlan MAC             IP Address          IPs  Name
    mgmt    Up    auto  auto   N/A   N/A  0000.1700.9792  10.0.0.2/24           1
    1       Disb  None  None   none  2125 0200.1703.f7b9  0.0.0.0/0             0
    2       Disb  None  None   none  2126 0200.1703.062e  0.0.0.0/0             0
    ve2125  Down  N/A   N/A    N/A   2125 0200.1703.f7b9  10.0.1.11/24          1
    ve2126  Down  N/A   N/A    N/A   2126 0200.1703.062e  10.0.2.11/24          1
    ```
1. Enable the ethernet interfaces by running the following commands:
    ```
    interface ethernet 1
    enable
    interface ethernet 2
    enable
    wr mem
    ```
1. To allow synchronization between the two vThunder instances SSH must be allowed on the interfaces.  SSH is the method which A10 synchronizes the configurations.
```
enable-management service ssh
  ve 2125 to 2126
  exit-module
!
wr mem
```
1. Run the 'sh interfaces brief' command again and the interfaces should reflect the `UP` status
1.  Create a default gateway `ip route 0.0.0.0 /0 10.0.1.1`

# Redundancy Configuration
The next step in the process is to configure redundancy.  It is A10 recommended practice to have a dedicated interface and subnet for an `HA` network.  In some instances, when the vThunder is deployed with 2 interfaces, the first interface is used for management and the second is used for the server and public network in a 1-arm configuration.  Also, if a customer is running a 4 vnic instance and requires the 3 data plane interfaces for production traffic, then utilizing a data plane interface may be needed.  To demonstrate the A10 deployment flexibility, this example will not use an HA vlan/network the Server_Network is used.
> **If a vnic is available follow the steps above and add an additional network  and interface for `HA`**

## IMPORT API PRIVATE KEY AND CLOUD CONFIG FILE TO VTHUNDER ADC
A10 vThunder ADC has a tighter integration with Oracle Cloud Infrastructure using APIs, enabling an ADC high availability deployment. This section describes how to import an API key and cloud config file that are used for the automation of ADC failover workflow.
1. Locate the API private key `(oci_api_key.pem)` prepared in the API Key Preparation section. On the vThunder CLI (config) mode, import the file as `oci_api_key.pem`. By default, this file is stored in the vThunder under the /a10data/cloud/ directory.
```
    vThunderADC-1# conf
    vThunderADC-1(config)#import cloud-creds oci_api_key.pem use-mgmt-port scp://a.b.c.d/root/oci/oci_api_key.pem
    User name []?root
    Password []?
    Done.
    vThunderADC-1(config)#show cloud-creds
    --------------------------------------------------
    Name Permissions
    --------------------------------------------------
    oci_api_key.pem 0400
    --------------------------------------------------
```
    **NOTE: The user can also download the file from a file share service such as Dropbox using the shared download link. Copy and paste the link into the command, as shown below. If the link is not set with a password, the user can use the vThunder login and password (Default user: admin, default password: <Unique ID of the Instance OCID>)**
```
    vThunderADC-1(config)#import cloud-creds oci_api_key.pem use-mgmt-port https://www.dropbox.com/s/qwertyxxxxxxx/oci-config?dl=0
    User name []?admin
    Password []?
    Done.
```
1. Locate the cloud config file (filename: config) prepared in the API Keys Preparation section. On the vThunder CLI (config) mode, import the file as `config`.
```
   vThunderADC-1(config)#import cloud-config config use-mgmt-port scp://a.b.c.d/root/oci/config
   User name []?root
   Password []?
   Done.
   vThunderADC-1(config)#sh cloud-config config
   [DEFAULT]
   user=ocid1.user.oc1..aaaaaaa1b2c3d4e5f6g7h8i9j0k1l2m3n4o5p6q7s8t9u0
   fingerprint=1b:2c:3d:4e:5f:aa:7h:bb:9j:0k:1l:2m:3n:4o:5p:6q
   key_file=/a10data/cloud/oci_api_key.pem
   pass_phrase=tenancy=ocid1.tenancy.oc1..aaxxaaaagz11111111bbbbbbdd2222222cccddccc333d3333
   region=us-ashburn-1
```
> NOTE: Key_file name (e.g. oci_api_key.pem) in the config must match the user’s cloud-cred key file imported earlier.
## Copy SSH Key to Primary vThunderADC
This is an optional step to synchronize VIP configuration of vThunder ADC-1 to standby vThunder ADC-2.
    >NOTE: If the user prefers to configure VIPs on vThunder ADC-2 manually, please skip this step.

    >NOTE: Configure sync command covers most of SLB configuration, security policies except routing and interface settings.

    Before running `configure sync` command, the user will need to import the SSH private key on to vThunder ADC-1 as it’s required for SSH authentication.

    Locate the SSH private key (**ssh_key_priv.pem**) prepared in the Deployment Prerequisites section. On the vThunder CLI (**config**) mode, import the SSH private key file “ssh_key_priv.pem”.
```
    vThunderADC-1(config)#import key sync_ssh_priv use-mgmt-port scp://192.168.0.254/root/oci/ssh_key_priv.pem
    User name []?admin
    Password []?
    Done.
    vThunderADC-1(config)#sh pki cert
    Name: sync_ssh_priv Type: key [Unbound]
```
    >NOTE: If this operation failed with an error related to key file format, please try to convert the private key to OpenSSH format (Old or New) again, then import it again.

## HIGH AVAILABILITY (VRRP-A) CONFIGURATION
In this section, you will configure the device redundancy feature, VRRP-A, on both vThunder ADCs. Here is the list of the CLI commands to form the VRRP-A and make vThunderADC-1 as an active device. You can copy and paste the following config, with appropriate modification if needed, to your vThunder ADCs.

**vThunderADC-1 VRRP-A CONFIGURATION EXAMPLE**
```
vrrp-a common
device-id 1
set-id 1
enable
exit
!
vrrp-a vrid 0
floating-ip 10.0.1.10
floating-ip 10.0.2.10
blade-parameters
priority 220
exit
!
vrrp-a interface ethernet 2
!
vrrp-a peer-group
peer 10.0.2.12
exit
!
vrrp-a session-sync enable
exit
wr mem
```
**vThunderADC-2 VRRP-A CONFIGURATION EXAMPLE**
```
vrrp-a common
device-id 2
set-id 1
enable
exit
!
vrrp-a vrid 0
floating-ip 10.0.1.10
floating-ip 10.0.2.10
blade-parameters
priority 220
exit
!
vrrp-a interface ethernet 2
!
vrrp-a peer-group
peer 10.0.2.11
exit
!
vrrp-a session-sync enable
exit
wr mem
```
## Assign floating-ip addresses and Virtual IP's in OCI
Once the failover configuration is completed The next step is to add the floating-ip addresses to the vnics in OCI.  In addition the Virtual IP address will also be configured on the Public VNIC.  This example will only reflect a single vip, but multiple can ben configured depending on the requirements.

1. Login to OCI and go to the instances screen.
1. Select the `vThunderADC-1` instances
    > NOTE:  This portion of the configuration is only completed on the primary (`vThunderADC-1`) device
1. Scroll down to the `Resources` section and select `Attached VNICS`
1. Select `Public_VNIC` and under `Resources` select IP Addresses.
1. First the floating IP address is configured by selecting `Assign Private IP address`
1. On the `Private IP Address` screen fill out the following information:
   * Private IP address:  10.0.1.10
   * Public IP address:  No Public IP
1. Finish the configuration by selecting `Assign`

### Create Virtual IP
Next step is to assign a virtual IP address.  For this the virtual IP address is 10.0.1.20.
1. Login to OCI and go to the instances screen.
1. Select the `vThunderADC-1` instances
   > NOTE:  This portion of the configuration is only completed on the primary (`vThunderADC-1`) device
1. Scroll down to the `Resources` section and select `Attached VNICS`
1. Select `Public_VNIC` and under `Resources` select IP Addresses.
1. First the floating IP address is configured by selecting `Assign Private IP address`
1. On the `Private IP Address` screen fill out the following information:
   * Private IP address:  10.0.1.20
   * Public IP address:  Reserved Public IP
   * Reserved Public IP:  Create a New Reserved Public IP
   * Reserved Public IP Name: web

     ![VIP IP](./images/vip-ip.png)
1. Finish the configuration by selecting `Assign`
1. Note the public IP address, this will be the external web site address.
      ![VIP public IP](./images/vip-public-ip.png)

### Configure Server Floating IP
1. Select the `vThunderADC-1` instances
    > NOTE:  This portion of the configuration is only completed on the primary (`vThunderADC-1`) device
1. Scroll down to the `Resources` section and select `Attached VNICS`
1. Select `Server_VNIC` and under `Resources` select IP Addresses.
1. First the floating IP address is configured by selecting `Assign Private IP address`
1. On the `Private IP Address` screen fill out the following information:
   * Private IP address:  10.0.2.10
   * Public IP address:  No Public IP
     > NOTE:  For any of the back-end servers, this will be their default gateway.
1. Finish the configuration by selecting `Assign`

# CONFIGURE THE VIRTUAL SERVER (VIP) ON VTHUNDER ADC-1

The user can configure virtual services, or VIP, using the CLI, GUI, AppCentric Templates (ACT) or Harmony Controller.
In this document, for ease of configuration and operation, ACT is used, the A10 ACOS GUI plug-in module that enhances the user experience to deploy, monitor and troubleshoot applications in a frictionless manner. ACT contains wizard-based configuration tools  for many different applications and use-case configurations, including Basic LB, HTTPS/SSL Offload, MS Exchange, GSLB and more.

## L4 VIP CREATION USING ACT
This section explains how to configure a basic VIP (virtual server) for a port 80 web service using the ACT.
>NOTE: ACT version used in the example is act-v2-1214-17-a10-0.tar.gz
