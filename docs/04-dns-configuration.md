# Step 4 — Configure a Local DNS Server with BIND9 for Harbor on Kubernetes

## Overview

This guide explains how to install and configure BIND9 as a local DNS server for accessing Harbor through the hostname `harbor.k8s.local` in a Kubernetes cluster.

The DNS server resolves the Harbor hostname to the external IP address assigned to Traefik by MetalLB. This allows clients on the local network to access Harbor using a hostname instead of an IP address, without manually adding a hosts-file entry on every client.

This configuration prepares local name resolution for the Harbor deployment. HTTPS access also requires the corresponding Traefik routing configuration and a valid TLS certificate.

## 1. Lab Environment

The following table describes the example environment used in this guide.

| Component | Example value |
|---|---|
| DNS server | `192.168.1.20` |
| Local network | `192.168.1.0/24` |
| Harbor hostname | `harbor.k8s.local` |
| Traefik external IP | `192.168.1.240` |
| DNS server interface | `ens33` |
| Example Windows client | `192.168.1.224` |

**Important:** These are example values. Verify the actual DNS server address, network interface, and external IP assigned to Traefik before applying the configuration.

The DNS server address must be reachable by clients on the local network. The Harbor DNS record must point to the current external IP address of the Traefik Service.

## 2. Identify the Traefik External IP

Before configuring DNS, identify the external IP address assigned to Traefik by MetalLB.

Run the following command on a machine with access to the Kubernetes cluster:

```bash
kubectl get svc -n traefik
```



Locate the Traefik Service and note the value displayed in the `EXTERNAL-IP` column.

This is the address that the DNS record for `harbor.k8s.local` must resolve to.

If the external IP is shown as `Pending`, verify the MetalLB installation, IP address pool, and Layer 2 advertisement before continuing.

**Example:** If Traefik has been assigned `192.168.1.240`, the Harbor DNS record must point to `192.168.1.240`.

## 3. Check the DNS Server Network Configuration

On the Ubuntu machine that will host BIND9, inspect the network configuration:

```bash
ip -br address
ip route
nmcli connection show
```


These commands help identify:

- The IP address assigned to the DNS server.
- The network interface connected to the local network.
- The default gateway and routing configuration.
- The NetworkManager connection name, if NetworkManager is in use.

In the following examples, the DNS server address is `192.168.1.20` and the local subnet is `192.168.1.0/24`.

Replace these values if your environment uses a different network configuration.

## 4. Install BIND9

### 4.1. Update the package index

Refresh the Ubuntu package index:

```bash
sudo apt update
```

### 4.2. Install BIND9 and DNS utilities

Install the DNS server and diagnostic tools:

```bash
sudo apt install -y bind9 bind9-utils dnsutils
```

The packages provide BIND9, configuration-validation utilities, and commands such as `dig` for testing DNS resolution.

### 4.3. Verify the BIND9 service

Check the service status:

```bash
systemctl status named --no-pager
```

![Screenshot: BIND9 Service Status](../images/dns/05-bind9-service-status.png)

On Ubuntu, the service may be available under the name `named` or `bind9`, depending on the installation and service-unit configuration.

If `named` is not recognized, check the alternative service name:

```bash
systemctl status bind9 --no-pager
```

Use the service unit that exists on your system in the subsequent steps.

If the service is not active, inspect the reported errors before proceeding.

## 5. Configure BIND9 Options

BIND9 must listen on the appropriate network addresses and restrict DNS queries to trusted clients.

### 5.1. Back up the existing configuration

Create a backup of the original options file:

```bash
sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.bak
```



The backup allows you to restore the previous configuration if necessary.

### 5.2. Edit the options file

Open the configuration file:

```bash
sudo nano /etc/bind/named.conf.options
```

Configure BIND9 as follows:

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

Replace `192.168.1.20` with the actual LAN address of the DNS server if it differs.

The configuration performs the following functions:

- **`acl "trusted"`** defines the clients allowed to query the DNS server.
- **`listen-on`** specifies the IPv4 addresses on which BIND9 listens.
- **`listen-on-v6 { none; }`** disables IPv6 listening for this configuration.
- **`allow-query`** restricts DNS queries to trusted clients.
- **`allow-recursion`** restricts recursive DNS requests to trusted clients.
- **`forwarders`** specifies upstream DNS resolvers for queries that BIND9 must forward.
- **`forward only`** instructs BIND9 to forward queries rather than attempting direct resolution if forwarding fails.
- **`dnssec-validation auto`** enables automatic DNSSEC validation for applicable DNS resolution.

The local zone `k8s.local` will be configured separately in the next section.

**Security note:** The trusted ACL allows queries from localhost and the specified local subnet. Do not expose a recursive DNS resolver to untrusted networks. Review the permitted subnet and firewall rules before deploying this configuration.

