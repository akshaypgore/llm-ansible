**Audience:** Intermediate DevOps | **Series:** Part 2 of 4

## Quick Recap from Part 1

- Set up Ubuntu Server VM (phi) on VirtualBox
- Installed and configured Ollama as a systemd service
- Automated entire setup with Ansible (llm-ansible repo)
- Interacted with phi3:mini via CLI, curl
- [Link to Part 1](https://dev.to/akshaygore/self-hosted-ai-on-linux-a-devops-home-lab-guide-28kc)

---
### Why custom monitoring setup

- Ollama **does not have a native Prometheus exporter** (a /metrics endpoint) primarily because it is designed as a lightweight, user-friendly tool for running local LLMs, focusing on simplicity and ease of setup for local developers rather than complex enterprise monitoring


## What This Post Covers
- Writing a custom Prometheus exporter in Python
- Installing Prometheus and Grafana with Ansible
- Building a monitoring dashboard for your LLM

---

##### Github Link
[Repository Link](https://github.com/akshaypgore/llm-ansible)

---
## Section 1 — The Problem

### 1.1 Ollama Has No Native Metrics

Most production services expose a `/metrics` endpoint in Prometheus format out of the box. Ollama does not.

```bash
curl http://192.168.1.52:11434/metrics
# 404 page not found
```

This is a common situation in DevOps — a service you depend on doesn't expose metrics. The solution is an **exporter**.

### 1.2 What is an Exporter

```
Service (Ollama)
      ↓
Exporter (queries Ollama API)
      ↓
Exposes /metrics in Prometheus format
      ↓
Prometheus scrapes exporter
      ↓
Grafana visualizes
```

This pattern is used across the ecosystem:
- MySQL exporter
- Redis exporter
- Node exporter

Same pattern, different service.


---


## Section 2 — Architecture

```
phi VM
──────────────────────
Ollama            →  port 11434  (LLM serving)
ollama-exporter   →  port 8000   (custom metrics)
node-exporter     →  port 9100   (system metrics)

monitoring VM
────────────────────────────
Prometheus        →  port 9090   (scrapes phi)
Grafana           →  port 3000   (visualizes)
```

**Why separate VMs:**

```
→  monitoring runs independently
→  if phi goes down monitoring still works
→  monitoring doesn't consume phi resources
→  mirrors production architecture
```

---


## Section 3 — Custom Ollama Exporter

### 3.1 What Metrics We Can Get

Ollama exposes data via REST API endpoints we explored in Part 1:

```
/api/ps    →  running models, RAM usage, context length
/api/tags  →  downloaded models, disk usage
/          →  health check
```

### 3.2 Metrics We Expose

```
ollama_up                  →  is Ollama API responding (0 or 1)
ollama_models_loaded       →  models currently in RAM
ollama_model_ram_bytes     →  RAM consumed per model
ollama_model_context_length → context window size
ollama_models_available    →  models downloaded on disk
ollama_model_disk_bytes    →  disk space per model
ollama_total_disk_bytes    →  total disk used by all models
```

### 3.3 How the Exporter Works

```python
# Simple structure
→  HTTP server on port 8000
→  on GET /metrics:
   query Ollama /api/ps
   query Ollama /api/tags
   format as Prometheus metrics
   return response
→  Prometheus scrapes every 15 seconds
```
[Python Exporter File](https://github.com/akshaypgore/llm-ansible/blob/master/roles/ollama/templates/ollama_exporter.py.j2)
---
### 3.4 Prometheus Metrics Format

```
# HELP ollama_up Whether Ollama API is responding
# TYPE ollama_up gauge
ollama_up 1

# HELP ollama_model_ram_bytes RAM consumed by each loaded model
# TYPE ollama_model_ram_bytes gauge
ollama_model_ram_bytes{model="phi3:mini"} 3730644480
```

Key things to notice:
- `# HELP` — human readable description
- `# TYPE` — metric type (gauge, counter, histogram)
- `labels` in `{}` — metadata attached to metric
- value at the end


![Exposing metrics at port 8000](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/a2dqlurlu6bto7g66te7.png)

### 3.5 Running as Systemd Service

```
ollama-exporter.service
────────────────────────
→  starts after ollama.service
→  restarts automatically on failure
→  runs as ollama user
→  logs to journalctl
```

![Status of ollama exporter service](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3za43fn2t2iobm3slht1.png)

---


## Section 4 — Automating with Ansible

Everything above is automated in the llm-ansible repo.

### 4.1 Updated Repo Structure


![Repo Structure](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n0rhjtmx8m80j15jzchy.png)

### 4.2 Updated Inventory

```ini
[llm_servers]
phi ansible_host=llm_server_ip ansible_user=your_username

[monitoring_servers]
monitoring ansible_host=monitoring_server_ip ansible_user=your_username
```

### 4.3 Updated Playbook

```yaml
---
- name: Deploy Ollama LLM Infrastructure
  hosts: llm_servers
  become: yes
  roles:
    - ollama

- name: Deploy Monitoring Infrastructure
  hosts: monitoring_servers
  become: yes
  roles:
    - monitoring
```

### 4.4 Key Variables

```yaml
# Prometheus
prometheus_port: 9090
prometheus_scrape_interval: "15s"
prometheus_retention_time: "15d"

# Scrape targets
ollama_exporter_host: "192.168.1.52"
ollama_exporter_port: 8000
phi_node_exporter_port: 9100

# Grafana
grafana_port: 3000
grafana_admin_user: "admin"
grafana_admin_password: "admin"
```

### 4.5 Running the Playbook

```bash
ansible-playbook -i inventory.ini playbook.yaml
```

![Image of ansible run](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ubmoqvct6js7hyrejmvc.png)

![Image of ansible run](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3tu14xq5phw4zcxmf6eh.png)


![Image of ansible run](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/47p8eb8w0r5h4lboon51.png)
---

## Section 5 — Verifying Prometheus Targets

```bash
curl http://localhost:9090/api/v1/targets | python3 -m json.tool
```

All three targets should show `"health": "up"`:

```
job: prometheus  →  localhost:9090   health: up
job: ollama      →  phi:8000         health: up
job: node        →  phi:9100         health: up
```

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/jc0cd60oy1pgc445dt0t.png)

---

## Section 6 — Grafana Dashboard

### 6.1 Add Prometheus Data Source

```
Connections → Data sources → Add data source
→  Select Prometheus
→  URL: http://localhost:9090
→  Save & Test
→  "Successfully queried the Prometheus API"
```

### 6.2 Dashboard Panels

**Row 1 — Ollama Health:**

| Panel | Query | Type |
|-------|-------|------|
| Ollama Status | `ollama_up` | Stat |
| Model Memory Usage | `ollama_model_ram_bytes` | Stat |
| Models in Memory | `ollama_models_loaded` | Stat |

**Row 2 — System Health (phi VM):**

| Panel | Query | Type |
|-------|-------|------|
| CPU Usage % | `100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` | Stat |
| Memory Usage % | `100 - ((node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100)` | Stat |
| Disk Usage % | `100 - ((node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100)` | Stat |

> 📸 Screenshot: complete Grafana dashboard

![Grafana dashboard panels](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4u4p804vbrr6ss9rp0da.png)


![Grafana dashboard panels](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vzpla6pjwr6247z77d7k.png)




### 6.3 What the Dashboard Tells You

```
Ollama Status       →  is LLM serving healthy?
Model Memory Usage  →  3.7GB when phi3:mini loaded
                        0 when model unloaded (keep_alive timeout)
Models in Memory    →  1 when active, 0 when idle
CPU Usage %         →  spikes during inference
                        baseline low when idle
Memory Usage %      →  stable, dominated by model RAM
Disk Usage %        →  increases as you pull more models
```
#### Demo of panels

> When no model is running
```
root@phi:/home/akshaygore# ollama ps
NAME    ID    SIZE    PROCESSOR    CONTEXT    UNTIL
root@phi:/home/akshaygore#
```
Below Dashboard shows stats accordingly

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/sfftuw5tyzqxmpjbwn1m.png)

> Once we load the phi model
```
root@phi:/home/akshaygore# ollama run phi3:mini
>>> hi
Hi there! How can I help you today?

>>> /bye
root@phi:/home/akshaygore# ollama ps
NAME         ID              SIZE      PROCESSOR    CONTEXT    UNTIL
phi3:mini    4f2222927938    3.7 GB    100% CPU     4096       4 minutes from now
root@phi:/home/akshaygore#
```
Dashboards updating the stats once we run the model
![Grafana Dashboard showing stats of model](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lcde7hhx7a1v3x8154p8.png)
