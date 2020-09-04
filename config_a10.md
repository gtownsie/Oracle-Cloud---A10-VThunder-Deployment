- [Main Menu](./README.md)
- [Previous - Deploy A10 Instances](./deploy_a10.md)
- [Next - Create Virtual Server](./virtual_a10.md)
---
- [A10 vThunder Configuration](#configvthunder)
   - [Primary vThunder - Configuration](#configpri)
      - [DNS and Hostname Configuration](#configpridnshost)
      - [Network Configuration](#configprinetwork)
   - [Secondary vThunder - Configuration](#configsec)
      - [Configure DNS and Hostname](#configsecdnshost)
      - [Network Configuration](#configsecnetwork)
   - [Redundancy Configuration](#redundancy)
      - [Import API Private Key and Cloud Config File](#redundancyconfig)]
      - [Copy SSH Key to Primary vThunderADC](#redundancykey)
      - [High Availability (VRRP-A) Configuration (HA)](#configha)
      - [Add Floating IP to Oracle Cloud](#configocifloat)
---
# [A10 vThunder Configuration](#configvthunder)
## [Primary vThunder - Configuration](#configpri)
The next step is the configuration of the data plane network interfaces, default gateway, DNS, and Hostname.

Name|IP Address|Floating IP
---------|---------|---------
Hostname Name|vThunderADC-1|
Management Network|DHCP|
Public Network|10.0.1.11|
Server Network|10.0.2.11|10.0.2.10

### [DNS and Hostname Configuration](#configpridnshost)
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

### [Network Configuration](#configprinetwork)
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

## [Secondary vThunder - Configuration](#configsec)
The next step is the configuration of the data plane network interfaces, default gateway, DNS, and Hostname.

>***NOTE:  The Hostname MUST match the Instance name***

Name|IP Address|Floating IP
---------|---------|---------
Hostname Name|vThunderADC-2|
Management Network|DHCP|
Public Network|10.0.1.12|10.0.1.10
Server Network|10.0.2.12|10.0.2.10

### [Configure DNS and Hostname](#configsecdnshost)
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
### [Network Configuration](#configsecnetwork)
1. Validate that the vThunder recognizes the network interfaces by running the `sh interfaces brief`, below is a sample of the output, if only the management interface is shown issue a reboot command, the interfaces are recognized after the vnics are created and the instance is rebooted:
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

## [Redundancy Configuration](#redundancy)
The next step in the process is to configure redundancy.  It is A10 recommended practice to have a dedicated interface and subnet for an `HA` network.  In some instances, when the vThunder is deployed with 2 interfaces, the first interface is used for management and the second is used for the server and public network in a 1-arm configuration.  Also, if a customer is running a 4 vnic instance and requires the 3 data plane interfaces for production traffic, then utilizing a data plane interface may be needed.  To demonstrate the A10 deployment flexibility, this example will not use an HA vlan/network the Server_Network is used.
> **If a vnic is available follow the steps above and add an additional network  and interface for `HA`**

### [Import API Private Key and Cloud Config File](#redundancyconfig)
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

### [Copy SSH Key to Primary vThunderADC](#redundancykey)
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

### [High Availability (VRRP-A) Configuration (HA)](#configha)
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

### Add Floating IP to Oracle Cloud
1. Login to OCI and go the instances screen
1. Select the `vThunderADC-1` instances
    > NOTE:  This portion of the configuration is only completed on the primary (`vThunderADC-1`) instance

1. Scroll down to the `Resources` section and select `Attached VNICS`
1. Select `Server_VNIC` and under `Resources` select IP Addresses.
1. First the floating IP address is configured by selecting `Assign Private IP address`
1. On the `Private IP Address` screen fill out the following information:
   * Private IP address:  10.0.2.10
   * Public IP address:  No Public IP
     > NOTE:  For any of the back-end servers, this will be their default gateway.
1. Finish the configuration by selecting `Assign`

---
- [Main Menu](./README.md)
- [Previous - Deploy A10 Instances](./deploy_a10.md)
- [Next - Create Virtual Server](./virtual_a10.md)
