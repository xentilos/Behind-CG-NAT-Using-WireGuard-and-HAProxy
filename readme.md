# Successfully Migrating a Server Room Behind CG-NAT Using WireGuard and HAProxy

Last week, I executed a significant server room transfer for one of my client. Moving their servers to a new location, presented unique challenges due to the new site's placement behind CG-NAT (Carrier Grade Network Address Translation), so no public IP. Here's how I tackled and overcame these challenges using WireGuard for secure tunneling and HAProxy for SMTP traffic management.

## The Challenge

Transferring server rooms is a daunting task in itself. The client's new location, operating behind CG-NAT, added layers of complexity. CG-NAT often complicates direct access to internal services like OpenVPN and SMTP. To overcome this, I devised a solution to route traffic effectively using WireGuard for secure tunneling and HAProxy for efficient load balancing and proxying of SMTP traffic.

## The Solution

I set up a Virtual Private Server (VPS) to terminate a WireGuard tunnel, allowing me to route traffic through this secure tunnel to the client's internal network services. Additionally, I configured HAProxy to manage SMTP traffic effectively.

## Network Layer Description

Before diving into the technical details, let's visualize the network layer involved in this setup:

```plaintext
Internet --> VPS --> WireGuard Tunnel --> pfSense
```

For mail, there is a Proxmox Mail Gateway on the local network, which handles SMTP traffic:

```plaintext
Internet --> VPS --> WireGuard Tunnel --> pfSense --> Proxmox Mail Gateway
```

### WireGuard Configuration

Below is the WireGuard configuration setup for routing OpenVPN traffic through the secure tunnel at the default port 1194:

```ini
[Interface]
PrivateKey = <YourPrivateKey>
Address = 10.69.69.1/24
ListenPort = 51820

# OpenVPN certificate-based connection
PostUp = iptables -t nat -A PREROUTING -p udp --dport 1194 -j DNAT --to-destination 10.69.69.5:1194
PostUp = iptables -t nat -A POSTROUTING -p udp -d 10.69.69.5 --dport 1194 -j SNAT --to-source 10.69.69.1
PostUp = iptables -A FORWARD -p udp -d 10.69.69.5 --dport 1194 -j ACCEPT

# Cleanup rules on interface down
PostDown = iptables -t nat -D PREROUTING -p udp --dport 1194 -j DNAT --to-destination 10.69.69.5:1194
PostDown = iptables -t nat -D POSTROUTING -p udp -d 10.69.69.5 --dport 1194 -j SNAT --to-source 10.69.69.1
PostDown = iptables -D FORWARD -p udp -d 10.69.69.5 --dport 1194 -j ACCEPT

[Peer]
PublicKey = <NewDC_PublicKey>
AllowedIPs = 10.69.69.5/32
PersistentKeepalive = 25
```

### Explanation of Configuration

Let's break down some of the key settings in this configuration:

- **ListenPort = 51820**: This is the port on which WireGuard listens for incoming connections. The default port is 51820.
- **PostUp and PostDown rules**: These `iptables` rules are essential for routing traffic correctly when the WireGuard interface comes up or goes down.

  - **PostUp**: The `PostUp` commands are executed when the WireGuard interface is brought up. In this case, they are used to set up `iptables` rules to direct OpenVPN traffic to the internal server.
    - `iptables -t nat -A PREROUTING -p udp --dport 1194 -j DNAT --to-destination 10.69.69.5:1194`: This line adds a rule to the `nat` table that changes the destination IP address of packets coming into the VPS on UDP port 1194 to the internal server's IP address (10.69.69.5).
    - `iptables -t nat -A POSTROUTING -p udp -d 10.69.69.5 --dport 1194 -j SNAT --to-source 10.69.69.1`: This rule ensures that the internal server sees the packets as coming from the WireGuard server's internal IP (10.69.69.1), rather than the original external IP.
    - `iptables -A FORWARD -p udp -d 10.69.69.5 --dport 1194 -j ACCEPT`: This rule allows the forwarding of OpenVPN traffic to the internal server.

  - **PostDown**: The `PostDown` commands are executed when the WireGuard interface is brought down. They remove the `iptables` rules added by the `PostUp` commands to ensure the system is left clean.
    - `iptables -t nat -D PREROUTING -p udp --dport 1194 -j DNAT --to-destination 10.69.69.5:1194`: This deletes the rule that redirected OpenVPN traffic to the internal server.
    - `iptables -t nat -D POSTROUTING -p udp -d 10.69.69.5 --dport 1194 -j SNAT --to-source 10.69.69.1`: This deletes the rule that rewrote the source address of the forwarded traffic.
    - `iptables -D FORWARD -p udp -d 10.69.69.5 --dport 1194 -j ACCEPT`: This removes the forwarding rule for OpenVPN traffic.

- **PersistentKeepalive = 25**: This setting ensures that the WireGuard tunnel remains active even if there is no traffic, preventing the connection from timing out in environments with NAT.

### HAProxy Configuration

I used HAProxy to handle SMTP traffic efficiently. Below is the configuration for HAProxy:

