# Privacy-Focused Self-Hosted Traccar Docker Stack for GPS Tracker 

This project provides a complete, self-contained Docker Compose architecture to reclaim ownership of your GPS tracking data.

### The Privacy Problem with Cheap Trackers

When you purchase a budget GPS tracker on eBay like the SinoTrack ST-901, it very often arrives pre-configured by default to transmit your real-time coordinates, vehicle status, and speed straight to a proprietary cloud server hosted in China (for *convenience*).

This stack closes that loophole completely by configuring the tracker to send its telemetry inside an encrypted VPN tunnel, and hosting your own private traccar server.

## Architecture Overview

To achieve total isolation with zero public port forwarding, this setup orchestrates three distinct services sharing a unified network footprint:
1.	**OpenVPN Client Sidecar**: The 1NCE cellular core network offers an [OpenVPN server free of charge](https://help.1nce.com/docs/network-services/network-services-vpn-service/) with their 10-year IoT SIM cards. We configure it to route incoming tracker traffic exclusively through the VPN interface, but allows the container space to access the public internet for cloud dependencies (split tunneling).
2.	**Traccar Server**: The open-source heart of the project. It uses the network stack of the OpenVPN container to securely bind to the 1NCE network and listen natively for the tracking protocol. It records coordinate history to the embedded file-based H2 database, but it could be configured to use a real DB if you have many trackers.
3.	**Cloudflare Tunnel (`cloudflared`)**: Creates a [secure, outbound-only tunnel to Cloudflare's edge proxy](https://developers.cloudflare.com/tunnel/). This allows you to securely access your Traccar web interface and use the official Traccar Manager mobile app from anywhere in the world using a custom domain, without ever opening inbound ports on your home firewall.

## Phase 1: Prerequisites and Portal Configurations

### Prerequisites
- [GPS tracker supported by Traccar](https://www.traccar.org/devices/) (I used a SinoTrack ST-901 4G).
- 1NCE IoT SIM Card
- Cloudflare account with a domain configured on their nameservers.
- Host machine running Docker and Docker Compose.

Before spinning up the Docker containers, you need to configure your external accounts and gather the required credentials.

### Cloudflare Tunnel Setup (Get the cloudflared Token)

1. Log in to your [Cloudflare Dashboard](https://dash.cloudflare.com/login) and navigate to **Zero Trust**.
2. In the left sidebar, go to **Networks** and select **Connectors**.
3. Click **Create a Tunnel**, select **cloudflared**, and give it a name (e.g., traccar-home).
4. Save the tunnel. On the next screen, look at the Choose environment section.
5. Select the Docker command, look for the long string of alphanumeric characters immediately following the `--token` flag. Copy this token string. You will need to paste it into your docker-compose environment variables.

### 1NCE Portal Setup (Configure OpenVPN)

> IMPORTANT: The VPN network is setup **per order**, so if you do like I did and order 1 SIM card to test things out, then order a couple more later, they're not going to be on the same VPN network. It's much easier to order all of your SIM cards at the same time.

1. Log in to your [1NCE Management Console](https://portal.1nce.com/).
2. Navigate to the **Configuration tab**, the select the **Breakout Settings**.
3. In manual mode, select the region closest to you. This is mandatory to enable OpenVPN.
4. After saving the change, scroll down to **OpenVPN Configuration**, then select **Linux/macOS**
5. Download the two OpenVPN configuration files that are generated for you. If you ever change the breakout region, you'll have to download these again.
6. Create a directory on your Docker host machine named `ovpn` and place these files inside it. 

## Phase 2: Server Deployment (Docker Compose)

Create your local project directories, build your configuration files, and input your specific tokens.
1.	Ensure you have a directory named `./ovpn/` containing your 1NCE configuration files.
2.	Ensure you create a basic directory named `./traccar/` containing a default traccar.xml configuration file mapping an embedded H2 database engine.
3.	Create your `docker-compose.yml` file. In the cloudflared service block, add your Cloudflare Tunnel Token to the environment or command section.
4.	Spin up the infrastructure using the following terminal command:
`
docker compose up -d
`
 
## Phase 3: Determining Your Server's Private IP

To tell your physical tracker where to send its data, you must find the private IP address that 1NCE assigned to your OpenVPN container.
1.	Run the following command to inspect the startup logs of your OpenVPN container:
docker compose logs openvpn
2.	Look through the output for lines matching the initialization sequence. You are looking for a line that resembles:
`
net_addr_ptp_v4_add: 10.70.129.201 peer 10.70.129.202 dev tun0
`

Alternatively, look for the incoming PUSH_REPLY line showing an ifconfig block:
`
ifconfig 10.70.129.201 10.70.129.202
`

  3.	Note down the first IP address listed (in this example, `10.70.129.201`). This is your Traccar server's private endpoint within the isolated cellular network.

## Phase 4: Tracker Configuration (Via 1NCE Dashboard)

With your private server IP handily identified, you can now redirect the tracker away from its factory routing.

> **WARNING**: There is no way around an initial transmission to the third-party server. To configure the device, you must insert the active SIM card and power it on. The moment it connects to the cellular network, it will immediately begin sending its location to the factory server until you successfully override the configuration. If keeping your home or garage location completely hidden from that third-party server is a privacy concern, you should perform the initial power-on and configuration sequence in an anonymous location.

Because 1NCE IoT SIM cards do not support receiving standard text messages sent from personal mobile devices, you must transmit these commands exclusively through the SMS tab located inside the 1NCE Management Console.

Assuming the default factory PIN code of 0000, send these sequential messages via the 1NCE dashboard interface:
1.	Set the private APN network for your provider:
`8030000 iot.1nce.net`
2.	Redirect the tracking server to your private OpenVPN server IP address on the port required by your specific device protocol (e.g., port 5013 for the SinoTrack h02 protocol): `8040000 5013`
3. Send the `RCONF` SMS command, where you can verify that the settings took place. Also note the ID of the tracker, we'll use it when configuring traccar.

## Finishing Up

Once the tracker processes your commands, navigate back to your Cloudflare Zero Trust panel. Edit your tunnel configuration to point a custom public subdomain (`traccar.yourdomain.com`) to the internal hostname and port `openvpn:8082`. Open that URL in your browser, log into your secure dashboard, input the 10-digit Device ID found on your physical tracker, and begin mapping your trips privately.

## Optional: Enabling Native Push Notifications

You can configure your private server to send native push notifications directly to the official Traccar Manager app on your iPhone or Android device.
1.	Go to the official traccar.org website and register for a free account.
2.	Go to your Account page on the website and copy your personal API Key.
3.	Add the following lines to your local `./traccar/traccar.xml` file: web,traccar `YOUR_API_KEY_HERE`
4.	Restart your Traccar container:
docker compose restart traccar
5.	Log into your private server using the Traccar Manager mobile app, then configure your desired alerts in the Settings menu, making sure to select the "Notifications" channel.
