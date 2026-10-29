# Internet Pi

[![CI](https://github.com/AntoineAbhay/internet-pi/workflows/CI/badge.svg?event=push)](https://github.com/AntoineAbhay/internet-pi/actions?query=workflow%3ACI)

**A Raspberry Pi Configuration for Internet connectivity**

I have had a couple Pis doing random Internet-related duties for years. It's finally time to formalize their configs and make all the DNS/ad-blocking/monitoring stuff encapsulated into one Ansible project.

So that's what this is.

## Features

**Internet Monitoring**: Installs Prometheus and Grafana, along with a few Docker containers to monitor your Internet connection with Speedtest.net speedtests and HTTP tests so you can see uptime, ping stats, and speedtest results over time.

![Internet Monitoring Dashboard in Grafana](/images/internet-monitoring.png)

**Pi-hole**: Installs the Pi-hole Docker configuration so you can use Pi-hole for network-wide ad-blocking and local DNS. Make sure to update your network router config to direct all DNS queries through your Raspberry Pi if you want to use Pi-hole effectively.

**Unbound**: Installs Unbound DNS server for recursive, privacy-respecting DNS resolution.

**Traefik**: Installs Traefik reverse proxy with automated Let's Encrypt TLS certificates (via DNS challenge) to route and secure all web services on your domain.

**Wireguard**: Installs Wireguard VPN. Add `wireguard_peers` in the config to add VPN clients.

![Pi-hole on the Internet Pi](/images/pi-hole.png)

Other features:

  - **Nextcloud**: Installs Nextcloud All-in-One routed via Traefik with configurable storage mounts (`nextcloud_mount`). (Disabled by default. Enable using `nextcloud_enable` in `config.yml`.)
  - **Shelly Plug Monitoring**: Installs a [`shelly-plug-prometheus` exporter](https://github.com/geerlingguy/shelly-plug-prometheus) and a Grafana dashboard, which tracks and displays power usage on a Shelly Plug running on the local network. (Disabled by default. Enable and configure using the `shelly_plug_*` vars in `config.yml`.)
  - **AirGradient Monitoring**: Configures [`airgradient-prometheus`](https://github.com/geerlingguy/airgradient-prometheus) and a Grafana dashboard, which tracks and displays air quality over time via one or more AirGradient DIY monitors. (Disabled by default. Enable and configure using the `airgradient_enable` var in `config.yml`. See example configuration for ability to monitor multiple AirGradient DIY stations.)
  - **Starlink Monitoring**: Installs a [`starlink` prometheus exporter](https://github.com/danopstech/starlink_exporter) and a Grafana dashboard, which tracks and displays Starlink statistics. (Disabled by default. Enable and configure using the `starlink_enable` var in `config.yml`.)
  - **Home Assistant**: Installs Home Assistant Container routed via Traefik. (Disabled by default. Enable and configure using `homeassistant_enable` in `config.yml`. See [`docs/home-assistant-setup.md`](docs/home-assistant-setup.md) for initial setup and reverse proxy configuration, and [`docs/home-assistant-water-heater-guide.md`](docs/home-assistant-water-heater-guide.md) for managing connected water heaters via Cozytouch / Overkiz).

**IMPORTANT NOTE**: If you use the included Internet monitoring, it will download a decently-large amount of data through your Internet connection on a daily basis. Don't use it, or tune the `internet-monitoring` setup to not run the speedtests as often, if you have a metered connection!

## Recommended Pi and OS

You should use a Raspberry Pi 4 model B or better. The Pi 4 and later generations of Pi include a full gigabit network interface and enough I/O to reliably measure fast Internet connections.

Older Pis work, but have many limitations, like a slower CPU and sometimes very-slow NICs that limit the speed test capability to 100 Mbps or 300 Mbps on the Pi 3 model B+.

Other computers and VMs may run this configuration as well, but it is only regularly tested on a Raspberry Pi.

The configuration is tested against Raspberry Pi OS, both 64-bit and 32-bit, and runs great on that or a generic Debian installation.

It should also work with Ubuntu for Pi, or Arch Linux, but has not been tested on other operating systems.

## Setup

  1. [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html). The easiest way (especially on Pi or a Debian system) is via Pip:
     1. (If on Pi/Debian): `sudo apt-get install -y python3-pip`
     2. (Everywhere): `pip3 install ansible`
  2. Clone this repository: `git clone https://github.com/AntoineAbhay/internet-pi.git`, then enter the repository directory: `cd internet-pi`.
  3. Install requirements: `ansible-galaxy collection install -r requirements.yml` (if you see `ansible-galaxy: command not found`, restart your SSH session or reboot the Pi and try again)
  4. Make copies of the following files and customize them to your liking:
     - `example.inventory.ini` to `inventory.ini` (replace IP address with your Pi's IP, or comment that line and uncomment the `connection=local` line if you're running it on the Pi you're setting up).
     - `example.config.yml` to `config.yml`
  5. Run the playbook: `ansible-playbook main.yml`

> **If running locally on the Pi**: You may encounter an error like "Error while fetching server API version". If you do, please either reboot or log out and log back in, then run the playbook again.

## Usage

All web interfaces are routed securely through Traefik using HTTPS under your configured domain (`my_domain` in `config.yml`). Raw HTTP ports (like `:3030` for Grafana or `:80` for Pi-hole web) are intentionally not published to the host for reverse-proxy security and consistency.

### Web Services Access
- **Grafana**: Access at `https://grafana.<my_domain>` using username `admin` and your `monitoring_grafana_admin_password`.
- **Pi-hole Web Interface**: Access at `https://pihole.<my_domain>` and log in with your `pihole_password`.
- **Traefik Dashboard**: Access at `https://traefik.<my_domain>`.
- **Home Assistant**: (If enabled) Access at `https://homeassistant.<my_domain>` (see [Home Assistant Water Heater Guide](docs/home-assistant-water-heater-guide.md)).
- **Nextcloud**: (If enabled) Access at `https://nextcloud.<my_domain>`.

### DNS & Network Services
- **DNS (Pi-hole + Unbound)**: Listen directly on the Pi's IP address on port `53/tcp` and `53/udp`. Configure your network router's DHCP settings to distribute the Pi's IP address as your primary DNS server.
- **Wireguard VPN**: Connect to your Wireguard endpoint on port `51820/udp`.

## Updating

To manually pull new container images and recreate a service (e.g. Pi-hole), run:

```bash
cd ~/pi-hole
docker compose pull             # pulls the latest images
docker compose up -d --no-deps  # restarts containers with newer images
docker system prune -f          # cleans up unused dangling images
```

Upgrades for the other configurations are identical (navigate into the desired folder in your `config_dir` and run the same `docker compose` commands). Alternatively, re-running the Ansible playbook (`ansible-playbook -i inventory.ini main.yml`) will ensure all containers and configurations are up to date.

## Backups

A guide for backing up the configurations and historical data will be posted here as part of [Issue #194: Create Backup guide](https://github.com/geerlingguy/internet-pi/issues/194).

## License

MIT

## Author

This project was created in 2021 by [Jeff Geerling](https://www.jeffgeerling.com/).