```ini
global
    maxconn 2000
    stats socket /tmp/haproxy.socket level admin expose-fd listeners
    uid 80
    gid 80
    nbthread 2
    hard-stop-after 15m
    chroot /tmp/haproxy_chroot
    daemon
    server-state-file /tmp/haproxy_server_state
    log /dev/log local0 info
    tune.ssl.default-dh-param 2048

defaults
    log global
    mode tcp
    option dontlognull
    option tcplog
    retries 3
    timeout connect 10s
    timeout client 30s
    timeout server 30s
    timeout queue 30s
    timeout check 5s

frontend email_smtp
    bind *:25 name smtp
    mode tcp
    log global
    timeout client 30000
    option tcplog
    default_backend emailPMG

backend emailPMG
    mode tcp
    id 102
    log global
    option tcp-check
    retries 3
    load-server-state-from-file global
    server pmgmailserver 10.69.69.5:25 id 101 send-proxy
```

### Explanation of HAProxy Configuration

- **global section**: 
  - `maxconn 2000`: Limits the maximum number of concurrent connections.
  - `stats socket /tmp/haproxy.socket level admin expose-fd listeners`: Configures a stats socket for administrative tasks.
  - `chroot /tmp/haproxy_chroot`: Changes the root directory to increase security.
  - `daemon`: Runs HAProxy as a background service.
  - `log /dev/log local0 info`: Configures logging to the syslog.

- **defaults section**: 
  - `mode tcp`: Sets the mode to TCP for handling raw TCP sessions.
  - `timeout connect 10s`, `timeout client 30s`, and `timeout server 30s`: Sets various timeout values to ensure the timely handling of connections.

- **frontend section**: 
  - `bind *:25 name smtp`: Binds to port 25 for incoming SMTP traffic.
  - `default_backend emailPMG`: Specifies the backend to which SMTP traffic should be forwarded.

- **backend section**: 
  - `server pmgmailserver 10.69.69.5:25 id 101 send-proxy`: Defines the backend server where SMTP traffic will be sent. The `send-proxy` option is used to send the original client’s IP address to the backend server.

### Additional Setup for Proxmox Mail Gateway
To ensure that the Proxmox Mail Gateway can correctly handle the traffic being proxied by HAProxy, additional setup is required:
```sh
mkdir /etc/pmg/templates
cp /var/lib/pmg/templates/main.cf.in /etc/pmg/templates/original_main.cf.in
echo '[% INCLUDE original_main.cf.in %]'            >  /etc/pmg/templates/main.cf.in
echo '# find me in /etc/pmg/templates/main.cf.in'   >> /etc/pmg/templates/main.cf.in
echo 'postscreen_upstream_proxy_protocol = haproxy' >> /etc/pmg/templates/main.cf.in
pmgconfig sync --restart 1
```
This setup involves:

Creating a custom templates directory: This prevents overwriting the default configuration files during updates.
Copying the existing configuration: By copying the original main.cf.in to the custom directory, any changes are preserved.
Modifying the configuration: Add the necessary configuration for HAProxy by including the original configuration and appending the required Proxy Protocol settings.
Syncing the configuration: Apply changes and restart the necessary services.


### Ansible Playbook for Automation

To streamline the deployment, I used an Ansible playbook:

```yaml
---
- name: Setup WireGuard and HAProxy on Ubuntu 24.04 LTS
  hosts: all
  become: yes

  vars:
    wg_private_key: "<your_wireguard_private_key>"
    new_dc_public_key: "<new_dc_public_key>"
    haproxy_user_id: 80
    haproxy_group_id: 80
    haproxy_chroot_dir: "/tmp/haproxy_chroot"

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes

    - name: Install WireGuard and HAProxy
      apt:
        name:
          - wireguard
          - wireguard-tools
          - iptables
          - haproxy
        state: present

    - name: Configure WireGuard
      template:
        src: wireguard.conf.j2
        dest: /etc/wireguard/wg0.conf
        mode: '0600'
        owner: root
        group: root

    - name: Enable and start WireGuard
      command: wg-quick up wg0
      register: wg_up

    - name: Ensure WireGuard is enabled on boot
      systemd:
        name: wg-quick@wg0
        enabled: yes
        state: started

    - name: Create HAProxy chroot directory
      file:
        path: "{{ haproxy_chroot_dir }}"
        state: directory
        owner: "{{ haproxy_user_id }}"
        group: "{{ haproxy_group_id }}"
        mode: '0755'

    - name: Ensure HAProxy is enabled and started
      systemd:
        name: haproxy
        enabled: yes
        state: started

    - name: Allow incoming traffic on port 1194 (OpenVPN) via iptables
      iptables:
        chain: INPUT
        protocol: udp
        destination_port: 1194
        jump: ACCEPT
        comment: "Allow incoming OpenVPN traffic"

    - name: Allow incoming traffic on port 25 (SMTP) via iptables
      iptables:
        chain: INPUT
        protocol: tcp
        destination_port: 25
        jump: ACCEPT
        comment: "Allow incoming SMTP traffic"

    - name: Save iptables rules
      command: /sbin/iptables-save
      register: iptables_save
```

## Conclusion

By configuring WireGuard and HAProxy, I effectively navigated the challenges posed by CG-NAT at the new server location. This setup ensures secure and reliable access to the client's internal services, streamlining their operations while maintaining robust security protocols.

If you have any questions or would like more details on specific aspects of this setup, feel free to ask!
