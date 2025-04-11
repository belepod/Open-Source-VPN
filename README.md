# Personal VPN with Network-Wide Ad Blocking (OpenVPN + Pi-hole on AWS)

## Project Overview

This project details the setup of a personal VPN server using **OpenVPN Access Server** hosted on **Amazon Web Services (AWS)**, enhanced with network-level ad and tracker blocking using **Pi-hole**. This combination provides a secure, private browsing experience from any device connected to the VPN, effectively eliminating most advertisements and malicious tracking domains across all apps and browsers on the client device.

## Features

*   **Secure VPN Tunnel:** All internet traffic from connected clients is securely routed through the OpenVPN server hosted on AWS, masking your public IP address and encrypting your connection (especially useful on public Wi-Fi).
*   **Network-Level Ad Blocking:** Pi-hole acts as a DNS sinkhole, intercepting DNS queries from VPN clients and blocking requests to known ad-serving and tracking domains *before* they even reach your device.
*   **Centralized Blocking:** Ad blocking works for *all* devices and applications using the VPN connection, without needing client-side ad blocker software.
*   **Cloud Hosted:** Leverages the reliability and scalability of AWS EC2.
*   **Customizable:** Allows for fine-tuning of blocklists, whitelists, and blacklists via the Pi-hole admin interface.

## Technologies Used

*   **Cloud Provider:** Amazon Web Services (AWS) - EC2 for virtual server hosting.
*   **VPN Software:** OpenVPN Access Server (via AWS Marketplace AMI)
*   **Ad Blocking Software:** Pi-hole
*   **Operating System:** Linux (Typically Ubuntu or Amazon Linux, depending on the OpenVPN AS AMI)

## Prerequisites

*   An AWS Account with permissions to create EC2 instances, Security Groups, and potentially Elastic IPs.
*   An SSH client (like PuTTY, Terminal, or Windows Subsystem for Linux) to connect to the EC2 instance.
*   An SSH key pair configured in your AWS region.
*   Basic familiarity with the Linux command line.
*   OpenVPN client software installed on the devices you want to connect (e.g., OpenVPN Connect for Windows, macOS, iOS, Android).

## Setup Steps

**1. Launch OpenVPN Access Server Instance on AWS**

*   Log in to your AWS Management Console.
*   Navigate to the EC2 service.
*   Click "Launch Instances".
*   Go to the "AWS Marketplace" tab and search for "OpenVPN Access Server".
*   Select the official OpenVPN Access Server offering (usually includes 2 free connected devices, suitable for personal use). Read the pricing details.
*   Choose an instance type (e.g., `t2.micro` or `t3.micro` is often sufficient for personal use).
*   Configure Instance Details: Choose your VPC and subnet. Ensure "Auto-assign Public IP" is enabled (or assign an Elastic IP later for a fixed address).
*   Configure Storage: Default settings are usually fine.
*   Add Tags: Optionally tag your instance (e.g., `Name: OpenVPN-Pihole-Server`).
*   **Configure Security Group:** This is crucial. Create a *new* security group with the following **inbound** rules:
    *   **SSH (TCP port 22):** Allow access from your current IP address (for management).
    *   **HTTPS (TCP port 443):** Allow access from `0.0.0.0/0` (or your specific IPs) for the OpenVPN client web UI.
    *   **TCP port 943:** Allow access from `0.0.0.0/0` (or your specific IPs) for the OpenVPN Admin Web UI.
    *   **UDP port 1194:** Allow access from `0.0.0.0/0` for the OpenVPN data channel (standard VPN port).
    *   **DNS (TCP port 53):** Allow access from `0.0.0.0/0` *or*, more securely, from your VPN client subnet CIDR (e.g., `172.27.224.0/20` - check your OpenVPN AS settings later) **(Needed for Pi-hole)**.
    *   **DNS (UDP port 53):** Allow access from `0.0.0.0/0` *or* your VPN client subnet CIDR **(Needed for Pi-hole)**.
    *   **(Optional) HTTP (TCP port 80):** Allow access from your VPN client subnet CIDR *only* if you need to access the Pi-hole admin UI via HTTP. Accessing via HTTPS is generally recommended if you configure it. *It's often best to access Pi-hole Admin via its Private IP while connected to the VPN.*
