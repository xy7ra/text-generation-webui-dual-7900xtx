# Text Generation WebUI - Dual AMD Radeon RX 7900 XTX Setup

Optimized configuration for running text-generation-webui with dual AMD Radeon RX 7900 XTX GPUs on ROCm.

## Hardware Configuration

**CPU:** AMD Ryzen 9 9900X 12-Core Processor (5.65 GHz Max)  
**GPUs:** 2x AMD Radeon RX 7900 XTX (24GB VRAM each = 48GB total)  
**Motherboard:** ASRock X870E Taichi Lite  
**BIOS Version:** 3.50 (09/18/2025)  
**ROCm Version:** 6.2.4  

## System Configuration

### BIOS Settings (Required)
- **SVM (AMD-V):** Enabled
- **IOMMU:** Enabled
- **Above 4G Decoding:** Enabled
- **Resizable BAR:** Enabled

### Kernel Parameters
Add to `/etc/default/grub` GRUB_CMDLINE_LINUX_DEFAULT:
```bash
amdgpu.ppfeaturemask=0xffffffff 
amdgpu.multi_gpu_support=1 
amdgpu.dc=1 
amdgpu.vm_fragment_size=9 
amdgpu.noretry=0
```

Then update GRUB:
```bash
sudo update-grub
sudo update-initramfs -u
sudo reboot
```

### Environment Variables
```bash
export HSA_OVERRIDE_GFX_VERSION=11.0.0
export PYTORCH_ROCM_ARCH=gfx1100
export HIP_VISIBLE_DEVICES=0,1
export ROCR_VISIBLE_DEVICES=0,1
```

## Docker Setup (Recommended)

### Build Docker Image
```bash
docker build -f Dockerfile.rocm-dual-7900xtx -t textgen-dual-7900xtx .
```

### Run Container
```bash
docker run -it --rm \
  --device=/dev/kfd \
  --device=/dev/dri \
  -e HSA_OVERRIDE_GFX_VERSION=11.0.0 \
  -e HIP_VISIBLE_DEVICES=0,1 \
  -p 7860:7860 \
  textgen-dual-7900xtx
```

## Tensor Parallel Mode

To use both GPUs with tensor parallelism for single large models:

```bash
python server.py \
  --model meta-llama/Llama-3.1-70B-Instruct \
  --tensor-parallel-size 2 \
  --gpu-memory-utilization 0.92 \
  --listen
```

## Performance Notes

- **Single GPU:** ~24GB VRAM, good for models up to 30B parameters
- **Dual GPU (spillover):** ~48GB VRAM, fits models up to 70B parameters
- **Dual GPU (tensor parallel):** Near 2x performance on large models
- **Batch Processing:** Both GPUs work simultaneously

## Verified Working

- ✅ PyTorch 2.3.0 + ROCm 6.2
- ✅ Both GPUs detected and accessible
- ✅ Tensor parallelism support
- ✅ Docker containerized setup

## Known Issues

- vLLM 0.11+ requires PyTorch 2.9 (CUDA only) - not compatible
- LM Studio only supports GPU spillover, not tensor parallelism
- hipBLASLt warnings are expected for gfx1100 (non-critical)

## Alternative: venv Setup

If you prefer not to use Docker:

```bash
python3 -m venv venv
source venv/bin/activate
pip install torch==2.5.1 torchvision==0.20.1 torchaudio==2.5.1 \
    --index-url https://download.pytorch.org/whl/rocm6.2
pip install -r requirements.txt
```

## Testing GPU Access

```bash
python -c "
import torch
print('PyTorch:', torch.__version__)
print('ROCm Available:', torch.cuda.is_available())
print('GPU Count:', torch.cuda.device_count())
for i in range(torch.cuda.device_count()):
    print(f'GPU {i}: {torch.cuda.get_device_name(i)}')
"
```

Expected output:
```
PyTorch: 2.3.0+rocm6.2
ROCm Available: True
GPU Count: 2
GPU 0: AMD Radeon RX 7900 XTX
GPU 1: AMD Radeon RX 7900 XTX
```

## Credits

- Original project: [oobabooga/text-generation-webui](https://github.com/oobabooga/text-generation-webui)
- ROCm documentation: [ROCm Documentation](https://rocm.docs.amd.com/)
- Configuration tested and documented by: xy7ra

## License

Same as upstream text-generation-webui
