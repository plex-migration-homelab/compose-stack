# Immich Machine Learning Stack (Fileserver)

This stack runs the Immich machine learning container with NVIDIA GPU acceleration on the fileserver. It provides ML services to the main Immich stack running on the mini PC.

**Deployment Method:** This is designed to be deployed via Docker's web UI (Portainer, Yacht, etc.). See **[DOCKER-UI-DEPLOYMENT.md](DOCKER-UI-DEPLOYMENT.md)** for complete setup instructions.

The `compose.yml` file is provided as a reference but is not the primary deployment method.

## What's Included

- **Immich Machine Learning** - GPU-accelerated ML processing
  - Facial recognition
  - Object detection
  - Image classification
  - Smart search

## Hardware Requirements

- **GPU**: NVIDIA P2000 (or compatible CUDA GPU)
- **NVIDIA Drivers**: Latest proprietary drivers
- **NVIDIA Container Toolkit**: Required for Docker GPU access
- **Memory**: 4GB+ RAM recommended for ML models

## Prerequisites

### 1. Install NVIDIA Drivers

Check if drivers are installed:
```bash
nvidia-smi
```

If not installed, install proprietary drivers:
```bash
sudo apt update
sudo apt install nvidia-driver-535  # Or latest version
sudo reboot
```

### 2. Install NVIDIA Container Toolkit

```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
  sudo tee /etc/apt/sources.list.d/nvidia-docker.list

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit
sudo systemctl restart docker
```

### 3. Verify GPU Access in Docker

```bash
docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
```

You should see your P2000 GPU listed.

## Quick Start

**See [DOCKER-UI-DEPLOYMENT.md](DOCKER-UI-DEPLOYMENT.md) for detailed web UI deployment instructions.**

Quick overview:
1. Ensure prerequisites are installed (NVIDIA drivers, Container Toolkit)
2. Configure firewall to allow mini PC access to port 3003
3. Deploy via Docker web UI using settings in DOCKER-UI-DEPLOYMENT.md
4. Verify GPU access and connectivity
5. Configure mini PC Immich stack to connect to this ML service

## Network Configuration

The ML container exposes port 3003 for the mini PC to connect to.

### Firewall Configuration

Allow traffic from mini PC:
```bash
# Replace 192.168.1.50 with your mini PC IP
sudo ufw allow from 192.168.1.50 to any port 3003
```

### Test Connectivity from Mini PC

From the mini PC, run:
```bash
curl http://FILESERVER_IP:3003/ping
```

You should get a response indicating the service is accessible.

## Model Management

### Model Cache

ML models are cached in the `immich_model_cache` volume:
- **First run**: Models download (1-2GB), takes a few minutes
- **Subsequent runs**: Models load from cache, much faster

### Clear Model Cache

If you need to re-download models:
```bash
docker compose down
docker volume rm immich_model_cache
docker compose up -d
```

## Monitoring

### Check GPU Usage

```bash
# Real-time GPU monitoring
watch -n 1 nvidia-smi

# Check container GPU usage
docker stats immich_machine_learning
```

### View Logs

```bash
docker compose logs -f
```

### Container Status

```bash
docker compose ps
```

## Performance Tuning

### GPU Memory

The P2000 has 5GB VRAM. If you experience memory issues:

1. Limit CUDA devices in `.env`:
   ```bash
   CUDA_VISIBLE_DEVICES=0
   ```

2. Monitor memory: `nvidia-smi`

### Model Settings

Models are automatically optimized for your GPU. No manual configuration needed.

## Troubleshooting

### GPU Not Detected

1. Check NVIDIA drivers: `nvidia-smi`
2. Verify Docker runtime: `docker info | grep -i runtime`
3. Check container toolkit: `nvidia-container-cli info`
4. Restart Docker: `sudo systemctl restart docker`

### Container Crashes

Check logs:
```bash
docker compose logs
```

Common issues:
- Insufficient GPU memory (restart helps)
- CUDA version mismatch (update drivers)
- Model download failures (check internet connection)

### Connection Issues from Mini PC

1. Test network: `ping FILESERVER_IP`
2. Test port: `nc -zv FILESERVER_IP 3003`
3. Check firewall: `sudo ufw status`
4. Verify container: `docker ps | grep immich_machine_learning`

### Performance Issues

- Ensure GPU drivers are up to date
- Check GPU temperature: `nvidia-smi`
- Monitor system resources: `htop`
- Verify no other processes using GPU

## Updates

```bash
docker compose pull
docker compose up -d
```

## Resources

- [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-docker)
- [Immich ML Documentation](https://immich.app/docs/features/ml)
- [CUDA Compatibility](https://docs.nvidia.com/deploy/cuda-compatibility/)
