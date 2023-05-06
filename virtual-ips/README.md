# Virtual IPs

Virtual IPs, or VIPs, are used to decouple the service from the server.

A VIP can move from one server to another, allowing you to keep a service online while taking a specific server offline for maintenance. Or in an automated fashion, providing redundancy if a server goes offline unexpectedly.

## What can be a Virtual IP?

Almost anything that you reach over the network can be a Virtual IP.

* DNS Server
* Web Server
* Network Gateway
* VPN Endpoint
* Database
* API Endpoints/Microservices

## What's the catch?

Setting up a Virtual IP is easy. Setting up the services behind the virtual IP to function together or replicate in a bidirectional fashion is the real challenge.

The easiest things to make redundant are services which do not carry any state to be transferred between instances of the service. For example, a recursive DNS resolver can be instantiated without the two instances needing to share any information. At your worst case, moving the VIP points to a resolver with an empty cache, potentially providing an increase in response times as it warms up.

This article will start with easier to implement services and later iterations will contain more complex setups.

## Software

### Linux

On Linux keepalived is a commonly used tool to manage VIPs. It supports multicast and unicast VRRP as well as load balancing.

On ubuntu it can be installed with

    sudo apt install keepalived

The simplest possible keepalived configuration would consist of a single VIP with a low security password.

**Server 1**

    vrrp_instance vrrp_1 {
      state MASTER
      interface eth0
      virtual_router_id 1
      advert_int 1
      priority 200
      authentication {
        auth_type PASS
        auth_pass SECUREUNIQUEPASSWORD
      }
      virtual_ipaddress {
        192.0.2.1/24
      }
    }

**Server 2**

    vrrp_instance vrrp_1 {
      state BACKUP
      interface eth0
      virtual_router_id 1
      advert_int 1
      priority 100
      authentication {
        auth_type PASS
        auth_pass SECUREUNIQUEPASSWORD
      }
      virtual_ipaddress {
        192.0.2.1/24
      }
    }

You will also need to allow the vrrp service in firewalld if you are using it.

    sudo firewall-cmd --zone=public --add-service=vrrp --permanent && sudo firewall-cmd --reload

What we've established here is a VIP that will be assigned to eth0 on one but not both of the boxes at any given time, so long as the multicast VRRP traffic arrives at its destination.

Enable and start keepalived with the following on both servers.

    sudo systemctl enable --now keepalived

You can verify this by viewing the logs and the showing IP addresses on both servers. The logs should show communication between the two keepalived instances. Only one of the two servers should hold 192.0.2.1/24.

    sudo journalctl -u keepalived

    ip addr show eth0

### BSD based firewalls (opnsense et al)

On many BSD based firewalls with a webgui such as opnsense, CARP addresses are used for redundant addresses.

These can be used to provide redundant:

* Port forwards
* Gateways
* VPN servers
* whatever else your firewall provides

These firewalls can additionally be synced in an HA configuration so that rules flow from the primary to the secondary firewall and state tables are shared.

To set up a Virtual IP in opnsense 23.x:

**On the first firewall**

1. Nagivate to Interfaces -> Virtual IPs -> Settings.
2. Click the plus [+] to add a new virtual IP.
3. Select mode "CARP".
4. Select the interface that this VIP will be configured on.
5. Enter the Virtual IP and its subnet under Network / Address. eg 192.0.2.1/24  
Ensure that your firewall is not configured with address 192.0.2.1/24 -- consider addressing your first firewall as 192.0.2.2/24 and your second firewall as 192.0.2.3/24 leaving 192.0.2.1/24 for a highly available VIP as the gateway.
6. If you do not want the services presented by the firewall (eg openvpn, DNS forwarding, HTTP[S] server) select Deny service binding. This will still permit the IP to be used as a gateway or for port forwards if checked.
7. Enter a secure password for the Password.
8. Select a positive integer that you have not used for a virtual network ID elsewhere.
9. Enter a useful description about what the VIP is for.
10. Click save. Then click apply if prompted.


**Repeat these steps on the second firewall.**  Be sure to enter the exact same values used on the first firewall.

You will likely need to further configure the firewall with allow rules as required on the VIP.
