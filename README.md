# ImageGenAI

A flexible, containerized API server for image generation that supports multiple AI models with seamless CPU/GPU switching.

## Overview

ImageGenAI provides a unified interface for generating images using various AI models. Built for flexibility, it supports both CPU and GPU environments, making it suitable for any deployment scenario from development to production.

## Supported Models

### 1. CPU Mode (Stable Diffusion v1.5)
- Optimized for CPU-only environments
- Configuration: `.env.cpu`
```bash
cp .env.cpu .env
docker compose -f docker-compose.yml up --build  # Note: No GPU compose file
```
Features:
- Resolution: 512x512
- Uses runwayml/stable-diffusion-v1-5
- CPU optimizations enabled
- Lower memory footprint
- Good for systems without GPU

### 2. SDXL (Stable Diffusion XL)
- High-quality image generation
- Configuration: `.env.sdxl`
```bash
cp .env.sdxl .env
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up --build
```
Features:
- Resolution: 1024x1024
- VAE Tiling enabled
- Best for high-quality, detailed images
- Excellent for photorealistic outputs
- Enhanced image quality

### 3. SDXL Turbo
- Fast image generation
- Configuration: `.env.sdxl-turbo`
```bash
cp .env.sdxl-turbo .env
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up --build
```
Features:
- Resolution: 512x512
- Fast inference (4-8 steps)
- Best for quick iterations
- Good for real-time applications
- Optimized for speed

### 4. PixArt-α LCM
- Fast, high-quality generation
- Configuration: `.env.pixart`
```bash
cp .env.pixart .env
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up --build
```
Features:
- Resolution: 768x768 (optimized for mobile/web)
- LCM variant for faster inference
- Configurable precision (float16/float32)
- Best for artistic images
- Excellent style consistency

## Key Features

- Unified API interface for all models
- Seamless CPU/GPU switching
- Docker containerization
- Automatic model caching
- Memory optimization
- Resource cleanup
- Environment-based configuration

## Installation

You can either use the stable release or clone the repository:

### Option 1: Using Release

