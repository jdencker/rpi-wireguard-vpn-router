# WireGuard VPN Router (Raspberry Pi)

Conventionally, I name the device `vpn-router` on the local network. I assigned the Raspberry Pi a reserved local IP of `10.0.0.2`, and configured the VPN server to reside on `10.0.2.1/32` serving the VPN subnet `10.0.2.0/24`.

This guide assumes the Pi is hardwired to the router over Ethernet (`eth0`). If you run it over Wi‑Fi, switch the interface in the config to `wlan0` and ensure it is connected before applying routing changes. We still assign the reserved IP manually for clarity; this only matters in the local IP routing section, which has instructions for both. 

## Quickstart

1. Run `install_vpn.sh` on a fresh Raspberry Pi OS install (as root/sudo).  
2. Generate keys on the server: `wg genkey | tee server_privatekey | wg pubkey > server_publickey`.  
3. Fill in `wg0.conf` with your keys, chosen subnet, and interface (`eth0` or `wlan0`), then copy it to `/etc/wireguard/wg0.conf` 
4. `chmod 600 /etc/wireguard/wg0.conf`.  
5. Enable IPv4 forwarding in `/etc/sysctl.conf` by setting `net.ipv4.ip_forward=1`, then `sudo sysctl -p`.  
6. Forward UDP port `51820` on your router to the WireGuard server IP (10.0.2.1 in this example).  
7. Bring up the interface: `sudo wg-quick up wg0` and enable on boot: `sudo systemctl enable wg-quick@wg0`.  
8. For each client, generate a keypair, fill `client.conf` with its keys and assigned IP, and add a matching `[Peer]` block in `wg0.conf`. See these files for additional hints in the comments. 

## Security & Hygiene
- Never commit private keys, real endpoints, or client configs to source control.
- Set strict perms on configs: `chmod 600 /etc/wireguard/wg0.conf` and on any key files.
- Use descriptive placeholders in shared configs (as in the sample files) when sharing this repo publicly.

## Port Forwarding on Router

Enable port forwarding on your router and point incoming WireGuard UDP traffic (default `:51820`) to the WireGuard server IP (`10.0.2.1` in this example).

## Server Side Install

### Install Dependencies

Starting from a fresh install of Raspberry Pi OS, run the `install_vpn.sh` file to update the pi, and install the latest version of wireguard. Depending on you linux distribution, there are other packages that may or may not need to be installed. In this repository, the `install_vpn.sh` file should install all necessary dependencies. You will need to install these as root. 

### WireGuard Server Configuration File

#### PKI Generation
Generate public and private keys for your wireguard server:

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```
This command does the following:
	•	wg genkey: Generates a new private key.
	•	tee privatekey: Saves the private key to a file named privatekey.
	•	wg pubkey > publickey: Pipes the private key into the wg pubkey command to generate the corresponding public key and saves it to a file named publickey.

#### Server Configuation File

An example `wg0.conf` file is provided with this repository. This file needs to be put in `/etc/wireguard` and filled out with all required information - see file for helpful comments. 

#### Server Internal IP Forwarding

##### Wireguard Configuration File

The internal IP routing is stood up and down as part of this file in the `PostUp` and `PostDown` sections of the file.

NOTE: if you opt to run over wifi, the only thing that changes is the network interface in the commmand. In this case, replace `eth0` with `wlan0`. Of course, first check that this interface is connected by runnning `ip a` or `ifconfig`.

##### System Configuration File 

As the root user, `nano /etc/sysctl.conf` to open the system configuration file. Scroll down until you find the `net.ipv4.ip_forward=1` line.
If this line is commented, uncomment it, save, and then run `sudo sysctl -p` to apply the change.


### Start Wireguard Interface

- As root user, run `wg-quick up wg0`. 
- To ensure the interface is working run `wg show wg0`. 
- NOTE: you need to ensure that the client information is filled out first, at a minimum the `AllowedIPs` under the `[Peer]` section. 
- `sudo systemctl enable wg-quick@wg0` will enable `wg0` as a system service, so it will start anytime the PI reboots. 

## Client Side Setup

1. Install WireGuard on your client device. There is an app available on iPhone, Android, and can also be download for MacOS from the App Store, or downloaded for any OS from the [WireGuard Website](https://www.wireguard.com). This provides a nice UI and abstracts alot of the PKI away from the client.
2. Create a new tunnel from the client device. I have included an example `client.conf` in this repository. Make sure to fill it out with all the appropriate information
3. Save the configuration and activate the tunnel. If you see the `Latest Handshake` field, along with a comparable about of data sent/received, the connection is working.

## Troubleshooting

1. If you are sending and receiving traffic on your WG interface but your client isn't receiving traffic, you IP tables aren't working properly.

2. Local IP Routing - This is set up through the `PostUp` and `PostDown` sections of the wireguard configuration file. If the wg interface is up (check with `sudo wg show wg0`), the iptables *should* be configured. To check run `sudo iptables --list`. If they are not configured, each heading will be blank after you run the previous command. To manually configure them:

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

NOTE: if you opt to run over wifi, the only thing that changes is the network interface in the commmand. In this case, replace `eth0` with `wlan0`. Of course, first check that this interface is connected by runnning `ip a` or `ifconfig`.

3. To confirm the wireguard server is routing the traffic correctly over the `eth0` interface, you should run `sudo tcpdump -i eth0 host 8.8.8.8` on the server, and then `ping 8.8.8.8` from a connected client device. 

