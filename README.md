# Wazuh Single-Node (All-in-One) Deployment Guide

A complete, from-scratch deployment guide for Wazuh — Indexer, Manager, Filebeat, and Dashboard — all installed manually on a single Ubuntu 24.04 VM using the **step-by-step installation method**. Every command, config change, and verification step below was run and confirmed working.

![Dashboard Overview](screenshots/09-dashboard-overview.png)

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Step 1 — Generate Certificates](#step-1--generate-certificates)
4. [Step 2 — Add the Wazuh Repository](#step-2--add-the-wazuh-repository)
5. [Step 3 — Install & Configure the Wazuh Indexer](#step-3--install--configure-the-wazuh-indexer)
6. [Step 4 — Verify the Indexer](#step-4--verify-the-indexer)
7. [Step 5 — Install the Wazuh Manager & Filebeat](#step-5--install-the-wazuh-manager--filebeat)
8. [Step 6 — Connect the Manager to the Indexer](#step-6--connect-the-manager-to-the-indexer)
9. [Step 7 — Start the Manager & Filebeat](#step-7--start-the-manager--filebeat)
10. [Step 8 — Install & Configure the Wazuh Dashboard](#step-8--install--configure-the-wazuh-dashboard)
11. [Step 9 — Access the Dashboard](#step-9--access-the-dashboard)
12. [Step 10 — Rotate Default Passwords](#step-10--rotate-default-passwords)
13. [Step 11 — Lock Down the Repository](#step-11--lock-down-the-repository)
14. [Screenshot Checklist](#screenshot-checklist)
15. [Lessons Learned / Gotchas](#lessons-learned--gotchas)
16. [Disclaimer](#disclaimer)

---

## Overview

Wazuh is an open-source SIEM/XDR platform for threat detection, log analysis, file integrity monitoring, and compliance. This guide deploys all core components on a single VM:

- **Wazuh Indexer** (OpenSearch-based) — stores and indexes alerts and events
- **Wazuh Manager** — collects and analyzes agent data, generates alerts
- **Filebeat** — securely forwards alerts from the manager to the indexer
- **Wazuh Dashboard** — web UI for visualizing alerts and managing the deployment

### Architecture

```
┌─────────────────────────────────────────────┐
│                Single VM                     │
│                                               │
│  ┌────────────────┐      ┌────────────────┐ │
│  │  Wazuh Manager │─────▶│  Filebeat      │ │
│  └────────────────┘      └───────┬────────┘ │
│                                   │           │
│                                   ▼           │
│                          ┌────────────────┐  │
│                          │ Wazuh Indexer  │  │
│                          └───────┬────────┘  │
│                                   │           │
│                                   ▼           │
│                          ┌────────────────┐  │
│                          │ Wazuh Dashboard│  │
│                          └────────────────┘  │
└─────────────────────────────────────────────┘
```

All components communicate over `127.0.0.1` (the loopback address), since every service lives on the same host. TLS is handled with self-signed certificates generated via `wazuh-certs-tool.sh`.

## Prerequisites

| Item | Value |
|---|---|
| OS | Ubuntu 24.04 (Noble) |
| Wazuh version | 4.14.7 |
| Access | Root or sudo |
| Recommended specs | 4 vCPU / 8 GB RAM minimum for a lab/small deployment |

---

## Step 1 — Generate Certificates

Download the certificate tool and the node-definition template:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-certs-tool.sh
curl -sO https://packages.wazuh.com/4.14/config.yml
nano ./config.yml
```

Since this is an all-in-one deployment, every node in `config.yml` — indexer, server, and dashboard — points to `127.0.0.1`:

> 📸 **Screenshot:** `config.yml` open in your editor showing all three node IPs set to `127.0.0.1`

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "127.0.0.1"
  server:
    - name: wazuh-1
      ip: "127.0.0.1"
  dashboard:
    - name: dashboard
      ip: "127.0.0.1"
```

Generate the certificates:

```bash
bash ./wazuh-certs-tool.sh -A
```

> 📸 **Screenshot:** terminal output confirming root CA, admin, indexer, Filebeat, and dashboard certs were all generated (`INFO: ... certificates created` lines)

Archive them for deployment:

```bash
tar -cvf ./wazuh-certificates.tar -C ./wazuh-certificates/ .
```

Resulting archive contains: `root-ca.pem` / `root-ca.key`, `admin.pem` / `admin-key.pem`, `node-1.pem` / `node-1-key.pem` (indexer), `wazuh-1.pem` / `wazuh-1-key.pem` (server/Filebeat), `dashboard.pem` / `dashboard-key.pem`.

---

## Step 2 — Add the Wazuh Repository

```bash
apt-get install debconf adduser procps
apt-get install gnupg apt-transport-https

curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | \
  gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | tee -a /etc/apt/sources.list.d/wazuh.list

apt-get update
```

---

## Step 3 — Install & Configure the Wazuh Indexer

```bash
apt-get -y install wazuh-indexer
```

> 📸 **Screenshot:** terminal output of the indexer package install completing successfully

Edit `/etc/wazuh-indexer/opensearch.yml`. Key single-node settings: `node.name: "node-1"`, only `node-1` listed under `cluster.initial_master_nodes`, `discovery.seed_hosts` left commented out (single-node only), and only the `node-1` entry active under `plugins.security.nodes_dn`.

> 📸 **Screenshot:** `opensearch.yml` open in your editor, showing the single-node cluster settings

```bash
nano /etc/wazuh-indexer/opensearch.yml
```

Deploy the certificates:

```bash
NODE_NAME=node-1
mkdir /etc/wazuh-indexer/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-indexer/certs/ \
  ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./admin.pem ./admin-key.pem ./root-ca.pem
mv -n /etc/wazuh-indexer/certs/$NODE_NAME.pem /etc/wazuh-indexer/certs/indexer.pem
mv -n /etc/wazuh-indexer/certs/$NODE_NAME-key.pem /etc/wazuh-indexer/certs/indexer-key.pem
chmod 500 /etc/wazuh-indexer/certs
chmod 400 /etc/wazuh-indexer/certs/*
chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/certs
```

Start the indexer:

```bash
systemctl daemon-reload
systemctl enable wazuh-indexer
systemctl start wazuh-indexer
```

Initialize the security plugin (single node — run once):

```bash
/usr/share/wazuh-indexer/bin/indexer-security-init.sh
```

> 📸 **Screenshot:** terminal output showing `Clusterstate: GREEN`, `Number of nodes: 1`, and all 10 security config types updated (`Done with success`)

Enable memory locking so the JVM doesn't swap, then set the heap size to match your VM's available RAM (rule of thumb: ~50% of total RAM, min/max equal):

```bash
mkdir -p /etc/systemd/system/wazuh-indexer.service.d/
cat > /etc/systemd/system/wazuh-indexer.service.d/wazuh-indexer.conf << EOF
[Service]
LimitMEMLOCK=infinity
EOF

nano /etc/wazuh-indexer/jvm.options
```

> 📸 **Screenshot:** `jvm.options` open in your editor showing the `-Xms`/`-Xmx` heap settings

```bash
systemctl daemon-reload
systemctl restart wazuh-indexer
```

---

## Step 4 — Verify the Indexer

```bash
curl -k -u admin https://127.0.0.1:9200
curl -k -u admin https://127.0.0.1:9200/_cat/nodes?v
curl -k -u admin:admin "https://127.0.0.1:9200/_nodes?filter_path=**.mlockall&pretty"
```

> 📸 **Screenshot:** terminal output showing the cluster info JSON, the `_cat/nodes` table with a single active node, and `"mlockall": true`

---

## Step 5 — Install the Wazuh Manager & Filebeat

```bash
apt-get -y install wazuh-manager
apt-get -y install filebeat
curl -so /etc/filebeat/filebeat.yml https://packages.wazuh.com/4.14/tpl/wazuh/filebeat/filebeat.yml
```

Filebeat's `output.elasticsearch.hosts` stays at `127.0.0.1:9200` by default — no change needed for all-in-one:

> 📸 **Screenshot:** `filebeat.yml` open in your editor, showing `hosts: ["127.0.0.1:9200"]` and the cert paths

```bash
nano /etc/filebeat/filebeat.yml
```

Store indexer credentials in Filebeat's keystore instead of hardcoding them:

```bash
filebeat keystore create
echo admin | filebeat keystore add username --stdin --force
echo admin | filebeat keystore add password --stdin --force
```

Install the Wazuh alert template and module:

```bash
curl -so /etc/filebeat/wazuh-template.json \
  https://raw.githubusercontent.com/wazuh/wazuh/v4.14.7/extensions/elasticsearch/7.x/wazuh-template.json
chmod go+r /etc/filebeat/wazuh-template.json

curl -s https://packages.wazuh.com/4.x/filebeat/wazuh-filebeat-0.5.tar.gz | \
  tar -xvz -C /usr/share/filebeat/module
```

Deploy Filebeat's certs — note these are generated under the **server** node name (`wazuh-1`), then renamed to the generic `filebeat.pem`:

```bash
NODE_NAME=wazuh-1
mkdir /etc/filebeat/certs
tar -xf ./wazuh-certificates.tar -C /etc/filebeat/certs/ \
  ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
mv -n /etc/filebeat/certs/$NODE_NAME.pem /etc/filebeat/certs/filebeat.pem
mv -n /etc/filebeat/certs/$NODE_NAME-key.pem /etc/filebeat/certs/filebeat-key.pem
chmod 500 /etc/filebeat/certs
chmod 400 /etc/filebeat/certs/*
chown -R root:root /etc/filebeat/certs
```

---

## Step 6 — Connect the Manager to the Indexer

Store indexer credentials in the manager's keystore (used by the vulnerability detection module):

```bash
echo 'admin' | /var/ossec/bin/wazuh-keystore -f indexer -k username
echo 'admin' | /var/ossec/bin/wazuh-keystore -f indexer -k password
```

Edit `ossec.conf`'s `<indexer>` block to point at the indexer over HTTPS with the Filebeat-generated certs. **Use `127.0.0.1:9200` here — not `0.0.0.0`**, which is only valid as a bind/listen address, not a connection target:

```bash
nano /var/ossec/etc/ossec.conf
```

> 📸 **Screenshot:** `ossec.conf`'s `<indexer>` block, confirming the host is `https://127.0.0.1:9200`

```xml
<indexer>
  <enabled>yes</enabled>
  <hosts>
    <host>https://127.0.0.1:9200</host>
  </hosts>
  <ssl>
    <certificate_authorities>
      <ca>/etc/filebeat/certs/root-ca.pem</ca>
    </certificate_authorities>
    <certificate>/etc/filebeat/certs/filebeat.pem</certificate>
    <key>/etc/filebeat/certs/filebeat-key.pem</key>
  </ssl>
</indexer>
```

---

## Step 7 — Start the Manager & Filebeat

```bash
systemctl daemon-reload
systemctl enable wazuh-manager
systemctl start wazuh-manager
systemctl status wazuh-manager   # confirm "active (running)"
```

> 📸 **Screenshot:** `systemctl status wazuh-manager` output showing "active (running)" and all sub-processes started

```bash
systemctl daemon-reload
systemctl enable filebeat
systemctl start filebeat
filebeat test output
```

> 📸 **Screenshot:** `filebeat test output` result — confirms TLS handshake OK (TLSv1.3) and "talk to server... OK"

---

## Step 8 — Install & Configure the Wazuh Dashboard

```bash
apt-get install debhelper tar curl libcap2-bin
apt-get -y install wazuh-dashboard
```

> 📸 **Screenshot:** terminal output of the dashboard package install completing successfully

Edit `/etc/wazuh-dashboard/opensearch_dashboards.yml`. Points `opensearch.hosts` at the local indexer (`localhost:9200`), binds `server.host` to `0.0.0.0` so the *dashboard UI* is reachable from other machines, and references the dashboard's own TLS certs:

```bash
nano /etc/wazuh-dashboard/opensearch_dashboards.yml
```

> 📸 **Screenshot:** `opensearch_dashboards.yml` open in your editor

Deploy dashboard certs and start the service:

```bash
NODE_NAME=dashboard
mkdir /etc/wazuh-dashboard/certs
tar -xf ./wazuh-certificates.tar -C /etc/wazuh-dashboard/certs/ \
  ./$NODE_NAME.pem ./$NODE_NAME-key.pem ./root-ca.pem
[ ! -e /etc/wazuh-dashboard/certs/dashboard.pem ] && \
  mv -n /etc/wazuh-dashboard/certs/$NODE_NAME.pem /etc/wazuh-dashboard/certs/dashboard.pem
[ ! -e /etc/wazuh-dashboard/certs/dashboard-key.pem ] && \
  mv -n /etc/wazuh-dashboard/certs/$NODE_NAME-key.pem /etc/wazuh-dashboard/certs/dashboard-key.pem
chmod 500 /etc/wazuh-dashboard/certs
chmod 400 /etc/wazuh-dashboard/certs/*
chown -R wazuh-dashboard:wazuh-dashboard /etc/wazuh-dashboard/certs

systemctl daemon-reload
systemctl enable wazuh-dashboard
systemctl start wazuh-dashboard
```

Finally, confirm the dashboard's API connection settings, which link it to the Wazuh Manager's API (default `wazuh-wui` user, port `55000`):

```bash
nano /usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
```

> 📸 **Screenshot:** `wazuh.yml` showing the `hosts.default` block with `url: https://localhost`, `port: 55000`, and the `wazuh-wui` username

---

## Step 9 — Access the Dashboard

Navigate to `https://<YOUR_VM_IP>` from a browser and log in with the `admin` indexer credentials.

> 📸 **Screenshot:** the Wazuh Dashboard Overview page in the browser — showing the Agents Summary, Last 24 Hours Alerts panel, and Endpoint Security / Threat Intelligence module tiles

Confirmed working state: 0 agents registered (expected pre-agent-deployment), rule-severity breakdown populated, and all modules (Configuration Assessment, Malware Detection, Threat Hunting, Vulnerability Detection, File Integrity Monitoring, MITRE ATT&CK) visible and clickable.

---

## Step 10 — Rotate Default Passwords

```bash
/usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
  --api --change-all --admin-user wazuh --admin-password wazuh
```

> 📸 **Screenshot:** terminal output listing each rotated password (blur/crop the actual password values before publishing — see [Screenshot Checklist](#screenshot-checklist))

This generates new strong passwords for all internal indexer users (`admin`, `kibanaserver`, `readall`, etc.) and the Wazuh API users (`wazuh`, `wazuh-wui`). It automatically updates Filebeat's keystore and the dashboard's `wazuh-wui` user — **but not** the manager's own `wazuh-keystore` indexer credentials. Re-run Step 6's `wazuh-keystore` commands with the new password and restart the manager afterward (see [Lessons Learned](#lessons-learned--gotchas)).

---

## Step 11 — Lock Down the Repository

Disable the Wazuh package repo to prevent accidental version-breaking upgrades:

```bash
sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
apt update
```

---

## Screenshot Checklist

Save screenshots into a `screenshots/` folder in the repo root, named to match the embeds used throughout this guide:

| Filename | Content |
|---|---|
| `screenshots/01-config-yml.png` | `config.yml` — all nodes set to `127.0.0.1` |
| `screenshots/02-certs-generated.png` | Terminal output of `wazuh-certs-tool.sh -A` |
| `screenshots/03-indexer-install.png` | Terminal output of `wazuh-indexer` package install |
| `screenshots/04-opensearch-yml.png` | `opensearch.yml` single-node settings |
| `screenshots/05-security-init.png` | `indexer-security-init.sh` success output |
| `screenshots/06-jvm-options.png` | `jvm.options` heap size settings |
| `screenshots/07-indexer-verify.png` | `curl` cluster health / `_cat/nodes` output |
| `screenshots/08-filebeat-yml.png` | `filebeat.yml` config |
| `screenshots/09-ossec-conf.png` | `ossec.conf` `<indexer>` block |
| `screenshots/10-manager-status.png` | `systemctl status wazuh-manager` |
| `screenshots/11-filebeat-test.png` | `filebeat test output` result |
| `screenshots/12-dashboard-install.png` | Terminal output of `wazuh-dashboard` package install |
| `screenshots/13-dashboards-yml.png` | `opensearch_dashboards.yml` config |
| `screenshots/14-wazuh-yml.png` | `wazuh.yml` API connection block |
| `screenshots/09-dashboard-overview.png` | Dashboard Overview page in the browser (used at top of README) |
| `screenshots/15-password-rotation.png` | `wazuh-passwords-tool.sh` output *(redact real passwords first)* |

> ⚠️ **Before committing any screenshot:** blur or crop out real IPs and passwords, especially the terminal screenshot from Step 10 — it displays actual generated secrets in plaintext.

---

## Lessons Learned / Gotchas

- **`ossec.conf`'s `<indexer><hosts>` must be a real, connectable address — not `0.0.0.0`.** `0.0.0.0` is a bind/listen address used in `opensearch.yml`'s `network.host`; it is not something a client can connect *to*. Use `127.0.0.1` (or your host's real IP) here, matching the rest of the all-in-one config.
- **`wazuh-passwords-tool.sh --change-all` doesn't update every consumer of the indexer credentials.** It automatically updates Filebeat's keystore and the dashboard's `wazuh-wui` user, but **not** the Wazuh manager's own `wazuh-keystore` entry for indexer connectivity (used by the vulnerability detection module). Update it manually with `wazuh-keystore -f indexer -k username/password`, then restart the manager.
- For an all-in-one setup, every host/IP field across `config.yml`, `opensearch.yml`, `filebeat.yml`, `ossec.conf`, and `opensearch_dashboards.yml` should point to the same address (`127.0.0.1` is the officially recommended default for all-in-one) — no cluster-specific settings (`discovery.seed_hosts`, multiple `nodes_dn` entries, worker node types) are needed.
- Cert file naming from `wazuh-certs-tool.sh` follows the node names in `config.yml` (`node-1.pem`, `wazuh-1.pem`, `dashboard.pem`) and must be renamed into each component's expected filename (`indexer.pem`, `filebeat.pem`, `dashboard.pem`).
- `filebeat test output` and the indexer's `_cat/nodes?v` endpoint are the fastest sanity checks to confirm TLS and cluster health before moving to the next component.
- JVM heap size in `jvm.options` should be set explicitly (min = max) rather than left at the small defaults, sized to roughly half the VM's available RAM.

## Disclaimer

This is a lab/learning deployment using self-signed certificates and is not hardened for production use. IP addresses and credentials shown in this guide are either loopback (`127.0.0.1`) or placeholders — actual lab credentials were rotated and are not published.

## References

- [Wazuh Official Documentation](https://documentation.wazuh.com/current/)
- [Wazuh Quickstart Guide](https://documentation.wazuh.com/current/quickstart.html)