Check the latest stable release from https://github.com/raunakkathuria/imagegenai/releases.
Download and extract the releases for a specific version, example [v0.0.1 release](https://github.com/raunakkathuria/imagegenai/releases/tag/v0.0.1):

```bash
wget https://github.com/raunakkathuria/imagegenai/archive/refs/tags/v0.0.1.tar.gz
tar xzf v0.0.1.tar.gz
cd imagegenai-0.0.1
```

### Option 2: Using Git

```bash
git clone https://github.com/raunakkathuria/imagegenai.git
cd imagegenai
```

## API Usage

### Generate Image

The API supports both custom prompts and negative prompts, with flexible request options:

1. Using environment defaults (no request body needed):
```bash
curl -X POST "http://localhost:8080/generate"
```

2. Custom prompt only:
```bash
curl -X POST "http://localhost:8080/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "custom_prompt": "your prompt here"
  }'
```

3. With custom negative prompt:
```bash
curl -X POST "http://localhost:8080/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "custom_prompt": "your prompt here",
    "negative_prompt": "things to avoid in the image"
  }'
```

4. Runtime parameter adjustment:
```bash
curl -X POST "http://localhost:8080/generate" \
  -H "Content-Type: application/json" \
  -d '{
    "custom_prompt": "your prompt here",
    "num_inference_steps": 30,
    "guidance_scale": 7.5,
    "height": 768,
    "width": 768
  }'
```

Parameter Ranges:
- num_inference_steps: 1-150 steps
- guidance_scale: 0.0-20.0
- height/width: 128-1024 (must be multiple of 8)

Response:
```json
{
  "message": "Image generated successfully",
  "filename": "output/image_20240227_123456_abcd1234.png"
}
```

Notes:
- All parameters are optional
- If no request body is provided, uses environment defaults
- Parameters can be adjusted at runtime without restarting
- Mix and match any combination of parameters
- Valid ranges enforced by API validation
- Changes only affect current generation

## Resource Management

### Directory Structure
```
./cache   - HuggingFace cache (~22GB for full downloads)
./models  - Model cache (saved model files)
./output  - Generated images
./assets  - Example images with parameter variations
```

### Assets Organization

The `assets/` folder contains example images generated with different parameters, organized by model:

```
assets/
├── pixart/          - PixArt-α model results
├── runwaymlsdv1.5/  - RunwayML v1.5 model results
├── sdv1.4/          - Stable Diffusion v1.4 results
├── sdxl/            - SDXL base model results
└── sdxlturbo/       - SDXL Turbo model results
```

Image naming convention:
```
[model]-steps-[inference_steps]-guidance-[guidance_scale]-[dimension].png
```
Example: `pixart-steps-4-guidance-0-768-best.png`
- Model: pixart
- Inference steps: 4
- Guidance scale: 0.0
- Dimension: 768x768
- Suffix 'best': Indicates optimal parameters for this model

### Initial Setup
```bash
# Create required directories with proper permissions
mkdir -p cache models output
chmod 777 cache models output  # Ensure Docker has write permissions
```

### Memory Management

Each model configuration includes optimized memory settings:
- GPU Memory Usage: Configurable via MAX_MEMORY (60-80%)
- Attention Slicing enabled for all models
- VAE Tiling for SDXL high-res images
- Configurable precision (float16/float32)
- Automatic device placement

### Memory Optimization Features:
1. Dynamic Memory Allocation
   - Automatic GPU memory management
   - Efficient CPU offloading when needed
   - Proper meta device handling

2. Memory Efficiency Options:
   - Attention slicing for reduced memory footprint
   - VAE tiling for high-resolution images
   - Sequential CPU offloading
   - Memory-efficient attention mechanisms

3. Cleanup Processes:
   - Automatic cache clearing
   - Proper resource deallocation
   - Garbage collection optimization

### Disk Space Management

1. Cache Cleanup:
```bash
# Remove specific model cache
rm -rf ./models/stable-diffusion-xl-base-1.0
rm -rf ./cache/models--stabilityai--stable-diffusion-xl-base-1.0

# Clean all caches (will re-download on next use)
rm -rf ./cache/* ./models/*
```

2. Monitor Usage:
```bash
# Check cache sizes
du -sh ./cache ./models ./output

# Monitor during usage
watch -n 10 'du -sh ./cache ./models ./output'
```

## Performance Tips

1. CPU vs GPU Mode:
```bash
# For CPU-only mode (using Stable Diffusion v1.5)
cp .env.cpu .env
docker compose -f docker-compose.yml up --build

# For GPU mode (SDXL, SDXL Turbo, or PixArt)
cp .env.[model] .env  # where [model] is sdxl, sdxl-turbo, or pixart
docker compose -f docker-compose.yml -f docker-compose.gpu.yml up --build
```

2. Single Worker Mode (Recommended for most cases):
```bash
WORKERS=1 docker compose -f docker-compose.yml -f docker-compose.gpu.yml up
```

3. Multiple Workers (For high-throughput scenarios):
```bash
WORKERS=2 docker compose -f docker-compose.yml -f docker-compose.gpu.yml up
```

4. Optimization Guidelines:
   - Use appropriate resolution for each model
   - Adjust inference steps based on quality needs
   - Monitor GPU memory usage
   - Enable memory optimizations as needed

## Environment Variables

Common settings across all models:
- `USE_GPU`: Enable/disable GPU usage
- `TORCH_DTYPE`: Precision (float32/float16)
- `MAX_MEMORY`: GPU memory limit
- `WORKERS`: Number of server workers
- `HEIGHT/WIDTH`: Output image dimensions
- `NUM_INFERENCE_STEPS`: Generation steps
- `GUIDANCE_SCALE`: CFG scale

Advanced settings:
- `ENABLE_ATTENTION_SLICING`: Memory optimization
- `ENABLE_VAE_TILING`: For high-res images
- `EMPTY_CACHE_BETWEEN_RUNS`: Memory management
- `ENABLE_MODEL_CPU_OFFLOAD`: Resource optimization

## Model-Specific Notes

1. CPU Mode (SD v1.5):
   - Optimized for systems without GPU
   - Uses CPU offloading for memory efficiency
   - Good balance of quality and speed
   - Works on any system

2. SDXL:
   - Uses VAE Tiling for high-res images
   - Best with higher inference steps (30-50)
   - Excellent for detailed compositions
   - Supports high-resolution outputs
   - Requires GPU

3. SDXL Turbo:
   - Optimized for speed (4-8 steps)
   - Works well with low inference steps
   - Perfect for rapid prototyping
   - Good quality-speed balance
   - Requires GPU

4. PixArt-α LCM:
   - Configurable precision (float16/float32)
   - No guidance needed (scale=0.0)
   - Fast inference with LCM variant
   - 768x768 resolution for optimal quality
   - Excellent for artistic styles
   - Requires GPU

## Requirements

See `requirements.txt` for full dependencies. Key components:
- Python 3.11+
- PyTorch 2.2+
- diffusers 0.24.0
- transformers 4.35.2
- accelerate 0.25.0
- ftfy 6.1.3 (for caption cleaning)

## Docker Support

The application is fully dockerized with:
- Multi-stage builds for optimization
- GPU support via NVIDIA Container Toolkit
- Volume mounting for persistent storage
- Environment-based configuration

### NVIDIA Setup (Required for GPU Models Only)

1. Install NVIDIA Driver:
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install nvidia-driver-535  # or latest version
```

2. Install NVIDIA Container Toolkit:
```bash
# Add NVIDIA package repositories
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Install NVIDIA Container Toolkit
sudo apt update
sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

3. Verify Installation:
```bash
# Check NVIDIA driver
nvidia-smi

# Test Docker GPU access
docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
```

Note: For other distributions or detailed instructions, visit:
- [NVIDIA Container Toolkit Installation Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- [NVIDIA Driver Installation Guide](https://docs.nvidia.com/datacenter/tesla/tesla-installation-notes/index.html)

## Error Handling

The API includes robust error handling:
- Memory management errors
- Model loading issues
- Generation failures
- Resource cleanup
- Proper error reporting

## Development

### Local Setup
1. Choose installation method (release or git)
2. Install dependencies
3. Copy appropriate .env file
4. Run with docker-compose

### Contributing
- Follow PEP 8 guidelines
- Include tests for new features
- Update documentation
- Submit pull requests

## License
This project is open source and available under the MIT License.
