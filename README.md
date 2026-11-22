flowchart TD
    subgraph WAN["External Networks & Upstream Uplink"]
        ISP["Upstream Lab / WAN Uplink<br/>198.x.x.x/30 or 10.x.x.x/24"]
    end

    subgraph Hypervisor["Type-1 Hypervisor Layer (Proxmox VE)"]
        PVE["Proxmox VE Host (pve-node01)<br/>• Bare-Metal Type-1 Hypervisor<br/>• LVM-Thin / ZFS Storage Provisioning<br/>• Cloud-Init Golden Image Templates<br/>• QEMU Guest Agent Integration"]
    end

    subgraph VirtualNet["Virtual Switching & Perimeter Routing Layer"]
        VMBR0["vmbr0: WAN Linux Bridge<br/>• Bridged to Physical Uplink (eno1)"]
        FW["Virtual Router & Firewall (fw-router01)<br/>• Inter-VLAN Routing & Outbound NAT<br/>• Stateful Firewall & Zone Isolation<br/>• DHCP & Split-Horizon DNS Forwarding"]
        VMBR1["vmbr1: VLAN-Aware Linux Bridge<br/>• 802.1Q VLAN Trunking<br/>• Segmented Virtual Switching"]
    end

    subgraph Internal["Virtualized Internal Segments (Multi-OS VMs)"]
        subgraph V10["VLAN 10: Infrastructure Management"]
            Mgmt["Proxmox Web UI & Host Mgmt<br/>• Isolated from Guest VM Subnets<br/>• Admin-Only Access (:8006)<br/>(10.x.x.x/24)"]
        end

        subgraph V20["VLAN 20: Windows Core Infrastructure"]
            WinVM["Windows Server VM (win-dc01)<br/>• Active Directory (AD DS) & DNS<br/>• OVMF (UEFI) & TPM 2.0 Enabled<br/>• Paravirtualized VirtIO Storage/NIC<br/>(10.x.x.x/24)"]
        end

        subgraph V30["VLAN 30: Linux Application Tier"]
            LnxApp["Ubuntu Server VM (lnx-app01)<br/>• Automated Cloud-Init Provisioning<br/>• Docker & Internal App Services<br/>• UFW Host Firewall Hardening<br/>(10.x.x.x/24)"]
        end

        subgraph V99["VLAN 99: Admin & Monitoring"]
            LnxMon["Linux Bastion & Monitor (lnx-mon01)<br/>• SSH Jumpbox & Centralized Admin<br/>• Cross-VLAN Visibility & Metrics<br/>(10.x.x.x/24)"]
        end
    end

    ISP --> VMBR0
    VMBR0 === FW
    FW === VMBR1
    PVE -.- VMBR0
    PVE -.- VMBR1
    VMBR1 --- V10
    VMBR1 --- V20
    VMBR1 --- V30
    VMBR1 --- V99
