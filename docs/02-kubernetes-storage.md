# Configure a Local DNS Server with BIND9 for Harbor on Kubernetes

## Overview

This guide explains how to install and configure BIND9 on an Ubuntu server to resolve the Harbor hostname `harbor.k8s.local` to the external IP address assigned to Traefik by MetalLB.

The objective is to allow clients on the local network to resolve the Harbor hostname through the local DNS server.

**Scope:** This guide covers BIND9 installation, DNS zone configuration, client DNS configuration, and DNS resolution tests.

## 1. Lab Environment

| Component | Example value |
|---|---|
| DNS server | `192.168.1.20` |
| Local network | `192.168.1.0/24` |
| Harbor hostname | `harbor.k8s.local` |
| Traefik external IP | `192.168.1.240` |
| DNS server interface | `ens33` |
| Example Windows client | `192.168.1.224` |

**Important:** Replace the example values with those used in your environment. In particular, verify the actual external IP address assigned to Traefik by MetalLB.

## 2. Identify the Traefik External IP

Run the following command on a machine with access to the Kubernetes cluster:

```bash
kubectl get svc -n traefik
```

Identify the Traefik Service and note its `EXTERNAL-IP`. This address will be used in the DNS zone file.

![Screenshot: Traefik External IP](../images/dns/01-traefik-external-ip.png)

If the external IP is shown as `<pending>`, check the MetalLB configuration and IP address pool before continuing.

## 3. Check the DNS Server Network Configuration

On the Ubuntu server that will host BIND9, run:

```bash
ip -br address
ip route
nmcli connection show
```

![Screenshot: DNS Server Network Configuration](../images/dns/02-dns-server-network.png)

Confirm the DNS server's IP address and identify the correct network interface. The examples in this guide use `192.168.1.20` as the DNS server address.

## 4. Install BIND9

Update the package index:

```bash
sudo apt update
```

![Screenshot: APT Package Update](../images/dns/03-apt-update.png)

Install BIND9 and the DNS diagnostic utilities:

```bash
sudo apt install -y bind9 bind9-utils dnsutils
```

![Screenshot: BIND9 Installation](../images/dns/04-bind9-installation.png)

Check the service status:

```bash
systemctl status named --no-pager
```

![Screenshot: BIND9 Service Status](../images/dns/05-bind9-service-status.png)

If the `named` service is not found, try:

```bash
systemctl status bind9 --no-pager
```

Use the service name that exists on your system in the following steps.

## 5. Configure BIND9 Options

Back up the original configuration:

```bash
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
```

![Screenshot: BIND9 Options Backup](../images/dns/06-bind9-options-backup.png)

Open the configuration file:

```bash
sudo nano /etc/bind/named.conf.options
```

![Screenshot: Editing BIND9 Options](../images/dns/07-bind9-options-editor.png)

Configure BIND9 with the following settings. Adjust the IP address and trusted subnet if your network is different.

```conf
acl "trusted" {
    127.0.0.1;
    192.168.1.0/24;
};

options {
    directory "/var/cache/bind";

    listen-on {
        127.0.0.1;
        192.168.1.20;
    };

    listen-on-v6 { none; };

    recursion yes;
    allow-query { trusted; };
    allow-recursion { trusted; };

    forwarders {
        1.1.1.1;
        8.8.8.8;
    };

    forward only;
    dnssec-validation auto;
};
```

![Screenshot: BIND9 Options Configuration](../images/dns/08-bind9-options.png)

This configuration allows DNS queries from localhost and the specified local network. The forwarders are used to resolve DNS queries that are not answered by the local zone.

**Security note:** Restrict access to your actual LAN. Do not expose a recursive DNS resolver to untrusted networks.

## 6. Declare the DNS Zone

Back up the local configuration file:

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
```

![Screenshot: BIND9 Local Configuration Backup](../images/dns/09-bind9-local-backup.png)

Edit the file:

```bash
sudo nano /etc/bind/named.conf.local
```

![Screenshot: Editing the Local DNS Configuration](../images/dns/10-bind9-local-editor.png)

Add the following zone declaration:

```conf
zone "k8s.local" {
    type primary;
    file "/etc/bind/db.k8s.local";
    allow-query { trusted; };
    allow-update { none; };
};
```

![Screenshot: DNS Zone Declaration](../images/dns/11-dns-zone-declaration.png)

This declares `k8s.local` as a primary DNS zone and specifies the file containing its DNS records.

## 7. Create the DNS Zone File

Create the zone file:

```bash
sudo nano /etc/bind/db.k8s.local
```

![Screenshot: Creating the DNS Zone File](../images/dns/12-dns-zone-file-editor.png)

Add the following records:

```dns
$TTL 86400
@   IN  SOA ns.k8s.local. admin.k8s.local. (
        2026102601 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400      ; Negative Cache TTL
)

