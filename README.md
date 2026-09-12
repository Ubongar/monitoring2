# Monitoring Stack: Prometheus + Grafana + Ollama (GPU)

Repo: https://github.com/Ubongar/monitoring2

This stack runs Prometheus, Grafana, node-exporter, cAdvisor, an NVIDIA GPU exporter, and Ollama, all networked together in Docker. It monitors host metrics, per-container metrics, and GPU metrics while Ollama runs LLM inference.

## Requirement: NVIDIA GPU is compulsory

This setup will not work on a machine without an NVIDIA GPU. The `ollama` and `nvidia-gpu-exporter` services both request GPU access via `deploy.resources.reservations.devices`, and Docker will refuse to start those containers if no GPU is available or if GPU passthrough is not configured. If your machine has no NVIDIA GPU, stop here, this stack is not for you as written.

## Prerequisites

- A PC with an NVIDIA GPU (tested on a GTX 1650, 4GB VRAM)
- Windows 11 with Docker Desktop installed, using the WSL 2 backend
- Latest NVIDIA driver installed from nvidia.com (not just GeForce Experience)

Note on VRAM: 4GB is tight. Only small quantized models (1B to 3B parameters, e.g. `llama3.2:1b`, `phi3:mini`) will fully offload to GPU. Larger models will spill into CPU/RAM and run slower.

## Step 1: Update the NVIDIA driver

1. Go to https://www.nvidia.com/Download/index.aspx
2. Select your exact GPU model and Windows 11
3. Download and install the latest Game Ready or Studio driver
4. Choose Clean Install if offered, then reboot

Recent NVIDIA drivers include WSL2 GPU passthrough support, which Docker Desktop relies on. No separate CUDA toolkit install is needed on the Windows host itself.

## Step 2: Enable GPU support in Docker Desktop

1. Open Docker Desktop, go to Settings > General
2. Confirm "Use the WSL 2 based engine" is checked
3. Go to Settings > Resources > WSL Integration
4. Confirm your default WSL distro is toggled on
5. Click Apply & Restart

GPU passthrough works automatically through WSL2 once the driver above is installed, no extra toolkit is needed on Windows.

## Step 3: Verify the GPU is visible to containers

Run:

```bash
docker run --rm --gpus all nvidia/cuda:12.4.0-base-ubuntu22.04 nvidia-smi
```

Expected output includes a table showing your GPU model and driver version. If you get an error like `could not select device driver "" with capabilities: [[gpu]]`, stop here and fix Steps 1 and 2 before continuing. Nothing past this point will work without GPU passthrough confirmed.

## Step 4: docker-compose.yml

Full file, already in this repo:

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    networks:
      - monitoring

  nvidia-gpu-exporter:
    image: utkuozdemir/nvidia_gpu_exporter:1.2.1
    container_name: nvidia-gpu-exporter
    restart: unless-stopped
    ports:
      - "9835:9835"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      - monitoring

  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    restart: unless-stopped
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      - monitoring

volumes:
  prometheus-data:
  grafana-data:
  ollama-data:

networks:
  monitoring:
    driver: bridge
```

Note: cAdvisor here omits the `/dev/kmsg` device mount used in the standard Linux recipe. Docker Desktop on Windows does not expose that device the same way, and mounting it can prevent the container from starting. Container CPU, memory, and network metrics still work without it.

## Step 5: prometheus.yml

Full file, already in this repo:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'nvidia-gpu'
    static_configs:
      - targets: ['nvidia-gpu-exporter:9835']
```

## Step 6: Clone and start the stack

```bash
git clone https://github.com/Ubongar/monitoring2.git
cd monitoring2
docker compose up -d --build
```

Confirm everything is running:

```bash
docker compose ps
```

You should see six containers Up: `prometheus`, `node-exporter`, `grafana`, `cadvisor`, `nvidia-gpu-exporter`, and `ollama`.

## Step 7: Pull a small model and generate load

```bash
docker exec -it ollama ollama pull llama3.2:1b
docker exec -it ollama ollama run llama3.2:1b "explain prometheus in one sentence"
```

Run it a few times in a row to get sustained load visible on the dashboards, rather than a single blip:

```bash
for i in 1 2 3 4 5; do docker exec ollama ollama run llama3.2:1b "write a haiku about docker"; done
```

## Step 8: Confirm metrics and view dashboards

Check that Prometheus is scraping every target:

```bash
curl http://localhost:9090/api/v1/targets
```

Or open `http://localhost:9090/targets` in a browser. Both `cadvisor` and `nvidia-gpu` should show `health: up`, alongside `prometheus` and `node-exporter`.

Spot-check the raw GPU metric directly:

```bash
curl http://localhost:9835/metrics | grep nvidia_smi_utilization_gpu_ratio
```

If that returns a number, the exporter is working.

### Access points

- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (login `admin` / `admin`, you will be prompted to change it)
- cAdvisor raw metrics: http://localhost:8080
- GPU exporter raw metrics: http://localhost:9835/metrics
- Ollama API: http://localhost:11434

### Grafana setup

1. Connections > Data sources > Add data source > Prometheus
2. URL: `http://prometheus:9090` (use the service name, not `localhost`, since Grafana is calling from inside its own container)
3. Save and test
4. Import dashboards for cAdvisor and nvidia_gpu_exporter by searching grafana.com/dashboards for the current dashboard IDs, or build panels manually against:
   - `nvidia_smi_utilization_gpu_ratio`
   - `nvidia_smi_memory_used_bytes`
   - `container_cpu_usage_seconds_total{name="ollama"}`

## Known limitation

Ollama does not expose a native Prometheus `/metrics` endpoint with inference-level stats such as tokens per second or queue depth. What this stack measures is resource consumption while Ollama runs (GPU utilization, VRAM, container CPU and memory), not inference-level performance metrics. A custom wrapper around Ollama's API would be needed for that.