## 6. Declare the DNS Zone

A DNS zone defines the namespace for which the DNS server is authoritative. In this guide, the zone is `k8s.local`.

### 6.1. Back up the local configuration

Back up the existing local configuration file:

```bash
sudo cp /etc/bind/named.conf.local /etc/bind/named.conf.local.bak
```



### 6.2. Edit the local configuration

Open the file:

```bash
sudo nano /etc/bind/named.conf.local
```

Add the following zone declaration:

```conf
zone "k8s.local" {
    type primary;
    file "/etc/bind/db.k8s.local";
    allow-query { trusted; };
    allow-update { none; };
};
```

This declaration configures BIND9 as the primary DNS server for `k8s.local` and specifies the file containing the zone records.

The `allow-update { none; };` directive disables dynamic DNS updates for this zone.

The zone uses the previously defined `trusted` ACL to restrict queries. Ensure that the ACL is defined in a configuration file loaded by BIND9 before this zone declaration is evaluated.

## 7. Create the DNS Zone File

The zone file contains the DNS records used to resolve the Harbor hostname.

### 7.1. Create the zone file

Create or edit the zone file:

```bash
sudo nano /etc/bind/db.k8s.local
```

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

**Important:** Replace `192.168.1.240` with the actual external IP address assigned to Traefik in Step 2. The `ns` record must point to the DNS server's actual LAN address.

### 7.2. Understand the DNS records

| Record | Purpose |
|---|---|
| `SOA` | Defines the zone's primary authority and timing parameters. |
| `NS` | Identifies the authoritative DNS server for the zone. |
| `ns A` | Resolves the DNS server hostname to its IPv4 address. |
| `harbor A` | Resolves `harbor.k8s.local` to the Traefik external IP. |

The SOA serial number identifies the version of the zone. If you modify the zone file later, increment the serial number so that the change can be tracked correctly.

### 7.3. Set file ownership and permissions

Apply the expected ownership and permissions:

```bash
sudo chown root:bind /etc/bind/db.k8s.local
sudo chmod 644 /etc/bind/db.k8s.local
```

These permissions allow the zone file to be readable by the BIND group while remaining owned by root.

### Note about the `.local` suffix

The `.local` suffix is reserved for Multicast DNS (mDNS). Some operating systems and clients may handle `.local` names through mDNS rather than conventional unicast DNS, which can cause inconsistent resolution behavior.

For a new deployment, consider using a subdomain of a domain you own or `home.arpa` for a home network. If you change the suffix, update the BIND zone declaration, DNS records, client configuration, and TLS certificate consistently.

If you keep `harbor.k8s.local` for this lab, test resolution from each client operating system rather than assuming all clients will use the configured DNS server for that name.

## 8. Validate the BIND9 Configuration

Before restarting BIND9, validate the configuration syntax and the DNS zone.

### 8.1. Check the main configuration

Run:

```bash
sudo named-checkconf
```

No output normally indicates that the configuration syntax is valid.

If an error is reported, correct the indicated configuration issue before continuing.

### 8.2. Validate the DNS zone

Run:

```bash
sudo named-checkzone k8s.local /etc/bind/db.k8s.local
```

![Screenshot: DNS Zone Validation](../images/dns/16-named-checkzone.png)

The command should report that the zone loaded successfully and end with `OK`.

If validation fails, review the zone declaration, record syntax, trailing dots in fully qualified domain names, and SOA serial format.

Do not restart BIND9 until both configuration checks have completed successfully.

## 9. Restart and Verify BIND9

After validating the configuration, restart the appropriate service.

### 9.1. Restart the service

If the service unit is named `named`, run:

```bash
sudo systemctl restart named
sudo systemctl status named --no-pager
```