@       IN  NS  ns.k8s.local.
ns      IN  A   192.168.1.20
harbor  IN  A   192.168.1.240
```

![Screenshot: DNS Zone File Records](../images/dns/13-dns-zone-file.png)

**Important:** Replace `192.168.1.240` with the actual Traefik external IP identified in Step 2. The `harbor` record must point to Traefik, not directly to the DNS server.

If you edit the zone later, increment the SOA serial number.

Set the file ownership and permissions:

```bash
sudo chown root:bind /etc/bind/db.k8s.local
sudo chmod 644 /etc/bind/db.k8s.local
```

![Screenshot: DNS Zone File Permissions](../images/dns/14-zone-file-permissions.png)

**Note about `.local`:** The `.local` suffix is reserved for Multicast DNS (mDNS). Some operating systems may handle `.local` names differently from ordinary DNS queries. This guide retains `harbor.k8s.local` for the example environment. For a new deployment, consider using a subdomain of a domain you own or `home.arpa` for a home network.

## 8. Validate the BIND9 Configuration

Check the main configuration:

```bash
sudo named-checkconf
```

![Screenshot: BIND9 Configuration Validation](../images/dns/15-named-checkconf.png)

No output normally indicates that the configuration syntax is valid.

Validate the DNS zone:

```bash
sudo named-checkzone k8s.local /etc/bind/db.k8s.local
```

![Screenshot: DNS Zone Validation](../images/dns/16-named-checkzone.png)

The output should confirm that the zone loaded successfully and report `OK`.

Correct any reported errors before restarting BIND9.

## 9. Restart BIND9 and Check Its Status

If the service is named `named`, run:

```bash
sudo systemctl restart named
sudo systemctl status named --no-pager
```

![Screenshot: BIND9 Restart and Service Status](../images/dns/17-bind9-restart-status.png)

If your system uses the `bind9` service unit instead, run:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9 --no-pager
```

If the service fails to start, inspect its logs:

```bash
sudo journalctl -u named -n 50 --no-pager
```

![Screenshot: BIND9 Service Logs](../images/dns/18-bind9-logs.png)

For a service named `bind9`, replace `named` with `bind9` in the log command.

## 10. Test DNS Resolution on the DNS Server

Test the DNS service through localhost:

```bash
dig @127.0.0.1 harbor.k8s.local
```

![Screenshot: Local DNS Resolution](../images/dns/19-dig-localhost.png)

Test through the DNS server's LAN address:

```bash
dig @192.168.1.20 harbor.k8s.local
```

![Screenshot: DNS Resolution Using the Server IP](../images/dns/20-dig-server-ip.png)

In the answer section, verify that `harbor.k8s.local` resolves to the external IP address assigned to Traefik.

You can also use `nslookup`:

```bash
nslookup harbor.k8s.local 192.168.1.20
```

![Screenshot: DNS Resolution with nslookup](../images/dns/21-nslookup-server.png)

If the returned IP address is incorrect, check the `harbor` A record in `/etc/bind/db.k8s.local`, increment the zone serial if you edited the file, and reload or restart BIND9.

## 11. Check Firewall Rules (If UFW Is Enabled)

Check the firewall status:

```bash
sudo ufw status verbose
```

![Screenshot: UFW Status](../images/dns/22-ufw-status.png)

If UFW is active, allow DNS queries from your local network over UDP and TCP port 53:

```bash
sudo ufw allow from 192.168.1.0/24 to 192.168.1.20 port 53 proto udp
sudo ufw allow from 192.168.1.0/24 to 192.168.1.20 port 53 proto tcp
sudo ufw status numbered
```

![Screenshot: DNS Firewall Rules](../images/dns/23-dns-firewall-rules.png)

Do not enable UFW without reviewing existing rules, especially SSH and any NFS or Kubernetes-related requirements. If you use a different firewall, configure equivalent rules there.

