---
{"dg-publish":true,"permalink":"/Program/AI/ComfyUI 常见模型目录的意义与用途/","noteIcon":"","created":"2025-12-01T16:49:18.785+08:00"}
---

###  checkpoints

**最重要的模型目录之一**  
用于存放 **主模型 / 基础模型**，例如：

- SD1.5 / SD2.1 → `*.ckpt` / `*.safetensors`
- SDXL 基础模型 → `sd_xl_base_1.0.safetensors`
- Anime/Anything 差分大模型
- 越大的模型越需要放这里
    

📌 **用途：**  
在 Workflow 中常用的 **Checkpoint Loader / Load Checkpoint** 节点会从这里读取。

---

###   vae

用于存放 **VAE（变分自编码器）** 文件。  
常见后缀：`*.vae.pt`、`*.safetensors`

📌 **用途：**  
决定输出风格偏“糊/清晰/偏色”，Anime 类模型常建议替换 VAE。

---

###  loras

存放 **LoRA 模型**，用于强化某种风格、人物、画风。

📌 **用途：**  
LoRA Loader、LoRA Stack 节点会从这里读取。

✨ 例如：

- 二次元风格 LoRA
    
- 某演员脸部 LoRA
    
- 动漫角色服装 LoRA
    

---

### embeddings

存放 **文本嵌入 (textual inversion / TI)**  
文件名类似：

- `badhandv4.pt`
    
- `neg_hand.pt`
    

📌 **用途：**  
在 Prompt 文本中直接写 `"badhandv4"` 就能调用。

---

###  unet

存放 **UNet 优化模型 / 训练中间文件**  
只有某些节点使用，例如：

- Diffusers
    
- ACN
    
- Turbo 模型（如 SDXL Turbo）
    

普通用户一般较少手动放文件进去。

---

###  controlnet

存放 **ControlNet 模型**  
如：

- Canny
- Depth
- SoftEdge
- OpenPose
- Face Swap (IPAdapter OpenPose)
- Lineart Anime
    

📌 **用途：**  
ControlNet Loader 节点自动加载。

---

###   ipadapter

存放 **IP-Adapter 模型**  
例如：

- `ip-adapter-plus-face_sd15.safetensors`
- `ip-adapter-xl-face.safetensors`
    
📌 主用途：  
图生图的“风格参考”“人脸保真”。

---

### clip

存放 **CLIP 文本编码器**  
通常随 SDXL 使用，为 prompt 做编码。

---

###   upscale_models

图像超分模型，例如：

- ESRGAN
- RealESRGAN
- 4xAnimeSharp
    

用于 UpScaler 节点。

---

###  animatediff

用于 **AnimateDiff 动画生成** 的模型：

- Motion modules
- Full models
- V3 运动扩散模型
    

---

### 总结表格（最快速理解）

|目录|用途|示例|
|---|---|---|
|checkpoints|主模型|SDXL_base.safetensors|
|vae|编码器|animevae.pt|
|loras|LoRA|anime-style.safetensors|
|embeddings|TI|badhandv4.pt|
|controlnet|ControlNet|openpose.safetensors|
|ipadapter|IP-Adapter|ip-adapter-face.safetensors|
|clip|文本编码器|clip_g.safetensors|
|upscale_models|超分模型|realesrgan_x4.pth|
|animatediff|动画|mm_sd15_v3.safetensors|
|unet|UNet结构|sd15_unet_fp16.pt|
