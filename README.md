# 🛰️ Sentinel

A lightweight, self-hosted observability stack for monitoring system **metrics**, **logs**, and **services** — powered by **Grafana**, **Prometheus**, **Loki**, and **Promtail**.

---

## 🚀 Stack Overview

| Component     | Purpose                              | Port  |
|----------------|---------------------------------------|-------|
| **Prometheus** | Collects system and app metrics        | `9090` |
| **Grafana**    | Visualization and dashboards          | `3200` |
| **Loki**       | Stores and queries logs               | `3100` |
| **Promtail**   | Ships logs from Docker & system       | `9080` |

---

## ⚙️ Folder Structure

```

sentinel/
├── loki/
│   └── loki-config.yaml
├── promtail/
│   ├── promtail-config.yml
│   └── docker-compose.yml
├── prometheus/
│   ├── config/
│   │   └── prometheus.yml
│   ├── data/
│   └── docker-compose.yml
└── grafana/ (optional)

````

---

## 🧩 Setup & Run

### 1️⃣ Clone the Repository
```bash
git clone https://github.com/<your-username>/sentinel.git
cd sentinel
````

### 2️⃣ Start Each Service

Start them one by one:

```bash
# Prometheus
cd prometheus && docker compose up -d

# Loki
cd ../loki && docker compose up -d

# Promtail
cd ../promtail && docker compose up -d
```

*(Grafana can be deployed separately or on the same VM.)*

---

## 🔍 Verify Setup

* **Prometheus:** [http://localhost:9090](http://localhost:9090)
* **Grafana:** [http://localhost:3200](http://localhost:3200)
* **Loki:** [http://localhost:3100/metrics](http://localhost:3100/metrics)
* **Promtail:** [http://localhost:9080/service-discovery](http://localhost:9080/service-discovery)

---

## 🧠 Common Issues & Fixes

### 🪣 1. Loki Restart Loop

**Error:**
`mkdir /tmp/loki/rules: permission denied`

**Fix:**
Grant Loki a writable directory:

```bash
sudo mkdir -p /tmp/loki && sudo chmod -R 777 /tmp/loki
docker restart loki
```

---

### 🧾 2. Promtail “Timestamp too old”

**Error:**
`server returned HTTP 400 Bad Request: entry has timestamp too old`

**Fix:**
Promtail is reading very old container logs. Reset its position file:

```bash
sudo rm -f /tmp/positions.yaml
docker restart promtail
```

---

### 🧱 3. Missing `container_name` or `service_name`

**Symptoms:** No container label appears in Grafana logs.

**Fix:**

* Ensure Promtail mounts Docker socket:

  ```yaml
  - /var/run/docker.sock:/var/run/docker.sock
  ```
* Then restart Promtail:

  ```bash
  docker restart promtail
  ```

---

### 🌐 4. Grafana Not Showing Logs

**Fix:**

* Go to Grafana → **Settings → Data Sources → Loki**.
* Set URL:
  `http://loki:3100` (if on same network)
  or
  `https://loki.yourdomain.com`
* Try query:

  ```
  {container_name="docmatrix"} |= ""
  ```

---

### ⚡ 5. Clean Restart (after config changes)

```bash
docker compose down
sudo rm -f /tmp/positions.yaml
docker compose up -d
```

---

## 🧩 Add New Containers for Log Collection

Promtail is already configured to **auto-discover Docker containers** through the Docker socket:

```yaml
- job_name: docker
  docker_sd_configs:
    - host: unix:///var/run/docker.sock
      refresh_interval: 10s
  relabel_configs:
    - source_labels: ['__meta_docker_container_name']
      regex: '/(.*)'
      target_label: 'container_name'
```

So whenever you start a **new container**, Promtail will automatically detect and start collecting its logs.

✅ **No config change needed.**

If you want to filter logs from a specific container or app in Grafana:

* **By container name:**

  ```
  {container_name="docmatrix"} |= ""
  ```

* **By service or app label (if added):**

  ```
  {app="docmatrix", env="production"} |= ""
  ```

* **By stream (stdout/stderr):**

  ```
  {container_name="docmatrix", stream="stderr"}
  ```

If your new container doesn’t appear in logs:

```bash
docker exec -it promtail curl -s http://localhost:9080/service-discovery | jq '.docker'
```

You should see it listed there. If not:

* Check that the container’s logs exist in `/var/lib/docker/containers/`
* Restart Promtail

---

## 🧭 Notes

* Promtail labels each log line with:

  * `container_name`
  * `service_name`
  * `stream` (`stdout` / `stderr`)
  * `app` and `env` (if defined)
* This repo is self-contained — you can copy it to any VM and redeploy easily.
* Keep your configs versioned; no secrets are stored here.

---

## 💬 Author

Maintained by **Abhinav Aditya** — *DevOps and Fullstack Engineer*.