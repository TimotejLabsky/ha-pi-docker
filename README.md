# Homeassistant with Nginx Reverse Proxy

## Homeassistant

Chnages needed:
- Add `http` to `configuration.yaml`:
```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 1287.0.0.1
    - ::1
```

## Run on host startup 

- create a systemd service file: `/etc/systemd/system/homea-ssistant.service` [home-assistant.service](home-assistant.service)

```bash
sudo systemctl enable home-assistant
sudo systemctl start home-assistant
```