If your system uses the `bind9` service unit instead, run:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9 --no-pager
```

Verify that the service is active and that no configuration errors are reported.

### 9.2. Inspect service logs if startup fails

For a `named` service unit:

```bash
sudo journalctl -u named -n 50 --no-pager
```

For a `bind9` service unit:

```bash
sudo journalctl -u bind9 -n 50 --no-pager
```

Review the logs for configuration errors, invalid zone records, permission problems, or port-binding issues.

## 10. Test DNS Resolution on the DNS Server

Test the DNS records directly against BIND9 before configuring client machines.

### 10.1. Query the loopback address

Run:

```bash
dig @127.0.0.1 harbor.k8s.local
```

![Screenshot: Local DNS Resolution](../images/dns/19-dig-localhost.png)

Inspect the `ANSWER SECTION`. It should contain an A record for `harbor.k8s.local` pointing to the current external IP address of Traefik.

### 10.2. Query the DNS server's LAN address

Test the server's LAN address:

```bash
dig @192.168.1.20 harbor.k8s.local
```

![Screenshot: DNS Resolution Using the Server IP](../images/dns/20-dig-server-ip.png)

The answer should contain the same IP address returned by the loopback query.

The `@192.168.1.20` argument explicitly directs the query to the DNS server, so this test does not depend on the machine's default DNS resolver configuration.

### 10.3. Troubleshoot an incorrect answer

If the returned IP address is incorrect:

1. Verify the `harbor` A record in `/etc/bind/db.k8s.local`.
2. Confirm that the record points to the current Traefik external IP.
3. Increment the SOA serial number if the zone file has been modified.
4. Revalidate the zone with `named-checkzone`.
5. Reload or restart the appropriate BIND9 service.
6. Repeat the `dig` tests.

If the query times out, verify that BIND9 is running, listening on the expected address, and permitted through the firewall.

## 11. Configure Firewall Rules (If UFW Is Enabled)

Check the firewall status:

```bash
sudo ufw status verbose
```

If UFW is active, allow DNS requests from the local subnet to the DNS server over both UDP and TCP port 53:

```bash
sudo ufw allow from 192.168.1.0/24 to 192.168.1.20 port 53 proto udp
sudo ufw allow from 192.168.1.0/24 to 192.168.1.20 port 53 proto tcp
sudo ufw status numbered
```

UDP is commonly used for DNS queries, while TCP is also required for certain DNS responses and operations.

Do not enable UFW without reviewing the existing rules and required access, especially SSH, NFS, and Kubernetes-related traffic. If another firewall is used, configure equivalent rules there.

## 12. Configure DNS on a Linux Client

Configure a client to use the BIND9 server so that it can resolve `harbor.k8s.local`.

### 12.1. Inspect the client network configuration

On a Linux client using `systemd-resolved`, inspect the current resolver configuration and interfaces:

```bash
resolvectl status
ip -br address
```

Identify the interface connected to the local network. The examples below use `ens33`; replace it with the actual interface name on the client.

### 12.2. Apply a temporary DNS configuration

Set the DNS server for the interface:

```bash
sudo resolvectl dns ens33 192.168.1.20
sudo resolvectl domain ens33 '~k8s.local'
sudo resolvectl flush-caches
```

The `~k8s.local` setting configures a routing-only DNS domain so queries for this domain are directed to the DNS server associated with the interface.

Test the result:

```bash
resolvectl query harbor.k8s.local
```

These settings may not persist after a reboot or network reconnection.

### 12.3. Configure persistent DNS settings with NetworkManager

First, identify the actual NetworkManager connection name:

```bash
nmcli connection show
```

Replace `netplan-ens33` in the following commands with the connection name shown on your system:

```bash
sudo nmcli connection modify "netplan-ens33" ipv4.dns "192.168.1.20"
sudo nmcli connection modify "netplan-ens33" ipv4.dns-search "~k8s.local"
sudo nmcli connection modify "netplan-ens33" ipv4.ignore-auto-dns yes
```

Apply the changes to the interface:

```bash
sudo nmcli device reapply ens33
```

Use the actual interface name instead of `ens33` if it differs.

**Note:** NetworkManager settings can interact with existing Netplan and DHCP configuration. Review the connection settings before applying changes. Network changes may interrupt connectivity, so perform this step from a local console or ensure you have another way to recover access.

### 12.4. Verify resolution from the client

Run:

```bash
resolvectl query harbor.k8s.local
```

The returned address should match the current external IP assigned to Traefik.

You can also query the DNS server directly:

```bash
dig @192.168.1.20 harbor.k8s.local
```

A successful direct query verifies that the client can reach the DNS server and obtain the expected record. To verify that ordinary application requests use the configured resolver, test the hostname without explicitly specifying a DNS server.

## 13. Next Steps: Harbor HTTPS Access

After local DNS resolution is working, continue with the Harbor deployment:

1. Configure the Harbor Helm chart and persistent storage.
2. Configure Traefik Ingress routing for `harbor.k8s.local`.
3. Configure the TLS certificate and corresponding certificate trust on client machines.
4. Verify that the hostname resolves to Traefik's external IP.
5. Test HTTPS access from the client.

DNS resolution alone does not configure the Ingress route or TLS certificate. Those components must also be configured before Harbor can be accessed successfully over HTTPS.

## References

- [BIND 9 Documentation](https://bind9.readthedocs.io/)
- [Ubuntu Server Documentation](https://documentation.ubuntu.com/server/)
- [systemd-resolved Manual](https://www.freedesktop.org/software/systemd/man/latest/resolvectl.html)
- [NetworkManager Documentation](https://networkmanager.dev/docs/)
- [MetalLB Documentation](https://metallb.io/configuration/)
- [Traefik Kubernetes Documentation](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)
