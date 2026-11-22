==============================================================================
​FILE: configs/router-and-vms.conf
​DESCRIPTION: Virtual Router (VyOS/OPNsense), Linux Guest (Netplan/UFW), and
​Windows Server (PowerShell AD DS/DNS) Configurations (10.x.x.x)
​==============================================================================
​==============================================================================
​PART 1: VIRTUAL ROUTER CONFIGURATION (fw-router01 - VyOS Declarative Syntax)
​==============================================================================
​1.1 Hostname & System DNS
​set system host-name 'fw-router01'
set system domain-name 'corp.local'
set system name-server '10.10.20.10'
set system name-server '10.255.0.1'
​1.2 Network Interfaces & 802.1Q VLAN Sub-Interfaces
​set interfaces ethernet eth0 address '10.255.0.2/24'
set interfaces ethernet eth0 description 'WAN-UPLINK-VMBR0'
​set interfaces ethernet eth1 description 'LAN-TRUNK-VMBR1'
set interfaces ethernet eth1 vif 10 address '10.10.10.1/24'
set interfaces ethernet eth1 vif 10 description 'VLAN10-INFRA-MGMT'
​set interfaces ethernet eth1 vif 20 address '10.10.20.1/24'
set interfaces ethernet eth1 vif 20 description 'VLAN20-WINDOWS-CORE'
​set interfaces ethernet eth1 vif 30 address '10.10.30.1/24'
set interfaces ethernet eth1 vif 30 description 'VLAN30-LINUX-APPS'
​set interfaces ethernet eth1 vif 99 address '10.10.99.1/24'
set interfaces ethernet eth1 vif 99 description 'VLAN99-ADMIN-MONITORING'
​1.3 Default Static Route & Outbound Source NAT (Masquerade)
​set protocols static route 0.0.0.0/0 next-hop 10.255.0.1
​set nat source rule 100 description 'NAT-INTERNAL-10.10-TO-WAN'
set nat source rule 100 outbound-interface name 'eth0'
set nat source rule 100 source address '10.10.0.0/16'
set nat source rule 100 translation address 'masquerade'
​1.4 DHCP Server for Dynamic Linux Clones (VLAN 30)
​set service dhcp-server shared-network-name VLAN30-POOL subnet 10.10.30.0/24 default-router '10.10.30.1'
set service dhcp-server shared-network-name VLAN30-POOL subnet 10.10.30.0/24 name-server '10.10.20.10'
set service dhcp-server shared-network-name VLAN30-POOL subnet 10.10.30.0/24 domain-name 'corp.local'
set service dhcp-server shared-network-name VLAN30-POOL subnet 10.10.30.0/24 range 0 start '10.10.30.100'
set service dhcp-server shared-network-name VLAN30-POOL subnet 10.10.30.0/24 range 0 stop '10.10.30.150'
​1.5 DNS Forwarding (Split-Horizon to Windows AD DC for corp.local)
​set service dns forwarding listen-address '10.10.10.1'
set service dns forwarding listen-address '10.10.20.1'
set service dns forwarding listen-address '10.10.30.1'
set service dns forwarding listen-address '10.10.99.1'
set service dns forwarding allow-from '10.10.0.0/16'
set service dns forwarding domain corp.local name-server '10.10.20.10'
set service dns forwarding name-server '10.255.0.1'
​1.6 Inter-VLAN Firewall Policies (Isolate VLAN 10 Mgmt from Guest VMs)
​set firewall ipv4 forward filter default-action 'drop'
set firewall ipv4 forward filter rule 10 action 'accept'
set firewall ipv4 forward filter rule 10 state 'established'
set firewall ipv4 forward filter rule 10 state 'related'
​Allow Admin VLAN 99 full access to all internal subnets
​set firewall ipv4 forward filter rule 20 action 'accept'
set firewall ipv4 forward filter rule 20 source address '10.10.99.0/24'
set firewall ipv4 forward filter rule 20 destination address '10.10.0.0/16'
​Allow Linux VLAN 30 to query Windows AD/DNS (10.10.20.10)
​set firewall ipv4 forward filter rule 30 action 'accept'
set firewall ipv4 forward filter rule 30 source address '10.10.30.0/24'
set firewall ipv4 forward filter rule 30 destination address '10.10.20.10'
set firewall ipv4 forward filter rule 30 destination port '53,88,389,445,636'
set firewall ipv4 forward filter rule 30 protocol 'tcp_udp'
​Allow all internal VLANs outbound internet access via WAN (eth0)
​set firewall ipv4 forward filter rule 40 action 'accept'
set firewall ipv4 forward filter rule 40 source address '10.10.0.0/16'
set firewall ipv4 forward filter rule 40 outbound-interface name 'eth0'
​==============================================================================
​PART 2: LINUX GUEST VMs CONFIGURATION (Ubuntu Server 24.04 LTS)
​==============================================================================
​2.1 Netplan Static Network Config: lnx-app01 (/etc/netplan/50-cloud-init.yaml)
​network:
version: 2
ethernets:
eth0:
dhcp4: false
addresses:
- 10.10.30.10/24
routes:
- to: default
via: 10.10.30.1
nameservers:
addresses:
- 10.10.20.10
- 10.10.30.1
search:
- corp.local
​2.2 Netplan Static Network Config: lnx-mon01 (/etc/netplan/50-cloud-init.yaml)
​network:
version: 2
ethernets:
eth0:
dhcp4: false
addresses:
- 10.10.99.10/24
routes:
- to: default
via: 10.10.99.1
nameservers:
addresses:
- 10.10.20.10
- 10.10.99.1
search:
- corp.local
​2.3 Linux Host Firewall (UFW) & SSH Hardening Commands
​ufw default deny incoming
​ufw default allow outgoing
​ufw allow from 10.10.99.0/24 to any port 22 proto tcp
​ufw allow from 10.10.0.0/16 to any port 80,443 proto tcp
​ufw --force enable
​==============================================================================
​PART 3: WINDOWS SERVER 2022/2025 CONFIGURATION (win-dc01 - PowerShell)
​==============================================================================
​3.1 Rename Host & Configure Static IP (10.10.20.10/24) on VirtIO Adapter
​Adapter = Get-NetAdapter \vert{} Where-Object {_.Status -eq "Up" }
New-NetIPAddress -InterfaceIndex $Adapter.ifIndex -IPAddress "10.10.20.10"
-PrefixLength 24 `
-DefaultGateway "10.10.20.1"
​Set-DnsClientServerAddress -InterfaceIndex $Adapter.ifIndex `
-ServerAddresses ("127.0.0.1", "10.10.20.1")
​Rename-Computer -NewName "win-dc01" -Force
​3.2 Install Active Directory Domain Services & DNS Server Roles
​Install-WindowsFeature -Name AD-Domain-Services, DNS -IncludeManagementTools
​3.3 Promote win-dc01 to New Forest Root Domain Controller (corp.local)
​$SafeModePassword = ConvertTo-SecureString "P@ssw0rdLab2026!" -AsPlainText -Force
Install-ADDSForest -DomainName "corp.local"
-DomainNetbiosName "CORP" -SafeModeAdministratorPassword $SafeModePassword
-InstallDns:$true `
-Force:$true
​3.4 Post-Promotion DNS Reverse Lookup Zone & A-Records (Run after reboot)
​Add-DnsServerPrimaryZone -NetworkID "10.10.0.0/16" -ReplicationScope "Forest"
Add-DnsServerResourceRecordA -Name "pve-node01" -ZoneName "corp.local" -IPv4Address "10.10.10.10" -CreatePtr
Add-DnsServerResourceRecordA -Name "lnx-app01"  -ZoneName "corp.local" -IPv4Address "10.10.30.10" -CreatePtr
Add-DnsServerResourceRecordA -Name "lnx-mon01"  -ZoneName "corp.local" -IPv4Address "10.10.99.10" -CreatePtr
​3.5 Enable ICMP Ping & RDP from Admin VLAN 99 (10.10.99.0/24)
​New-NetFirewallRule -DisplayName "Allow-ICMPv4-Internal-10.10" -Direction Inbound -Protocol ICMPv4 -IcmpType 8
-RemoteAddress "10.10.0.0/16" -Action Allow
​New-NetFirewallRule -DisplayName "Allow-RDP-Admin-VLAN99" -Direction Inbound -Protocol TCP -LocalPort 3389
-RemoteAddress "10.10.99.0/24" -Action Allow