*   Review and Launch: Select your SSH key pair and launch the instance.
*   **(Optional but Recommended):** Allocate an Elastic IP address and associate it with your instance to ensure the public IP doesn't change upon stopping/starting.

**2. Initial OpenVPN Access Server Configuration**

*   Once the instance is running, find its Public IP address or Public DNS name from the EC2 console.
*   SSH into the instance:
    ```bash
    ssh -i /path/to/your/key.pem openvpnas@[Your-EC2-Instance-Public-IP]
    ```
    *(Note: The default username is often `openvpnas`, but check the Marketplace AMI instructions)*
*   The server might run an initial configuration script automatically. Follow the prompts. You will likely need to agree to the EULA and set the password for the primary `openvpn` admin user.
*   Take note of the **Admin UI address** (usually `https://[Your-EC2-Instance-Public-IP]:943/admin`) and the **Client UI address** (usually `https://[Your-EC2-Instance-Public-IP]:943/`).
*   Log in to the Admin UI using the `openvpn` username and the password you set. Perform any initial setup wizards.
*   Navigate the Admin UI and find the **Private IP address** of your EC2 instance (often under Network Settings or Status Overview). **Note this private IP down (e.g., `172.31.x.x`) - you will need it.**

**3. Install Pi-hole**

*   Ensure you are still connected via SSH to your EC2 instance.
*   Run the official Pi-hole installer:
    ```bash
    curl -sSL https://install.pi-hole.net | sudo bash
    ```
*   Follow the on-screen prompts carefully:
    *   It will check dependencies.
    *   Confirm the need for a **static IP address**. The installer should detect your instance's **private IP address**. Confirm this is correct.
    *   Choose an **Upstream DNS Provider** (e.g., Google, Cloudflare, Quad9). Pi-hole will forward non-blocked queries here.
    *   Accept the default **blocklists** for now (you can add more later).
    *   Install the **web admin interface** (Recommended: Yes).
    *   Install the **web server (`lighttpd`)** if prompted (Recommended: Yes).
    *   Enable **Query Logging** (Recommended: Yes).
    *   Select a privacy level.
    *   **CRITICAL:** Note down the **Pi-hole Admin Web Interface URL** and **Admin Password** displayed at the end of the installation.

**4. Integrate Pi-hole with OpenVPN Access Server**

*   Log in to your OpenVPN Access Server **Admin Web UI** (`https://[Your-EC2-Instance-Public-IP]:943/admin`).
*   Navigate to: **Configuration -> VPN Settings -> DNS Settings**.
*   Ensure **"Should client Internet traffic be routed through the VPN?"** is set to **Yes**.
*   Set **"Have clients use specific DNS servers"** to **Yes**.
*   For **"Primary DNS Server"**, enter the **Private IP address** of your EC2 instance (the one Pi-hole is running on, which you noted earlier).
*   For **"Secondary DNS Server"**, you can either leave it blank or enter the same **Private IP address** again to ensure clients *only* use Pi-hole. *Do not use a public DNS server here if you want comprehensive blocking.*
*   Click **"Save Settings"** at the bottom of the page.
*   Click **"Update Running Server"** in the banner that appears at the top.

**5. Verify Firewall Rules (Recap)**

*   Double-check that your AWS Security Group allows inbound traffic on **UDP port 53** and **TCP port 53** from the appropriate source (VPN client subnet recommended for better security, or `0.0.0.0/0` if needed).
*   If you are running a host-based firewall on the instance itself (like `ufw` or `firewalld`), ensure it also allows incoming traffic on ports 53/udp and 53/tcp from the VPN interface/subnet (e.g., `tun0` or `172.27.224.0/20`).
    ```bash
    # Example for ufw
    # sudo ufw status # Check status
    # sudo ufw allow 53/udp
    # sudo ufw allow 53/tcp
    # sudo ufw reload
    ```

## Usage Instructions

