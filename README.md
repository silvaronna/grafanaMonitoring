# 🔥 The All-Seeing Eye: A Badass Monitoring Stack

Yo! Welcome to the command center. This repo packs everything you need to deploy a rock-solid monitoring pipeline using the legendary trio: **Prometheus, Grafana, and Telegraf**, all neatly containerized with Docker.

The mission? To get sweet, sweet metrics from your servers (Linux OS-level stuff) and your VMware ESXi infrastructure, and then beam 'em up into some slick Grafana dashboards. No more flying blind.

---

## 🏗️ The Architecture: How The Magic Happens

This setup is split into two killer components:

1.  **`monitorServer`**: Your main rig. This is where the brain lives. It runs:
    * **Prometheus**: The data hoarder. Gathers and stores all your time-series metrics.
    * **Grafana**: The artist. Takes that raw data and paints beautiful, interactive dashboards.
    * **Telegraf**: The specialist agent. Talks to your VMware ESXi hosts and translates their metrics for Prometheus.
2.  **`targetServer`**: Any server you want to keep an eye on. This thing runs one simple job:
    * **Node Exporter**: A lightweight informant that exposes all the juicy OS-level metrics (CPU, RAM, Disk, etc.).

**Data Flow Breakdown:**

```
[Your Target Server] ----(Node Exporter)----> [Prometheus] ----> [Grafana]
[Your VMware ESXi]   ----(Telegraf)----------> [Prometheus] ----> [Grafana]
```

---

## ✅ The Checklist: What You Need

Before we dive in, make sure you've got the essentials:

* **On Both Servers (Monitor & Target):**
    * [ ] Root or `sudo` access. You're the captain now.
    * [ ] `git` installed to clone this masterpiece.
    * [ ] `docker` is up and running.
    * [ ] `docker compose` is ready to roll.
* **For VMware Monitoring:**
    * [ ] Admin access to your VMware ESXi host to create a new, limited user.

---

## ⚙️ The Setup: Let's Get This Party Started

Before we dive in to the setup, please make sure you have already docker images contain:
- prom/node-exporter:v1.8.1
- prom/prometheus:v2.53.1
- grafana/grafana-oss:11.1.0
- telegraf:1.30

We'll tackle this in two parts. First, the targets, then the main monitor server.

### Part 1: Rigging the Target Servers

Do this on **every Linux server** you want OS-level metrics from.

1.  **Clone This Repo**
    ```bash
    git clone https://github.com/silvaronna/grafanaMonitoring
    cd grafanaMonitoring/targetServer
    ```

2.  **Fire Up Node Exporter**
    This one-liner pulls the `prom/node-exporter` image and gets it running in the background. Easy peasy.
    ```bash
    docker-compose up -d
    ```

3.  **(CRITICAL) Punch a Hole in the Firewall**
    You gotta let your Monitor Server talk to this exporter. Lock it down so **only** your monitor's IP can get through on port `9100`.

    * **For RHEL / Rocky / AlmaLinux (using `firewalld`):**
        ```bash
        # Swap <MONITOR_SERVER_IP> with the real deal
        sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<MONITOR_SERVER_IP>" port protocol="tcp" port="9100" accept'
        sudo firewall-cmd --reload
        ```
    * **For Ubuntu / Debian (using `ufw`):**
        ```bash
        # Swap <MONITOR_SERVER_IP> with the real deal
        sudo ufw allow from <MONITOR_SERVER_IP> to any port 9100
        ```

### Part 2: Assembling the Monitor Server

This is the main event. Let's build your command center.

#### A. Initial Prep (One-Time Setup)

1.  **Clone This Repo**
    ```bash
    git clone https://github.com/silvaronna/grafanaMonitoring
    cd grafanaMonitoring/monitorServer
    ```

2.  **Create Persistent Data Dirs**
    This is a pro-move to make sure your dashboards and metrics survive a restart.
    ```bash
    mkdir -p data/prometheus
    mkdir -p data/grafana
    ```

3.  **Set Grafana Volume Permissions**
    Grafana's container is a good guy and doesn't run as root. This command gives it the keys to its own data directory.
    ```bash
    sudo chown -R 472:472 ./data/grafana
    ```

4.  **(LIFESAVER) Docker Firewall Fix for RHEL/Rocky/AlmaLinux**
    By default, Docker containers on these systems can feel a bit trapped. This command lets them talk to the outside world (like your ESXi host).
    ```bash
    sudo firewall-cmd --zone=public --add-masquerade --permanent
    sudo firewall-cmd --reload
    ```

#### B. Tweak Your Configs

1.  **Aim Prometheus (`prometheus/prometheus.yml`)**
    Open this file to tell Prometheus where to find your Node Exporters.
    ```bash
    nano prometheus/prometheus.yml
    ```
    Under `scrape_configs`, add new targets.
    ```yaml
    # ...
    # Example for a new Node Exporter target
    - job_name: 'your-cool-server-name'
      static_configs:
        - targets: ['<YOUR_TARGET_SERVER_IP>:9100']
    
    # The telegraf target is already set, just replace your credentials vmware vsphere.
    - job_name: 'telegraf-vsphere'
    # ...
    ```

2.  **Prep Telegraf (`telegraf/telegraf.conf`)**
    Only if you're monitoring VMware.
    * **First, create a read-only user on your ESXi host.** Don't use root! [Follow a guide like this](https://docs.vmware.com/en/VMware-vSphere/6.5/com.vmware.vsphere.security.doc/GUID-53D26342-322A-45A5-A473-65DA20422538.html) to create a user with a `Read-only` role.
    * **Then, edit the config file:**
        ```bash
        nano telegraf/telegraf.conf
        ```
        Plug in your ESXi host IP and the read-only credentials.
        ```conf
        # ...
        [[inputs.vsphere]]
          vcenters = [ "https://<YOUR_ESXI_IP>/sdk" ]
          username = "<YOUR_ESXI_READONLY_USER>"
          password = "<THAT_USER_PASSWORD>"
          insecure_skip_verify = true
        ```

#### C. Launch Time! 🚀

1.  **Engage!**
    From the `monitorServer` directory, let it rip.
    ```bash
    docker-compose up -d
    ```
    This command pulls all the necessary images and spins up your entire monitoring stack.

---

## 🚀 Post-Flight Checklist

Your stack is live! Time to see the goods.

1.  **Configure Grafana's Data Source**
    * Head over to `http://<YOUR_MONITOR_SERVER_IP>:3000` in your browser.
    * Log in with `admin` / `admin`. Change the password when it asks.
    * Navigate to **Configuration (⚙️) → Data Sources**.
    * Click **Add data source** → **Prometheus**.
    * For the **URL**, use the internal service name: `http://prometheus:9090`.
    * Hit **Save & test**. You should see a green "Data source is working" message.

2.  **Import Dashboards from the JSON Files**
    Since your server is offline, we'll upload the dashboards manually.
    * Navigate to **Dashboards (⊞) → New → Import**.
    * Click the **Upload file** button.
    * Browse to the `JSON/` folder in this repo.
    * Pick a file, like `node-exporter-full.json`.
    * On the next page, make sure to select your **Prometheus** data source from the dropdown.
    * Click **Import**.

Rinse and repeat for any other dashboard JSON files you have.

**And... you're done! Welcome to a world of beautiful metrics. Go grab a coffee, you've earned it.**

