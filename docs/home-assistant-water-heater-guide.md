# Guide: Controlling a Connected Water Heater with Home Assistant (Overkiz / Cozytouch)

This guide explains how to integrate and automate a connected electric water heater in Home Assistant, specifically to optimize power consumption during off-peak hours.

---

## Compatibility & Ecosystem

This integration relies on the **Overkiz** cloud API, which is natively supported in Home Assistant core. It supports connected devices (Wi-Fi or Cozytouch bridge) from brands including:
- **Groupe Atlantic**: **Atlantic**, **Thermor**, **Sauter** (via the Cozytouch ecosystem)
- **Somfy** (via TaHoma / Connexoon box)
- **Hitachi** (Hi-Kumo / Cozytouch compatible equipment)

---

## 1. Deployment & First Login to Home Assistant

For detailed first-time setup instructions and reverse proxy configuration, refer to the [Home Assistant Initial Setup Guide](home-assistant-setup.md).

Summary of steps:
1. Ensure `homeassistant_enable: true` and `homeassistant_expose_port: true` are set in your `config.yml`.
2. Deploy the configuration with Ansible:
   ```bash
   ansible-playbook -i inventory.ini main.yml
   ```
3. Complete the initial onboarding directly via `http://<PI_IP>:8123`.
4. In **Settings** > **System** > **Network** > **HTTP Server** > **Reverse proxy**, enable **Trust X-Forwarded-For** and add `172.16.0.0/12` to **Trusted proxies**, then restart Home Assistant.
5. You can now access Home Assistant securely through Traefik at `https://homeassistant.<your_domain>`.
6. Follow the steps below to configure your water heater integration.

---

## 2. Setting Up the Overkiz Integration

1. In Home Assistant, navigate to **Settings** > **Devices & services**.
2. Click **Add integration** in the bottom right corner and search for **Overkiz**.
3. Select your provider hub from the list (for example, **Atlantic Cozytouch** for Atlantic, Thermor, or Sauter devices).
4. Enter the credentials (email and password) associated with your mobile app account.
5. Submit: Home Assistant automatically discovers your devices. Your water heater will appear as an entity in the `water_heater` domain (e.g. `water_heater.water_heater`). Note its entity identifier (`entity_id`).

---

## 3. Off-Peak / Peak Hours Automation

### Principle
Modern connected water heaters often have automatic or eco modes that may trigger reheating during peak hours when a significant amount of hot water is drawn.
To restrict heating strictly to off-peak periods:
- **At off-peak start**: set the water heater to manual mode with your normal target setpoint.
- **At peak hours start**: set the water heater to standby/away mode (`away`, `off`, or minimum setpoint) to prevent heating during expensive hours.

### Example YAML Automation

In **Settings** > **Automations & scenes** > **Create automation** > **Edit in YAML**, customize the example below with your specific off-peak hours and entity ID:

```yaml
alias: "Water Heater - Peak / Off-Peak Scheduling"
description: "Heats the water heater only during off-peak hours and sets it to standby during peak hours"
trigger:
  # Off-peak window starts
  - trigger: time
    at: "00:00:00"
    id: "off_peak_night_start"
  - trigger: time
    at: "14:00:00"
    id: "off_peak_afternoon_start"

  # Peak window starts (off-peak ends)
  - trigger: time
    at: "06:00:00"
    id: "peak_morning_start"
  - trigger: time
    at: "16:00:00"
    id: "peak_afternoon_start"

condition: []

action:
  - choose:
      # Off-peak: switch to manual mode / regular setpoint
      - conditions:
          - condition: trigger
            id:
              - off_peak_night_start
              - off_peak_afternoon_start
        sequence:
          - action: water_heater.set_operation_mode
            target:
              entity_id: water_heater.water_heater
            data:
              operation_mode: "manual"

      # Peak hours: switch to standby/away mode to stop heating
      - conditions:
          - condition: trigger
            id:
              - peak_morning_start
              - peak_afternoon_start
        sequence:
          - action: water_heater.set_operation_mode
            target:
              entity_id: water_heater.water_heater
            data:
              operation_mode: "away"
    default: []
mode: single
```

---

## Best Practices & Tips

1. **Verify supported modes for your model**:
   In Home Assistant, go to **Developer Tools** > **States** tab and inspect the `operation_list` attribute of your `water_heater` entity. Depending on the model and firmware, the standby mode might be named `away`, `off`, `eco`, or `frostprotection`.
2. **Using a Schedule Helper**:
   Instead of static time triggers, you can create a graphical schedule helper in **Settings** > **Devices & services** > **Helpers** > **Schedule**, define your off-peak time slots visually, and trigger automations on schedule state changes (`on` / `off`).
3. **State synchronization on restart**:
   To ensure the water heater is in the correct mode if Home Assistant restarts mid-period, you can add an additional trigger on Home Assistant start (`trigger: homeassistant / event: start`) with time conditions.
