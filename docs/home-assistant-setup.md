# Home Assistant Initial Setup & Reverse Proxy Configuration Guide

This guide walks you through the first-time setup and reverse proxy configuration of Home Assistant on your Raspberry Pi.

---

## Why Direct Port Access is Needed for First Setup

Starting with **Home Assistant 2026.8**, static `http:` reverse proxy configuration in `configuration.yaml` is deprecated and moved exclusively to the **Web UI**.

By default, a clean Home Assistant installation rejects any request coming through a reverse proxy (such as Traefik) with `400: Bad Request` until the proxy is explicitly trusted. To avoid a chicken-and-egg problem where you cannot reach the UI behind Traefik to configure the reverse proxy, port `8123` is exposed directly to the local network during initial setup.

---

## Step-by-Step Initial Setup Procedure

### Step 1: Enable Home Assistant in Ansible Configuration

In your `config.yml`:

```yaml
# Home Assistant configuration
homeassistant_enable: true
homeassistant_hostname: homeassistant
homeassistant_timezone: "Europe/Paris"
homeassistant_expose_port: true   # Keep true for initial setup
```

Run the Ansible playbook to deploy:

```bash
ansible-playbook -i inventory.ini main.yml
```

### Step 2: Access Onboarding via Local LAN IP

1. In your browser, open Home Assistant **directly** using the Pi's local IP address and port `8123` (bypassing Traefik):
   ```text
   http://<YOUR_PI_IP>:8123
   ```
   *(e.g., `http://192.168.1.10:8123`)*

2. Follow the onboarding wizard:
   - Create your owner / administrator account (Name, Username, Password).
   - Set your home name and location.
   - Click **Finish**.

### Step 3: Configure Reverse Proxy Trust in the Web UI

Now that you have access to the UI, authorize Traefik as a trusted reverse proxy:

1. In the left sidebar, click **Settings** (*Paramètres*).
2. Go to **System** (*Système*) → **Network** (*Réseau*).
3. Scroll down and click to expand the **HTTP server** (*Serveur HTTP*) section.
4. Locate the **Reverse proxy** (*Serveur mandataire inverse*) subsection:
   - Toggle **Trust X-Forwarded-For** (*Faire confiance à X-Forwarded-For*) to **ON**.
   - Under **Trusted proxies** (*Serveurs mandataires de confiance*), click **Add** and enter your Docker network subnet:
     ```text
     172.16.0.0/12
     ```
     *(This covers Docker's standard bridge network ranges).*
5. Click **Save** (*Enregistrer*) at the bottom of the section.
6. Restart Home Assistant:
   - Go to **Developer Tools** (*Outils de développement*) → **YAML** tab → **Restart** (*Redémarrer*), or:
   - In **Settings** → **System**, click the power icon in the top right and select **Restart Home Assistant**.

### Step 4: Verify Access via Traefik

Once restarted, verify that you can access Home Assistant through Traefik using your domain and HTTPS:

```text
https://homeassistant.<my_domain>
```

You should now see the Home Assistant login screen without any 400 Bad Request error.

---

## (Optional) Step 5: Close Port 8123

If you want Home Assistant to be reachable **only** through Traefik (HTTPS):

1. Edit your `config.yml`:
   ```yaml
   homeassistant_expose_port: false
   ```
2. Re-run the Ansible playbook:
   ```bash
   ansible-playbook -i inventory.ini main.yml
   ```

*Note: Leaving `homeassistant_expose_port: true` is also safe if your Pi is on a trusted local LAN and port 8123 is not forwarded on your internet router. It provides a convenient fallback if Traefik or DNS certificates ever encounter an issue.*

---

## Next Steps

- To configure smart water heaters and off-peak / peak hour automations, see the [Water Heater Automation Guide](home-assistant-water-heater-guide.md).