1.  **Download Client Profile:**
    *   Access the OpenVPN Access Server **Client UI** using a web browser: `https://[Your-EC2-Instance-Public-IP]:943/`
    *   Log in with a valid user account (you might need to create non-admin users in the Admin UI -> User Management -> User Permissions). The `openvpn` user often works by default unless configured otherwise.
    *   Download the appropriate connection profile for your operating system (e.g., `.ovpn` file for desktop/Android, or follow instructions for OpenVPN Connect import).

2.  **Import and Connect:**
    *   Import the downloaded `.ovpn` file into your OpenVPN client software (e.g., OpenVPN Connect, Tunnelblick on macOS, etc.).
    *   Connect to the VPN using the imported profile. Enter your VPN username and password when prompted.

3.  **Verify Connection and Ad Blocking:**
    *   Once connected, check your public IP address (e.g., by searching "what is my IP" in a search engine). It should now show the IP address of your AWS EC2 instance.
    *   Try browsing websites known for heavy advertising (news sites, etc.). You should see significantly fewer ads.
    *   Use an online ad blocker test tool like `https://d3ward.github.io/toolz/adblock.html` to see the effectiveness.

4.  **Access Pi-hole Admin Interface (Optional):**
    *   While connected to the VPN, open a web browser and navigate to `http://[Your-EC2-Instance-PRIVATE-IP]/admin`.
    *   Log in using the Pi-hole admin password you saved during installation.
    *   Here you can view statistics, see the query log (to confirm blocking), manage blocklists, whitelist/blacklist specific domains, etc.

## Troubleshooting Common Issues

*   **Ads Not Blocked:**
    *   Verify the VPN client is actually using the server's private IP for DNS (check client's network settings).
    *   Disconnect/reconnect the VPN client to ensure it gets the latest DNS settings.
    *   Check the Pi-hole Query Log via its web admin interface to see if queries are arriving and if ad domains are being marked as "Blocked".
    *   Ensure Pi-hole is running (`ssh` in and run `sudo systemctl status pihole-FTL` or `pihole status`).
    *   Update Pi-hole's blocklists (`ssh` in and run `pihole -g`).
    *   Check firewall rules (AWS Security Group and host firewall) for port 53 (TCP/UDP).
    *   Clear DNS cache on your client device (`ipconfig /flushdns` on Windows, etc.).
*   **Cannot Connect to VPN:**
    *   Check if the OpenVPN Access Server service is running (`ssh` in and check service status).
    *   Verify AWS Security Group allows traffic on UDP 1194 and TCP 443/943.
    *   Check OpenVPN client logs for specific error messages.
    *   Ensure you are using the correct username/password.
*   **Cannot Access Pi-hole Admin UI:**
    *   Ensure you are connected to the VPN.
    *   Use the server's **private** IP address in the URL (`http://[PRIVATE_IP]/admin`).
    *   Check if `lighttpd` web server is running (`ssh` in and run `sudo systemctl status lighttpd`).
    *   Check firewall rules allow HTTP traffic (TCP 80) *from the VPN client subnet* if accessing via HTTP.

## Future Enhancements / Other Features

*   **DNS over HTTPS (DoH) / DNS over TLS (DoT):** Configure Pi-hole to use encrypted upstream DNS requests (using `cloudflared`).
*   **More Blocklists:** Add specialized Pi-hole blocklists (malware, tracking, telemetry, etc.).
*   **Security Hardening:** Install `fail2ban`, further restrict Security Group rules, enable MFA on VPN login.
*   **Monitoring:** Set up AWS CloudWatch alarms or install `netdata`/`htop` for resource monitoring.
*   **Split Tunneling:** Configure OpenVPN AS for split tunneling if you *don't* want all traffic to go through the VPN (Note: Ad blocking via Pi-hole requires DNS traffic to go through).

## Contributing

This is primarily a setup guide for a personal project. However, if you find errors or have suggestions for improvement in the documentation, feel free to open an Issue or Pull Request.

## License

This project setup guide is released under the [MIT License](LICENSE). Note that the underlying software (OpenVPN Access Server, Pi-hole, AWS services) have their own respective licenses.
