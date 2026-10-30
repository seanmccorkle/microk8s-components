# Shinobi

## microk8s

```BASH
microk8s enable dns hostpath-storage ingress
```

## Expose to the Internet

### Cloudflare Tunnels

A Cloudflare Tunnel establishes a secure, encrypted link between your local Ubuntu machine and the Cloudflare edge network, bypassing port forwarding altogether

1. Create a free account at Cloudflare and register a custom domain name
2. In your Cloudflare Dashboard, go to `Zero Trust > Networks > Tunnels`
3. Select `Create a Tunnel`, name it, and follow the instructions to install the cloudflared utility natively on your Ubuntu server or deploy it as a pod inside MicroK8s
4. Route your custom domain (e.g., ://yourdomain.com) in the Cloudflare settings to point directly to http://localhost:8080 or your cluster's local ingress IP address


### Tailscale VPN (Private Remote Access)

If you do not want your dashboard indexed by public search engines, install [Tailscale](https://tailscale.com/) on your Ubuntu server

1. Install Tailscale on Ubuntu and authenticate your machine
2. Install Tailscale on your mobile phone or laptop
3. Access your Shinobi instance from anywhere in the world using the private Tailscale IP address assigned to your Ubuntu machine (e.g., http://100.x.y.z:8080)

## References
[How to Access Shinobi outside your Network (LAN)](https://hub.shinobi.video/articles/view/pYUnteHIep5wUS0)<br>
[Link 2](docs.shinobi.video/remote/vpn)<br>