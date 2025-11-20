# Immich Machine Learning - Docker Web UI Deployment

This guide explains how to deploy the Immich ML container using a Docker web UI (Portainer, Yacht, etc.) on the fileserver.

## Container Configuration

Use these settings when creating the container in your Docker web UI:

### Basic Settings

- **Container Name**: `immich_machine_learning`
- **Image**: `ghcr.io/immich-app/immich-machine-learning:release`
- **Restart Policy**: `unless-stopped`

### Port Mapping

Map the following port:
- **Host Port**: `3003` → **Container Port**: `3003`

### Environment Variables

Add these environment variables:

| Variable | Value |
|----------|-------|
| `TZ` | `America/New_York` (or your timezone) |
| `LOG_LEVEL` | `log` |
| `NVIDIA_VISIBLE_DEVICES` | `all` |
| `NVIDIA_DRIVER_CAPABILITIES` | `compute,utility` |

### Volumes

Create and mount a volume for model caching:

| Host Path / Volume | Container Path | Mode |
|-------------------|----------------|------|
| `immich_model_cache` (named volume) | `/cache` | Read/Write |

### GPU Configuration

**CRITICAL:** Enable GPU access for this container.

In **Portainer:**
1. Scroll to "Runtime & Resources"
2. Enable "GPU Devices"
3. Select "All GPUs" or choose the P2000 specifically
4. Or add custom runtime: `--gpus all`

In **Yacht:**
1. Under "Devices", add GPU
2. Set device capabilities to include GPU

**Alternative - Manual Docker Run Command:**
If your UI doesn't support GPU configuration, use this command in the terminal:

```bash
docker run -d \
  --name immich_machine_learning \
  --restart unless-stopped \
  --gpus all \
  -p 3003:3003 \
  -e TZ=America/New_York \
  -e LOG_LEVEL=log \
  -e NVIDIA_VISIBLE_DEVICES=all \
  -e NVIDIA_DRIVER_CAPABILITIES=compute,utility \
  -v immich_model_cache:/cache \
  ghcr.io/immich-app/immich-machine-learning:release
```

## Prerequisites

Before deploying, ensure:

1. **NVIDIA Drivers Installed**
   ```bash
   nvidia-smi  # Should display your P2000 GPU
   ```

2. **NVIDIA Container Toolkit Installed**
   ```bash
   distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
   curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
   curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
     sudo tee /etc/apt/sources.list.d/nvidia-docker.list

   sudo apt-get update
   sudo apt-get install -y nvidia-container-toolkit
   sudo systemctl restart docker

   # Verify GPU access in Docker
   docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
   ```

3. **Firewall Configuration**
   ```bash
   # Allow mini PC to connect (replace with your mini PC IP)
   sudo ufw allow from 192.168.1.50 to any port 3003
   ```

## Verification

After deployment:

1. **Check Container Logs:**
   ```bash
   docker logs immich_machine_learning
   ```
   Should see ML models loading.

2. **Verify GPU Usage:**
   ```bash
   nvidia-smi
   ```
   Should show the `immich_machine_learning` process using GPU memory.

3. **Test Connectivity:**
   ```bash
   curl http://localhost:3003/ping
   ```
   Should return a response.

4. **Test from Mini PC:**
   ```bash
   curl http://FILESERVER_IP:3003/ping
   ```
   Should be accessible from mini PC.

## Network Setup

Make note of this container's details for configuring the main Immich stack:

- **ML Service URL**: `http://FILESERVER_IP:3003`
- This URL goes in the `IMMICH_MACHINE_LEARNING_URL` variable in the mini PC's Immich `.env` file

## Troubleshooting

### Container Won't Start

1. Check Docker logs: `docker logs immich_machine_learning`
2. Verify GPU access: `nvidia-smi`
3. Ensure NVIDIA Container Toolkit is installed
4. Check runtime: `docker info | grep -i runtime`

### GPU Not Detected

1. Restart Docker: `sudo systemctl restart docker`
2. Verify toolkit: `nvidia-container-cli info`
3. Check container was started with `--gpus all` flag

### Mini PC Can't Connect

1. Test firewall: `sudo ufw status`
2. Test port: `nc -zv FILESERVER_IP 3003` (from mini PC)
3. Check container is running: `docker ps | grep immich`
4. Verify port mapping in container settings

### Model Download Issues

- First run downloads 1-2GB of models
- Check internet connectivity
- Monitor with: `docker logs -f immich_machine_learning`
- May take 5-10 minutes on first start

## Updates

To update the container:

1. Pull latest image:
   ```bash
   docker pull ghcr.io/immich-app/immich-machine-learning:release
   ```

2. Stop and remove old container (via UI or command):
   ```bash
   docker stop immich_machine_learning
   docker rm immich_machine_learning
   ```

3. Recreate container with same settings (volume persists models)

Or use the web UI's update/recreate function.

## Performance Notes

- **First Run**: Models download, takes 5-10 minutes
- **Subsequent Runs**: Models load from cache, 30-60 seconds
- **GPU Memory**: Uses ~2-3GB of VRAM (P2000 has 5GB total)
- **Idle State**: Minimal GPU usage when not processing
- **Active Processing**: Full GPU utilization during facial recognition/object detection

## Reference

The `compose.yml` file in this directory is provided as a reference for the container configuration. You can use it with Docker Compose if preferred, but it's designed to match the web UI deployment settings above.
