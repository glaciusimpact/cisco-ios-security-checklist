# Cisco IOS security checklist

Below is a checklist of measures to strengthen the security of Cisco switches and routers (IOS and IOS XE).

## Security Checklist

- [ ] [1. Deferrence](#1-deferrence)
	- [ ] [1.1 Add a warning banner](#11-add-a-warning-banner)
- [ ] [2. Attack Surface Reduction](#2-attack-surface-reduction)
	- [ ] [2.1 Disable Telnet](#21-disable-telnet)
	- [ ] [2.2 Disable Web interfaces](#22-disable-web-interfaces)
	- [ ] [2.3 Disable FTP](#23-disable-ftp)
- [ ] [3. Secure local management access](#3-secure-local-management-access)
	- [ ] [3.1 Use enable secret](#31-use-enable-secret)
	- [ ] [3.2 Disable enable password](#32-disable-enable-password)
	- [ ] [3.3 Secure Console access](#33-secure-console-access)
- [ ] [4. Secure remote management access](#4-secure-remote-management-access)
	- [ ] [4.1 Enable and setup SSH](#41-enable-and-setup-ssh)
	- [ ] [4.2 Radius access](#42-radius-access)
- [ ] [5. Preventing loops](#5-preventing-loops)
	- [ ] [5.1 Spanning-tree](#51-spanning-tree)
	- [ ] [5.2 VLAN Root Election Protection](#52-vlan-root-election-protection)
	- [ ] [5.3 BPDU Guard](#53-bpdu-guard)
	- [ ] [5.4 Storm control](#54-storm-control)
- [ ] [6. Limit Reconnaissance techniques](#6-limit-reconnaissance-techniques)
	- [ ] [6.1 Disable CDP](#61-disable-cdp)
	- [ ] [6.2 Disable LLDP](#62-disable-lldp)
- [ ] [7. Denial of Service mitigation](#7-denial-of-service-mitigation)
	- [ ] [7.1 MAC Address Flooding Attack](#71-mac-address-flooding-attack)
	- [ ] [7.2 DHCP Snooping](#72-dhcp-snooping)
- [ ] [8. VLAN Attack Protection](#8-vlan-attack-protection)
	- [ ] [8.1 VTP mode transparent](#81-vtp-mode-transparent)
	- [ ] [8.2 VLAN hopping](#82-vlan-hopping)
- [ ] [9. Monitoring Protection]
	- [ ] 9.1 SNMPv3

Reference:
- [Cisco IOS Encryption/Hash](#cisco-ios-encryptionhash)

---

### 1. Deferrence

#### 1.1 Add a warning banner

The warning banner attempts to deter unauthorized individuals from logging into a device and informs them that their activity will be logged.

a) First banner

This banner appears before login for remote connections (SSH).

``` pascal
banner login $
###############################################################

WARNING: This is a private device. This service is restricted
to authorized users only. All connections are monitored and
recorded. Disconnect IMMEDIATELY if you are not an authorized
user! Unauthorized access will be fully investigated and
reported to the appropriate law enforcement agencies.

###############################################################$
```

b) Second banner

This banner appears before login for console and after login for remote connections (SSH).

``` pascal
banner motd $
###############################################################

WARNING: This is a private device. This service is restricted
to authorized users only. All connections are monitored and
recorded. Disconnect IMMEDIATELY if you are not an authorized
user! Unauthorized access will be fully investigated and
reported to the appropriate law enforcement agencies.

###############################################################$
```

---

### 2. Attack Surface Reduction

#### 2.1 Disable Telnet

Telnet is an insecure protocol and should no longer be used in production.

``` pascal
line vty 0 15
transport input ssh
```

#### 2.2 Disable Web interfaces

HTTP is an insecure protocol and should no longer be used in production.

``` pascal
no ip http server
no ip http secure-server
```

#### 2.3 Disable FTP

FTP is an insecure protocol and should no longer be used in production.

``` pascal
no ftp-server enable
```

---

### 3. Secure local management access

#### 3.1 Use enable secret

Enable secret command sets a encrypted/hashed password to access the enable mode when someone is in user mode.

(change "mysecret" with a more robust password)
```
enable secret mysecret
```

<details>
<summary>Output</summary>

You should get something like this in the running-configuration.

``` python
enable secret 5 $1$mERr$/x9VUDEedbClBAt8DhbGj0
```
</details>

#### 3.2 Disable enable password

In case a clear password was set for reaching enable mode this command will remove it.

```
no enable password
```

#### 3.3 Secure Console access

There are 2 configurations possible with Console access. This one is the prefered configuration using an account with a more secure option (Type 5) than using only a password crypted using service password-encryption (Type 7 encryption). A 30 seconds timeout is set for security reason.

a) Secure option

Replace "myconsolepassword" with a more secure password. If scrypt (Type 9) is not supported then remove "algorithm-type scrypt" for Type 5 encryption/hash.

``` pascal
username myconsoleaccount algorithm-type scrypt secret myconsolepassword
line console 0
no password
login local
exec-timeout 30
```

<details>
<summary>Output</summary>

``` python
Switch>en
Switch#configure terminal 
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#username myconsoleaccount algorithm-type scrypt secret myconsolepassword
Switch(config)#line console 0
Switch(config-line)#no password
Switch(config-line)#login local
Switch(config-line)#exec-timeout 30
Switch(config-line)#^Z
Switch#
%SYS-5-CONFIG_I: Configured from console by console

Switch#show running-config 

[...]

!
username myconsoleaccount secret 9 $9$J19FIAftPZf7c3$kkL9xjX51NpUX13uMcG3XdQE4z2Q.G3NKlAiYtBov9c
!

[...]

!
line con 0
 exec-timeout 30 0
 login local
!

[...]
```
</details>

b) Less secure option (Refrain from using)

Just for information here is the other Console access configuration but less secure. Prefer the more secure option.

```
line console 0
password 123456
login
exec-timeout 30
exit
service password-encryption
```

<details>
<summary>Output</summary>

``` python
Switch#configure terminal 
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#line console 0
Switch(config-line)#password 123456
Switch(config-line)#login
Switch(config-line)#exec-timeout 30
Switch(config-line)#do sh run | i password
no service password-encryption
 password 123456
Switch(config-line)#exit
Switch(config)#service password-encryption
Switch(config)#do sh run | i password
service password-encryption
 password 7 08701E1D5D4C53
Switch(config)#
```
</details>

---

### 4. Secure remote management access

#### 4.1 Enable and setup SSH

SSH allows a remote secure access to the device. A hostname of the device has to be set (here SW1 but you can set the one you want) a key must be generated. A domain name, an account and its password must be set also (they can be modified). A timeout of 30 seconds is set but can be modified if needed. Add of course an IP address to a SVI to reach the device if not already set.

``` pascal
hostname SW1
ip domain name mydomain.com
crypto key generate rsa

username myaccount secret mypassword
ip ssh version 2

line vty 0 15
login local
transport input ssh
exec-timeout 30
```

<details>
<summary>Output</summary>

``` python
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#hostname SW1
SW1(config)#ip domain name mydomain.com
SW1(config)#crypto key generate rsa
The name for the keys will be: SW1.mydomain.com
Choose the size of the key modulus in the range of 360 to 4096 for your
  General Purpose Keys. Choosing a key modulus greater than 512 may take
  a few minutes.

How many bits in the modulus [512]: 4096
% Generating 4096 bit RSA keys, keys will be non-exportable...[OK]

SW1(config)#
SW1(config)#username myaccount secret mypassword
*Mar 1 6:28:51.334: %SSH-5-ENABLED: SSH 1.99 has been enabled
SW1(config)#ip ssh version 2
SW1(config)#line vty 0 15
SW1(config-line)#login local
SW1(config-line)#transport input ssh
SW1(config-line)#^Z
SW1#
%SYS-5-CONFIG_I: Configured from console by console

SW1#show running-config 
Building configuration...

Current configuration : 1555 bytes
!
version 16.3.2
no service timestamps log datetime msec
no service timestamps debug datetime msec
no service password-encryption
!
hostname SW1
!
!
no profinet
!
!
!
!
!
!
no ip cef
no ipv6 cef
!
!
!
username myaccount secret 5 $1$mERr$7sOd0mgRuXYhHwfWsV4QZ/
!
!
!
!
!
!
!
!
!
!
ip ssh version 2
no ip domain-lookup
ip domain-name mydomain.com
!
!

[...]

!
line vty 0 4
 login local
 transport input ssh
line vty 5 15
 login local
 transport input ssh
!
!
!
!
end


SW1#
```
</details>

---

#### 4.2 Radius access

Instead of have several account on every devices an access with a Radius account simplify the management of the accounts. Below is a basic configuration. TACACS+ configuration is another option similar but proprietary (Cisco).

Note that the SSH account created previously can be removed or kept as a backup account in case the connection with the Radius server is no reacheable (this is why there is "aaa authentication login default group radius local").

Warning: as long as the device can make requests to the Radius server local accounts will not be usable for Console and SSH. Only Radius accounts will allow to access the device.

``` pascal
aaa new-model
radius server rad_config
address ipv4 10.0.1.254 auth-port 1812
key mys3cret
exit
aaa authentication login default group radius local
aaa authorization exec default group radius
line vty 0 15
login authentication default
```

<details>
<summary>Output</summary>

``` python
SW1(config)#aaa new-model
SW1(config)#radius server rad_config
SW1(config-radius-server)#address ipv4 10.0.1.254 auth-port 1812
SW1(config-radius-server)#key mys3cret
 WARNING: Command has been added to the configuration using a type 0 password. However, type 0 passwords will soon be deprecated. Migrate to a supported password type
*Feb 15 18:39:23.109: %AAAA-4-CLI_DEPRECATED: WARNING: Command has been added to the configuration using a type 0 password. However, type 0 passwords will soon be deprecated. Migrate to a supported password type
SW1(config-radius-server)#exit
SW1(config)#aaa authentication login default group radius local
SW1(config)#aaa authorization exec default group radius
SW1(config)#line vty 0 15
SW1(config-line)#login authentication default
SW1(config-line)#^Z
SW1#
%SYS-5-CONFIG_I: Configured from console by console

SW1#show running-config
Building configuration...

Current configuration : 2925 bytes
!
version 16.3.2
no service timestamps log datetime msec
no service timestamps debug datetime msec
no service password-encryption
!

[...]

!
aaa new-model
!
aaa authentication login default group radius local 
!
!
aaa authorization exec default group radius
!

[...]

!
radius server rad_config
 address ipv4 10.0.1.254 auth-port 1812
 key mys3cret
radius server 10.0.1.254
 address ipv4 10.0.1.254 auth-port 1812
 key mys3cret
!
!
!
line con 0
 exec-timeout 30 0
!
line aux 0
!
line vty 0 4
 login authentication default
 transport input ssh
line vty 5 15
 login authentication default
 transport input ssh
!
!
!
!
end


SW1#
```
</details>


----

### 5. Preventing loops

#### 5.1 Spanning-tree

MSTP and Rapid PVST+ are the best options for spanning-tree. They both have a good performance to prevent network loops.

Here is the configuration to enable spanning-tree is Rapid PVST+ in a switch.

``` pascal
spanning-tree mode rapid-pvst 
```

<details>
<summary>Output</summary>

``` python
SW1(config)#spanning-tree mode rapid-pvst 
SW1(config)#do show spanning-tree summary
Switch is in rapid-pvst mode
Root bridge for: default
Extended system ID           is enabled
Portfast Default             is disabled
PortFast BPDU Guard Default  is disabled
Portfast BPDU Filter Default is disabled
Loopguard Default            is disabled
EtherChannel misconfig guard is disabled
UplinkFast                   is disabled
BackboneFast                 is disabled
Configured Pathcost method used is short

Name                   Blocking Listening Learning Forwarding STP Active
---------------------- -------- --------- -------- ---------- ----------
VLAN0001                     0         0        0          3          3

---------------------- -------- --------- -------- ---------- ----------
2 vlans                      0         0        0          3          3

SW1(config)#
```
</details>

#### 5.2 VLAN Root Election Protection

In order to prevent another switch connected to the network to become the spanning-tree root it is mendatory to get the low possible value (0) of the Priority of the Bridge ID inside the BPDU.

Bridge ID (BID)
```
   4 bits           12 bits           48 bits
+-----------+---------------------+-------------+
| Priority  | System ID Extension |             |
| (Multiple | (Typically holds    | MAC Address |
| of 4096)  |  VLAN ID)           |             |
+-----------+---------------------+-------------+
```

Note: Root Guard could also be used to protect unexpected VLAN root election but it shutdowns a switch port which can be a problem.

So on every VLAN of the Root switch (core switch?) the priority should be set to 0:

``` pascal
spanning-tree vlan 1 priority 0
spanning-tree vlan 2 priority 0
spanning-tree vlan 3 priority 0
...
etc.
```

#### 5.3 BPDU Guard

BPDU guard prevents a rogue switch or a cabling mistake to create a loop. It can be configured globaly (i.e on every ports but without a configuration on each port) or on all ports individually. The following configuration shows how to do it individually:

``` pascal
interface range gigabitEthernet 1/0/5-24
spanning-tree bpduguard enable
```

<details>
<summary>Output</summary>

``` python
SW1(config-if)#interface range gigabitEthernet 1/0/5-24
SW1(config-if-range)#spanning-tree bpduguard enable
SW1(config-if-range)#
```
</details>

---

#### 5.4 Storm control

Broadcast, Multicast or Unknow Unicast storm may occur when the network has a loop. This happens if spanning tree is not correcty configured or if a network attack is running. By default the port reach the state of ERR-DISABLE (a shut/no shut is needed to re-enable the port).

``` pascal
interface gigabitEthernet 1/0/1
storm-control broadcast level 10
```

<details>
<summary>Output</summary>

``` python
Switch(config)#interface gigabitEthernet 1/0/1
Switch(config-if)#storm-control broadcast level 10
Switch(config-if)#^Z
Switch#
%SYS-5-CONFIG_I: Configured from console by console

Switch#
Switch#show storm-control broadcast 
Interface  Filter State   Upper        Lower        Current
---------  -------------  -----------  -----------  ----------
Gig1/0/1   Link Up         10.00%       10.00%        0.00%

Switch#
```
</details>

---

### 6. Limit Reconnaissance techniques

#### 6.1 Disable CDP

CDP can be remove globally (on all the device) or just on some interfaces (usually endpoint to endpoints or other company switches). By default CDP is enabled.

##### a) CDP removed globally

``` pascal
no cdp run
```

<details>
<summary>Output</summary>

``` python
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#no cdp run 
Switch(config)#^Z
Switch#
%SYS-5-CONFIG_I: Configured from console by console

Switch#show running-config 
Building configuration...

[...]

!
no cdp run
!
!

[...]
```
</details>

##### b) CDP removed partially

``` pascal
interface range gigabitEthernet 1/0/2 -7
no cdp enable
```

<details>
<summary>Output</summary>

``` python
Switch(config)#interface range gigabitEthernet 1/0/2 -7
Switch(config-if-range)#no cdp enable
Switch(config-if-range)#^Z
Switch#
%SYS-5-CONFIG_I: Configured from console by console

Switch#show running-config 
Building configuration...

[...]

interface GigabitEthernet1/0/2
 no cdp enable
!
interface GigabitEthernet1/0/3
 no cdp enable
!
interface GigabitEthernet1/0/4
 no cdp enable
!
interface GigabitEthernet1/0/5
 no cdp enable
!
interface GigabitEthernet1/0/6
 no cdp enable
!
interface GigabitEthernet1/0/7
 no cdp enable
!
interface GigabitEthernet1/0/8
!
interface GigabitEthernet1/0/9

[...]
```
</details>

---

#### 6.2 Disable LLDP

LLDP can be remove globally (on all the device) or just on some interfaces (usually endpoint to endpoints or other company switches). By default LLDP is not enabled (so disabling it globally will remove it from the configuration).

##### a) LLDP removed globally

``` pascal
no lldp run
```

<details>
<summary>Output</summary>

``` python
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#no lldp run
Switch(config)#^Z
Switch#
```
</details>

##### b) CDP removed partially

``` pascal
interface range gigabitEthernet 1/0/2 -7
no lldp transmit
no lldp receive
```

---

### 7. Denial of Service mitigation

#### 7.1 MAC Address Flooding Attack

MAC Address flooding attack is countered on port receiving more MAC addresses then expected (like endpoint interfaces) thanks to Port Security. The port is then shutdown if 2 MAC addresses or if a specified number of MAC addresses is reached on an interface.

Here are 2 port configurations to mitigate MAC address flooding attacks:
- FastEthernet0/1 has a limitation of 1 MAC address (that is a second MAC address on this port will shutdown the port)
- FastEthernet0/2 hasa limitation of 8 MAC addresses (so a 9th MAC address seen on the port will shutdown the port) 

``` pascal
interface FastEthernet0/1
 switchport mode access
 switchport port-security
!
interface FastEthernet0/2
 switchport mode trunk
 switchport port-security
 switchport port-security maximum 8
```

#### 7.2 DHCP Snooping

DHCP snooping is a protection mechanism on switches used to filter DHCP messages received on "untrusted" ports. This protects against rogue DHCP servers.

- If a DHCP message is received on a "trusted" port, the packet is forwarded without inspection.
- If a message is received on an "untrusted" port, the message is inspected and may be discarded.

``` pascal
ip dhcp snooping
ip dhcp snooping vlan 1
no ip dhcp snooping information option 

interface Fa0/1
ip dhcp snooping trust
```

---

### 8. VLAN Attack Protection


#### 8.1 VTP mode transparent

VTP is not recommended any longer by Cisco. It can potentially destroy the VLANs of a switch. If the VTP of a switch is configured in transparent mode it is not vulnerable to a VTP attack or misconfiguration and its VLAN configuration is stored in the running config and not any longer in vlan.dat.

``` pascal
vtp mode transparent
```

<details>
<summary>Output</summary>

``` python
SW1(config)#vtp mode transparent
Setting device to VTP TRANSPARENT mode.
SW1(config)#
```
</details>

#### 8.2 VLAN hopping

VLAN hopping is the ability for an attacker to access a network in a VLAN that he shouldn't be able to reach.

To perform a VLAN hopping there are 2 techniques:
1. by using the Dynamic Trunking Protocol (DTP) which allows a port to be in trunk mode, or by leaving a port configured in trunk mode accessible without filtering VLANs. This is also called switch spoofing.
2. by using double VLAN tagging.

Below, we will see the mitigations for both attacks.

##### a) DTP and trunk mitigation

DTP is a Cisco proprietary protocol used to set a switch port automatically in trunk or access mode. Unfortunately this feature allows an attacker to get access to all VLANs of a switch if he can send a DTP message to have his port configure in trunk with all the VLANs allowed by default.

Note: Disabling the DTP protocol also reduces the ability for an attacker to identify a switch. This is because DTP messages contain the OUI of a MAC address, which allows the manufacturer's name to be identified. What is more DTP being a proprietary protocol then the device sending those messages is mostly a Cisco switch.

To mitigate this we can either:

###### a.1) Configure ports in access mode (that disables DTP)

Enabling access mode of switch ports disables DTP so no DTP specific command is needed. The port configured in access mode cannot become a trunk so VLAN hopping with DTP is not possible.

Remeber: do not leave a port unconfigured since it will be in auto-mode (and can become a trunk). Either shutdown switch ports or configure them in access mode disabling DTP.

``` pascal
switchport mode access
```

<details>
<summary>Output</summary>

``` python
Switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
Switch(config)#int gi1/0/2
Switch(config-if)#switchport mode access 
Switch(config-if)#do show interface gi1/0/2 switchport
Name: Gig1/0/2
Switchport: Enabled
Administrative Mode: static access
Operational Mode: static access
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: native
Negotiation of Trunking: Off
Access Mode VLAN: 1 (default)
Trunking Native Mode VLAN: 1 (default)
Voice VLAN: none
Administrative private-vlan host-association: none
Administrative private-vlan mapping: none
Administrative private-vlan trunk native VLAN: none
Administrative private-vlan trunk encapsulation: dot1q
Administrative private-vlan trunk normal VLANs: none
Administrative private-vlan trunk private VLANs: none
Operational private-vlan: none
Trunking VLANs Enabled: All
Pruning VLANs Enabled: 2-1001
Capture Mode Disabled
Capture VLANs Allowed: ALL
Protected: false
Appliance trust: none


Switch(config-if)#
```

-> We get:

```
Administrative Mode: static access
Negotiation of Trunking: Off
```

instead of:

```
Administrative Mode: dynamic auto
Negotiation of Trunking: On
```

which is good!
</details>

###### a.2) Disable DTP protocol on trunk ports

Sometimes we cannot use access ports but trunk ports to connect devices to the switch. If a port is configured in trunk mode then DTP is enable by default. We can disable DTP on a trunk port which removes those messages to an attacker. But the VLAN hopping problem in trunk mode does not come from the DTP. Actually an attacker can send packets with all VLANS he wants if no VLAN filtering is set on a trunk port so VLAN hopping is still possible. So here we are going to configure a trunk, disable DTP messages and filter VLANs (VLAN 1, 2, 3, 4, 5 and 7 other VLANs will be blocked).

Note: it is not possible to disable DTP globally or on a port without specifying a mode. 

``` pascal
switchport mode trunk
switchport nonegotiate
switchport trunk allowed vlan 1-5,7
```

<details>
<summary>Output</summary>

``` python
Switch(config)#int gigabitEthernet 1/0/5
Switch(config-if)#switchport mode trunk

Switch(config-if)#
%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1/0/5, changed state to down

%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet1/0/5, changed state to up

Switch(config-if)#switchport nonegotiate
Switch(config-if)#switchport trunk allowed vlan 1-5,7
Switch(config-if)#do show int gi1/0/5 switchport
Name: Gig1/0/5
Switchport: Enabled
Administrative Mode: trunk
Operational Mode: trunk
Administrative Trunking Encapsulation: dot1q
Operational Trunking Encapsulation: dot1q
Negotiation of Trunking: Off
Access Mode VLAN: 1 (default)
Trunking Native Mode VLAN: 1 (default)
Voice VLAN: none
Administrative private-vlan host-association: none
Administrative private-vlan mapping: none
Administrative private-vlan trunk native VLAN: none
Administrative private-vlan trunk encapsulation: dot1q
Administrative private-vlan trunk normal VLANs: none
Administrative private-vlan trunk private VLANs: none
Operational private-vlan: none
Trunking VLANs Enabled: 1-5,7
Pruning VLANs Enabled: 2-1001
Capture Mode Disabled
Capture VLANs Allowed: ALL
Protected: false
Appliance trust: none


Switch(config-if)#
```
->

We get:

```
Negotiation of Trunking: Off
Trunking VLANs Enabled: 1-5,7
```

instead of:

```
Negotiation of Trunking: On
Trunking VLANs Enabled: All
```
</details>

##### b) Double tagging mitigation


### 9. Monitoring Protection

---

### Cisco IOS Encryption/Hash

| Type | Cisco Encryption/Hash                       | Ability to crack | NSA recommendations (2022)                                                     |
| :--: | ------------------------------------------- | :--------------: | :----------------------------------------------------------------------------- |
|  0   | Clear                                       |    Immediate     | $${\color{red}Do \space not \space use}$$                                      |
|  4   | SHA-256 (single iteration, no salt)         |       Easy       | $${\color{red}Do \space not \space use}$$                                      |
|  5   | MD5                                         |      Medium      | Not NIST approved, use only when Types 6, 8, and 9 are not available           |
|  6   | AES-128 Reversible Encryption               |    Difficult     | Use only when reversible encryption is needed, or when Type 8 is not available |
|  7   | Encrypted (Reversible Vigenere obfuscation) |    Immediate     | $${\color{red}Do \space not \space use}$$                                      |
|  8   | SHA-256                                     |    Difficult     | $${\color{green}Recommended}$$                                                 |
|  9   | Scrypt                                      |    Difficult     | Not NIST approved                                                              |

- Links:
	- NSA Best practice
https://media.defense.gov/2022/Feb/17/2002940795/-1/-1/1/CSI_CISCO_PASSWORD_TYPES_BEST_PRACTICES_20220217.PDF
	- Cisco Type 7 Password Decrypt / Decoder / Crack online Tool
https://www.firewall.cx/cisco/cisco-routers/cisco-type7-password-crack.html

