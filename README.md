# alertmanager-notify

Prometheus Alertmanager alerts as GNOME desktop notifications.

Alertmanager POSTs its webhook payload to a small daemon running in your login
session, which shells out to `notify-send`. Push rather than poll — Alertmanager
already knows how to deliver, and polling means either exposing its API or
holding credentials for it on the desktop.

- `critical` maps to `--urgency=critical`, which GNOME pins on screen until
  dismissed. `warning` is normal, everything else is low.
- A resolved alert replaces its own firing notification in place rather than
  stacking a second one beside it.
- Python standard library only. No dependencies, no virtualenv to keep alive
  across OS upgrades.

## Install

Runs as a `--user` service, not system-wide: `notify-send` needs the session
bus, and a system unit has no way to reach it.

```shell
install -Dm755 alertmanager-notify ~/.local/bin/alertmanager-notify
install -Dm644 alertmanager-notify.service ~/.config/systemd/user/alertmanager-notify.service

systemctl --user daemon-reload
systemctl --user enable --now alertmanager-notify.service
```

Open the port so Alertmanager can reach it:

```shell
sudo firewall-cmd --permanent --add-port=9099/tcp
sudo firewall-cmd --reload
```

## Configure Alertmanager

Point a webhook receiver at this host. Note that setting `config` in most Helm
charts *replaces* the packaged default rather than merging with it, so restate
anything you were relying on — commonly an inhibit rule and a route sending the
`Watchdog` dead-man's-switch alert to a null receiver.

```yaml
route:
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  # A desktop reads long silences as "resolved". Shorter than the usual 12h.
  repeat_interval: 4h
  receiver: 'gnome'
  routes:
    # Watchdog fires continuously by design. Never notify on it.
    - receiver: 'null'
      matchers:
        - alertname = "Watchdog"

receivers:
  - name: 'null'
  - name: 'gnome'
    webhook_configs:
      - url: "http://<desktop-address>:9099/"
        send_resolved: true
        # The desktop is asleep or rebooting a fair amount of the time.
        # Caps how much a burst can queue up against it.
        max_alerts: 20
```

## Verify

The daemon answers GET, so from wherever it runs:

```shell
curl http://localhost:9099/
```

To prove the whole path without waiting for something to break:

```shell
curl -X POST http://localhost:9099/ -H 'Content-Type: application/json' -d '{
  "alerts": [{
    "status": "firing",
    "labels": {"alertname": "TestAlert", "severity": "critical", "instance": "host1"},
    "annotations": {"description": "If you can read this, the path works."},
    "fingerprint": "test1"
  }]
}'
```

Send the same payload again with `"status": "resolved"` and the notification
should be replaced in place, not duplicated.

```shell
journalctl --user -u alertmanager-notify -f
```

## Configuration

All optional, set via `Environment=` in the unit.

| Variable | Default | |
|---|---|---|
| `ALERT_NOTIFY_HOST` | `0.0.0.0` | Listen address |
| `ALERT_NOTIFY_PORT` | `9099` | Listen port |
| `ALERT_NOTIFY_TOKEN` | unset | Require `Authorization: Bearer <token>` |

The port is unauthenticated by default. On a trusted LAN behind a firewall rule
that's usually fine, but anything that can reach it can post arbitrary desktop
notifications. Set `ALERT_NOTIFY_TOKEN` and add a matching `authorization`
block to the `webhook_configs` entry to close that.

## Payload

Expects the Alertmanager [webhook
format](https://prometheus.io/docs/alerting/latest/configuration/#webhook_config):
a JSON object with an `alerts` array. Each alert is read for `status`,
`fingerprint`, `labels.alertname`, `labels.severity`, and
`annotations.description` (falling back to `annotations.summary`). The label
used to identify *what* broke is the first of `node`, `app`, or `instance`.