## 12. Configure DNS on a Linux Client

On a Linux client using `systemd-resolved`, inspect the current DNS configuration:

```bash
resolvectl status
ip -br address
```

![Screenshot: Linux DNS Client Status](../images/dns/24-linux-dns-status.png)

For a temporary configuration, replace `ens33` with the correct client interface:

```bash
sudo resolvectl dns ens33 192.168.1.20
sudo resolvectl domain ens33 '~k8s.local'
sudo resolvectl flush-caches
```

![Screenshot: Linux DNS Client Configuration](../images/dns/25-linux-dns-client-config.png)

Test DNS resolution:

```bash
resolvectl query harbor.k8s.local
```

![Screenshot: Linux DNS Query](../images/dns/26-linux-dns-query.png)

These settings may not persist after a reboot or network reconnection.

For persistent settings with NetworkManager, identify the connection name:

```bash
nmcli connection show
```

![Screenshot: NetworkManager Connection Name](../images/dns/27-networkmanager-connection.png)

Replace `netplan-ens33` with the actual connection name on your machine:

```bash
sudo nmcli connection modify "netplan-ens33" ipv4.dns "192.168.1.20"
sudo nmcli connection modify "netplan-ens33" ipv4.dns-search "~k8s.local"
sudo nmcli connection modify "netplan-ens33" ipv4.ignore-auto-dns yes
```

![Screenshot: Persistent Linux DNS Configuration](../images/dns/28-linux-persistent-dns-config.png)

Apply the changes, replacing `ens33` with the correct interface:

```bash
sudo nmcli device reapply ens33
```

![Screenshot: Linux DNS Settings Applied](../images/dns/29-linux-dns-applied.png)

Applying network changes may interrupt connectivity. Run these commands from a local console or ensure you have another way to recover access.

## 13. Configure DNS on Windows

Open PowerShell and list the available network adapters:

```powershell
Get-NetAdapter
```

![Screenshot: Windows Network Adapter](../images/dns/30-windows-network-adapter.png)

Run PowerShell as Administrator and set the DNS server for the correct adapter. Replace `Ethernet` if your adapter has a different name:

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "192.168.1.20"
```

![Screenshot: Windows DNS Configuration](../images/dns/31-windows-dns-config.png)

Flush the local DNS cache and query the Harbor hostname:

```powershell
ipconfig /flushdns
nslookup harbor.k8s.local
```

![Screenshot: Windows DNS Resolution](../images/dns/32-windows-dns-resolution.png)

The output should show `192.168.1.20` as the DNS server and the Traefik external IP as the resolved address.

If your computer normally obtains DNS settings through DHCP, record the existing settings before changing them so you can restore them if needed.

## 14. Troubleshooting

Check whether BIND9 is listening on port 53:

```bash
sudo ss -lntup | grep ':53'
```

![Screenshot: DNS Listening Ports](../images/dns/33-dns-listening-ports.png)

Check the Traefik Service and Kubernetes Ingress resources:

```bash
kubectl get svc -n traefik
kubectl get ingress -A
```

![Screenshot: Kubernetes Ingress and Traefik](../images/dns/34-kubernetes-ingress-traefik.png)

Common problems:

- **The hostname does not resolve:** Check BIND9 status, `named-checkconf`, `named-checkzone`, the zone records, and the DNS server configured on the client.
- **The hostname resolves to the wrong IP:** Update the Harbor A record to the current Traefik external IP, increment the zone serial, and reload or restart BIND9.
- **Queries time out from other machines:** Check network connectivity, BIND9 `listen-on`, the trusted ACL, and firewall rules for UDP/TCP port 53.
- **DNS works on the server but not on Windows:** Verify the Windows adapter's DNS server and run `ipconfig /flushdns`.
- **The hostname resolves but the application is unreachable:** Check the Traefik external IP, the Harbor Ingress host rule, and Kubernetes Service endpoints.



## References

- [BIND 9 Documentation](https://bind9.readthedocs.io/)
- [Ubuntu Server Documentation](https://documentation.ubuntu.com/server/)
- [systemd-resolved Manual](https://www.freedesktop.org/software/systemd/man/latest/resolvectl.html)
- [NetworkManager Documentation](https://networkmanager.dev/docs/)
- [MetalLB Documentation](https://metallb.universe.tf/)
- [Traefik Kubernetes Documentation](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
