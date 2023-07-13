## 🤗  D🧨 ffusers调研

> 由于本人非算法背景、本文未描述任何算法原理、尽管如此，还是会有很多地方说的可能不正确，有任何问题都欢迎指正。

## Stable Diffusion基本概念与原理

![sd-pipeline](http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-07-13-053448.png)

文生推理阶段主要流程：

1. 生成一个随机的噪声，外加根据prompt生成prompt_embeds；

2. Stable Diffusion的核心逻辑是一个循环，每个循环将降低一些图像的噪声，将图像变的更清晰。其中每个循环：

- 将图像A和clip prompt_embeds输入模型预测噪声残差，然后将噪声残差、图像 A、step_no 输入采样器预测更清晰的图像B;

- 将图像B和clip prompt_embeds输入模型预测噪声残差..循环N次,得到图像C

3. 最终将图像 C 输入 VAE 模型,得到更加清晰的图像。

以下是对涉及模块的通俗解释。

### CLIP text-encoder

说白了，就是将prompt转换成prompt_embeds，而这个prompt_embeds可包含图像特征。

> CLIP 是一个多模态学习框架，涉及到同时训练文本编码器和图像编码器。这种联合训练有助于让文本编码器和图像编码器之间共享相同的语义空间。这使得当我们从文本编码器得到文本的向量表示时，能够与图像编码器生成的图像向量进行直接比较。
>
> 得益于这种联合训练，CLIP 文本编码器可以广泛应用于各种自然语言处理和多模态学习任务，比如图像分类、图像检索、图像生成等。通过利用文本编码器生成的文本向量表示，模型能更好地关联文本和图像信息，从而在这些任务中实现高性能。            
>
> -- GPT4生成内容

> Inspired by [Imagen](https://imagen.research.google/), Stable Diffusion does **not** train the text-encoder during training and simply uses an CLIP's already trained text encoder, [CLIPTextModel](https://huggingface.co/docs/transformers/model_doc/clip#transformers.CLIPTextModel).             --- [diffusers_intro colab](https://colab.research.google.com/github/huggingface/notebooks/blob/main/diffusers/diffusers_intro.ipynb)

### 模型

将prompt_embeds，带噪声的图像 输入模型 推理得到 噪声残差。即 model(prompt_embeds, 带噪声的图像) =  噪声残差

>  Without going in too much detail, the model is usually not trained to directly predict a slightly less noisy image, but rather to predict the "noise residual" which is the difference between a less noisy image and the input image (for a diffusion model called "DDPM") or, similarly, the gradient between the two time steps (like the diffusion model called "Score VE"). 

### 采样器

将噪声残差、图像 、推理步骤数 输入采样器预测更清晰的图像; 即sample(噪声残差, 推理步骤数, 带噪声的图像) = 更清晰的图像

### VAE

可以理解为一种压缩算法、encode 就是压缩、尽可能保留图像特征的同时压缩图片的大小，这样处理更快、更节省资源；decode 就是解压缩、恢复成原有的图片； 这是latent diffusion又快又高效的关键所在。

> The VAE model has two parts, an encoder and a decoder. The encoder is used to convert the image into a low dimensional latent representation, which will serve as the input to the U-Net model. The decoder, conversely, transforms the latent representation back into an image.
>
> Since latent diffusion operates on a low dimensional space, it greatly reduces the memory and compute requirements compared to pixel-space diffusion models. For example, the autoencoder used in Stable Diffusion has a reduction factor of 8. This means that an image of shape `(3, 512, 512)` becomes `(3, 64, 64)` in latent space, which requires `8 × 8 = 64` times less memory.

<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-07-13-053444.png" alt="img" style="zoom:67%;" />

## diffusers库

当前aigc概念火热，ai绘画作为其中的一个分支，随着stable diffusion的开源，大家纷纷开始**自研**。目前比较热门的开源项目有：

[Automatic1111 Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui)

[ComfyUI](https://github.com/comfyanonymous/ComfyUI)

[InvokeAI](https://github.com/invoke-ai/InvokeAI)

其中最为火热的是[Automatic1111 Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui)，功能也是最齐全的，不过代码结构和风格导致其不太合适自研(二次开发)。因此就想看有没有更适合二次开发的项目。

一种方式是像Automatic1111 Stable Diffusion WebUI一样，直接基于[CompVis/stable-diffusion](https://github.com/CompVis/stable-diffusion)、[runwayml/stable-diffusion](https://github.com/runwayml/stable-diffusion)、[Stability-AI/stablediffusion](https://github.com/Stability-AI/stablediffusion)、[https://github.com/crowsonkb/k-diffusion](https://github.com/crowsonkb/k-diffusion)等开源库进行开发，这样工作量会非常大。

而另外一种方式是基于diffusers库进行开发。

diffusers是huggingface开源的、一个diffusion models的工具库，包含unet、采样器、vae等实现，同时也支持将类似文生图、图生图的逻辑抽象成pipeline。

> 🤗 Diffusers is the go-to library for state-of-the-art pretrained diffusion models for generating images, audio, and even 3D structures of molecules. 

 另外[InvokeAI](https://github.com/invoke-ai/InvokeAI)的后端已切换为有diffusers驱动。

以下内容主要介绍diffusers使用、原理分析以及如何复现[Automatic1111 Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui)所支持的功能。

### 简单的实例

```python
from diffusers import DiffusionPipeline
from torch import autocast
# 加载模型配置、初始化 pipeline
pipeline = DiffusionPipeline.from_pretrained("runwayml/stable-diffusion-v1-5").to("cuda")
prompt = "a photo of an astronaut riding a horse on mars"

with autocast("cuda"):
    # 执行推理、StableDiffusionPipeline的__call__函数
    image = pipeline(prompt)[0][0]

image.save("astronaut_rides_horse.png")
```

### 概念：Schedulers 与 Models 

diffusers中的 models是预测残差、而 schedulers其实就是上文中所说的采样器,主要作用是增加噪声用于训练和将 models预测的残差应用到 有噪声的图像上生成更清晰的图片。(U-Net Model)

> **Schedulers** define the noise schedule which is used to add noise to the model during training, and also define the algorithm to **compute** the slightly less noisy sample given the model output (here `noisy_residual`). 
>
> The model **predict** the noise residual or slightly less noisy image with its trained weights, while the scheduler **computes** the previous sample given the model's output.

Schedulers配置参数

\- `num_train_timesteps` defines the length of the denoising process, e.g. how many timesteps are need to process random gaussian noise to a data sample.

## diffusers代码结构

diffusers代码结构合理、简单、外部依赖少，是开发人员二次开发diffusion models的首选。

```bash
├── docs
├── examples
├── Makefile
├── MANIFEST.in
├── README.md
├── scripts
├── setup.py
├── src
│   └── diffusers
│       ├── commands
│       ├── models 模型实现相关类，如 unet、vae
│       │   ├── activations.py
│       │   ├── attention_processor.py
│       │   ├── attention.py
│       │   ├── autoencoder_kl.py
│       │   ├── controlnet.py
│       │   ├── cross_attention.py
│       │   ├── dual_transformer_2d.py
│       │   ├── embeddings.py
│       │   ├── __init__.py
│       │   ├── modeling_flax_pytorch_utils.py
│       │   ├── modeling_flax_utils.py
│       │   ├── modeling_utils.py
│       │   ├── prior_transformer.py
│       │   ├── resnet.py
│       │   ├── t5_film_transformer.py
│       │   ├── transformer_2d.py
│       │   ├── transformer_temporal.py
│       │   ├── unet_1d_blocks.py
│       │   ├── unet_1d.py
│       │   ├── unet_2d_blocks_flax.py
│       │   ├── unet_2d_blocks.py
│       │   ├── unet_2d_condition_flax.py
│       │   ├── unet_2d_condition.py
│       │   ├── unet_2d.py
│       │   ├── unet_3d_blocks.py
│       │   ├── unet_3d_condition.py
│       │   ├── vae.py
│       │   └── vq_model.py
│       ├── pipelines 任务逻辑抽象层
│       │   ├── audio_diffusion
│       │   ├── controlnet
│       │   ├── dance_diffusion
│       │   ├── ddim
│       │   ├── ddpm
│       │   ├── kandinsky
│       │   ├── latent_diffusion
│       │   ├── latent_diffusion_uncond
│       │   ├── repaint
│       │   ├── score_sde_ve
│       │   ├── semantic_stable_diffusion
│       │   ├── spectrogram_diffusion
│       │   ├── stable_diffusion
│       │   ├── stable_diffusion_safe
│       │   ├── text_to_video_synthesis
│       │   ├── unclip
│       │   ├── unidiffuser
│       │   ├── versatile_diffusion
│       │   └── vq_diffusion
│       ├── schedulers  对应sd webui的采样器
│       │   ├── __init__.py
│       │   ├── README.md
│       │   ├── scheduling_ddim_flax.py
│       │   ├── scheduling_ddim.py
│       │   ├── scheduling_ddpm.py
│       │   ├── scheduling_dpmsolver_sde.py
│       │   ├── scheduling_euler_ancestral_discrete.py
│       │   ├── scheduling_euler_discrete.py
│       │   ├── scheduling_heun_discrete.py
│       │   ├── scheduling_ipndm.py
│       │   ├── scheduling_karras_ve.py
│       │   ├── scheduling_k_dpm_2_ancestral_discrete.py
│       │   ├── scheduling_k_dpm_2_discrete.py
│       │   ├── scheduling_lms_discrete.py
│       │   ├── scheduling_pndm.py
│       │   ├── scheduling_repaint.py
│       │   ├── scheduling_sde_ve.py
│       │   ├── scheduling_sde_vp.py
│       │   ├── scheduling_unclip.py
│       │   ├── scheduling_utils.py
│       │   └── scheduling_vq_diffusion.py
│       └── utils
├── tests
│   ├── models
│   ├── pipelines
│   └── schedulers
├── _typos.toml
└── utils
```

## diffusers核心逻辑

###Stable Diffusion加载模型、初始化 pipeline

#### from_pretrained

下载或者使用本地的模型、根据model_index.json初始化 pipeline 以及其所需要的模块

1. 下载模型或者直接使用本地的模型；pretrained_model_name_or_path 

   ```bash
   models--runwayml--stable-diffusion-v1-5
   ├── blobs
   │   ├── 193490b58ef62739077262e833bf091c66c29488058681ac25cf7df3d8190974
   │   ├── 1a02ee8abc93e840ffbcb2d68b66ccbcb74b3ab3
   │   ├── 1b134cded8eb78b184aefb8805b6b572f36fa77b255c483665dda93
   │   ├── 2c2130b544c0c5a72d5d00da071ba130a9800fb2
   │   ├── 469be27c5c010538f845f518c4f5e8574c78f7c8
   │   ├── 4d3e873ab5086ad989f407abd50fdce66db8d657
   │   ├── 5294955ff7801083f720b34b55d0f1f51313c5c5
   │   ├── 55d78924fee13e4220f24320127c5f16284e13b9
   │   ├── 5ba7bf706515bc60487ad0e1816b4929b82542d6
   │   ├── 5dbd88952e7e521aa665e5052e6db7def3641d03
   │   ├── 76e821f1b6f0a9709293c3b6b51ed90980b3166b
   │   ├── 770a47a9ffdcfda0b05506a7888ed714d06131d60267e6cf52765d61cf59fd67
   │   ├── 82d05b0e688d7ea94675678646c427907419346e
   │   ├── c7da0e21ba7ea50637bee26e81c220844defdf01aafca02b2c42ecd
   │   └── daf7e2e2dfc64fb437a2b44525667111b00cb9fc
   ├── refs
   │   └── main
   └── snapshots
       └── aa9ba505e1973ae5cd05f5aedd345178f52f8e6a
           ├── feature_extractor
           │   └── preprocessor_config.json -> ../../../blobs/5294955ff7801083f720b34b55d0f1f51313c5c5
           ├── model_index.json -> ../../blobs/daf7e2e2dfc64fb437a2b44525667111b00cb9fc
           ├── safety_checker
           │   ├── config.json -> ../../../blobs/5dbd88952e7e521aa665e5052e6db7def3641d03
           │   └── pytorch_model.bin -> ../../../blobs/193490b58ef62739077262e833bf091c66c29488058681ac25cf7df3d8190974
           ├── scheduler
           │   └── scheduler_config.json -> ../../../blobs/82d05b0e688d7ea94675678646c427907419346e
           ├── text_encoder
           │   ├── config.json -> ../../../blobs/4d3e873ab5086ad989f407abd50fdce66db8d657
           │   └── pytorch_model.bin -> ../../../blobs/770a47a9ffdcfda0b05506a7888ed714d06131d60267e6cf52765d61cf59fd67
           ├── tokenizer
           │   ├── merges.txt -> ../../../blobs/76e821f1b6f0a9709293c3b6b51ed90980b3166b
           │   ├── special_tokens_map.json -> ../../../blobs/2c2130b544c0c5a72d5d00da071ba130a9800fb2
           │   ├── tokenizer_config.json -> ../../../blobs/5ba7bf706515bc60487ad0e1816b4929b82542d6
           │   └── vocab.json -> ../../../blobs/469be27c5c010538f845f518c4f5e8574c78f7c8
           ├── unet
           │   ├── config.json -> ../../../blobs/1a02ee8abc93e840ffbcb2d68b66ccbcb74b3ab3
           │   └── diffusion_pytorch_model.bin -> ../../../blobs/c7da0e21ba7ea50637bee26e81c220844defdf01aafca02b2c42ecd
           └── vae
               ├── config.json -> ../../../blobs/55d78924fee13e4220f24320127c5f16284e13b9
               └── diffusion_pytorch_model.bin -> ../../../blobs/1b134cded8eb78b184aefb8805b6b572f36fa77b255c483665dda93
   ```

2. 加载模型下目录下的`model_index.json`到`config_dict`;

   其实从这里就可以看出来，在训练时，scheduler、unet、text_encoder、tokenizer、vae都已经确定了。但实际推理阶段，我们其实可以更换scheduler（采样器）。

   ```json
   {
     "_class_name": "StableDiffusionPipeline",
     "_diffusers_version": "0.6.0",
     "feature_extractor": [
       "transformers",
       "CLIPImageProcessor"
     ],
     "safety_checker": [
       "stable_diffusion",
       "StableDiffusionSafetyChecker"
     ],
     "scheduler": [
       "diffusers",
       "PNDMScheduler"
     ],
     "text_encoder": [
       "transformers",
       "CLIPTextModel"
     ],
     "tokenizer": [
       "transformers",
       "CLIPTokenizer"
     ],
     "unet": [
       "diffusers",
       "UNet2DConditionModel"
     ],
     "vae": [
       "diffusers",
       "AutoencoderKL"
     ]
   }
   ```


3. 根据`config_dict["_class_name"]`加载对应的pipeline class;
4. 根据`model_index.json`计算初始化管线需要的module；  init_dict, unused_kwargs, _ = pipeline_class.extract_init_dict(config_dict, **kwargs)
5. 加载 pipeline 需要的 model, 作为初始化 pipeline 的参数init_kwargs；
6. 初始化 pipeline，即调用`StableDiffusionPipeline.__init`__
#### from_cpkt

支持加载safetensors文件

### 执行推理

```python
    def __call__(
        self,
        prompt: Union[str, List[str]] = None,
        height: Optional[int] = None,
        width: Optional[int] = None,
        num_inference_steps: int = 50,
        guidance_scale: float = 7.5,
        negative_prompt: Optional[Union[str, List[str]]] = None,
        num_images_per_prompt: Optional[int] = 1,
        eta: float = 0.0,
        generator: Optional[Union[torch.Generator, List[torch.Generator]]] = None,
        latents: Optional[torch.FloatTensor] = None,
        prompt_embeds: Optional[torch.FloatTensor] = None,
        negative_prompt_embeds: Optional[torch.FloatTensor] = None,
        output_type: Optional[str] = "pil",
        return_dict: bool = True,
        callback: Optional[Callable[[int, int, torch.FloatTensor], None]] = None,
        callback_steps: int = 1,
        cross_attention_kwargs: Optional[Dict[str, Any]] = None,
    )
```

1. 参数校验、初始化 height、width、batch_size、prompt_embeds、timesteps、latents

2. 循环推理\降噪

   - 将图像噪声输入 unet、unet 预测噪声残差

   - 将图像噪声、噪声残差输入Scheduler(即采样器)生成新的图像噪声
   - 循环第一步、将新的图像噪声输入 unet； 一直循环num_inference_steps次

   ```python
           # 7. Denoising loop
           num_warmup_steps = len(timesteps) - num_inference_steps * self.scheduler.order
           with self.progress_bar(total=num_inference_steps) as progress_bar:
               for i, t in enumerate(timesteps):
                   # expand the latents if we are doing classifier free guidance
                   latent_model_input = torch.cat([latents] * 2) if do_classifier_free_guidance else latents
                   latent_model_input = self.scheduler.scale_model_input(latent_model_input, t)
   
                   # predict the noise residual
                   noise_pred = self.unet(
                       latent_model_input,
                       t,
                       encoder_hidden_states=prompt_embeds,
                       cross_attention_kwargs=cross_attention_kwargs,
                   ).sample
   
                   # perform guidance
                   if do_classifier_free_guidance:
                       noise_pred_uncond, noise_pred_text = noise_pred.chunk(2)
                       noise_pred = noise_pred_uncond + guidance_scale * (noise_pred_text - noise_pred_uncond)
   
                   # compute the previous noisy sample x_t -> x_t-1
                   latents = self.scheduler.step(noise_pred, t, latents, **extra_step_kwargs).prev_sample
   
                   # call the callback, if provided
                   if i == len(timesteps) - 1 or ((i + 1) > num_warmup_steps and (i + 1) % self.scheduler.order == 0):
                       progress_bar.update()
                       if callback is not None and i % callback_steps == 0:
                           callback(i, t, latents)
   ```

3. vae decode、safety checker (安全检测、比如鉴黄)

## 如何生成与[stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui)相同的结果
以下case中使用的模型与参数

- SD Model: [majicmix-realistic](https://civitai.com/models/43331/majicmix-realistic) v6

- LoRA Model: [MoXin](https://civitai.com/models/12597) v1

- 采样器：Euler A

### 基本功能

针对于简单的提示词，和sd webui生成的图是完全一致的。

```python
import torch
from torch import autocast
from diffusers import EulerAncestralDiscreteScheduler,StableDiffusionPipeline
from transformers import CLIPImageProcessor, CLIPProcessor, CLIPTextModel,CLIPTokenizer
from compel import Compel
from diffusers.utils import get_class_from_dynamic_module

pipeline = StableDiffusionPipeline.from_ckpt(
    "/data/stable-diffusion-webui/models/Stable-diffusion/majicmixRealistic_v6.safetensors"
).to("cuda")
pipeline.scheduler = EulerAncestralDiscreteScheduler.from_config(pipeline.scheduler.config)
pipeline.safety_checker = None // 默认会启用safety_checker
prompt0 = "1girl"

generator = torch.Generator(device="cuda").manual_seed(1432216924)


with autocast("cuda"):
    images = pipeline(
        prompt0,
        width = 512,
        height = 512,
        num_images_per_prompt = 1,
        generator=generator,
        num_inference_steps=30,
        guidance_scale=7).images
    

images[0]
```

<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-07-13-053520.png" alt="output" style="zoom: 67%;" />

### 复杂一些的提示词

测试中主要发现以下两个问题会导致出图不一致：

- diffusers和[stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui)提示词权重语法不一致; 

  [diffusers weighted_prompts](https://huggingface.co/docs/diffusers/main/en/using-diffusers/weighted_prompts) 

  [sd webui attention emphasis](https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Features#attentionemphasis)

- diffusers有最大 77提示词token的限制；

  > CLIP (Contrastive Language-Image Pretraining) 模型的输入有 75 个 token 的限制，主要是由于在训练 CLIP 模型时，设计者选择了一个固定的最大序列长度。这个限制有助于在计算方面保持效率，并限制模型大小和内存使用。
  >
  > 在自然语言处理任务中，输入数据通常具有不同的长度和变化。为了简化计算和提高效率，研究人员通常会设定一个最大序列长度，使得模型在内存和计算机能力范围内高效运行。对于 CLIP 模型，研究者设定了 75 个 token 的最大长度。因此，在使用 CLIP 处理文本数据时，需要将输入截断或填充为 75 个 token。
  >
  > 这个限制并不意味着 CLIP 模型只能处理由 75 个 token 构成的数据。在实践中，您可以将较长的输入分割成多个短段，并分别处理，但这需要额外的预处理步骤。在某些情况下，可能需要为超出限制的输入部分选择更合适的截断策略（如确保保留语法结构和主要意义）。
  >
  > 总之，CLIP 模型具有 75 个 token 的限制是为了在训练和应用阶段保持内存和计算效率。这种限制在实际应用中可能需要某些额外的预处理和适应步骤。然而，通过选择合适的截断或填充策略，CLIP 模型仍然可以广泛应用于各种自然语言和计算机视觉任务。     --- GPT4生成内容

社区版本的Stable Diffusion实现了与sd webui一样的功能。[Long Prompt Weighting Stable Diffusion community pipeline](https://github.com/huggingface/diffusers/blob/main/examples/community/lpw_stable_diffusion.py)

至于为什么不直接加到StableDiffusionPipeline，我理解原因有两个：

1. diffusers创建pipeline很简单，鼓励用户创建自己的pipeline；

2. 提示词权重、token数限制非diffusers的核心功能，且没必要和sd webui保持一致。


以下代码为使用diffusers [Long Prompt Weighting Stable Diffusion community pipeline](https://github.com/huggingface/diffusers/blob/main/examples/community/lpw_stable_diffusion.py)示例,结果与sd webui完全一致：

```python
import torch
from torch import autocast
from diffusers import EulerAncestralDiscreteScheduler,StableDiffusionPipeline
from transformers import CLIPImageProcessor, CLIPProcessor, CLIPTextModel,CLIPTokenizer
from diffusers.utils import get_class_from_dynamic_module


stablepipeline = StableDiffusionPipeline.from_ckpt(
    "/data/stable-diffusion-webui/models/Stable-diffusion/majicmixRealistic_v6.safetensors",
).to("cuda")

LPWStableDiffusionPipeline = get_class_from_dynamic_module("lpw_stable_diffusion", module_file="lpw_stable_diffusion.py")
pipeline = LPWStableDiffusionPipeline(**stablepipeline.components)
pipeline.scheduler = EulerAncestralDiscreteScheduler.from_config(pipeline.scheduler.config)
pipeline.safety_checker = None

pipeline.to("cuda")
prompt = "((wallpaper 8k)), ((high detailed)), ((masterpiece)), ((best quality:1.2)), ((hdr)), ((absurdres)), ((RAW photo)), 1girl, green eyes, medium hair, bangs, portrait, upper body, shoulder, shirt, simple background, blue background"
negative_prompt = "easynegative,ng_deepnegative_v1_75t, badhandv4,(worst quality:2),(low quality:2),(normal quality:2),lowres,bad anatomy,bad hands,normal quality,((monochrome)),((grayscale)),((watermark)),"

generator = torch.Generator(device="cuda").manual_seed(1432216924)

with autocast("cuda"):
    images = pipeline(
        prompt,
        width = 512,
        height = 512,
        num_images_per_prompt = 1,
        generator=generator,
        negative_prompt = negative_prompt,
        num_inference_steps=30,
        guidance_scale=7).images
images[0]
```

<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-07-13-053525.png" alt="00006-1432216924" style="zoom: 67%;" />

### LoRA支持

#### 单LoRA支持

目前已发布版本(diffusers [v0.18.1](https://github.com/huggingface/diffusers/releases/tag/v0.18.1))仅支持attention layers， 效果不太好。已经有人提交pr [Support to load Kohya-ss style LoRA file format (without restrictions)](https://github.com/huggingface/diffusers/pull/3756),尚未合并。

当然除此之外还有其他一些问题，比如[Unload lora weights after they have been loaded](https://github.com/huggingface/diffusers/issues/4027)

> Currently, LoRA is only supported for the attention layers of the `UNet2DConditionalModel`. We also support fine-tuning the text encoder for DreamBooth with LoRA in a limited capacity. Fine-tuning the text encoder for DreamBooth generally yields better results, but it can increase compute usage.    [Low-Rank Adaptation of Large Language Models (LoRA)](https://huggingface.co/docs/diffusers/training/lora)

>Usually, LoRA fine-tuning of the text encoder along with the UNet leads to better results than LoRA fine-tuning the UNet alone. A reference PR that might be relevant here: [#3180](https://github.com/huggingface/diffusers/pull/3180).    [Docs for LoRA are confusing](https://github.com/huggingface/diffusers/issues/3219)

> Discussed in [#3064 (comment)](https://github.com/huggingface/diffusers/issues/3064#issuecomment-1544159510), PR [#3437](https://github.com/huggingface/diffusers/pull/3437). In the PR [#3437](https://github.com/huggingface/diffusers/pull/3437), we implemented the loading of Kohya-ss style LoRA, but this was only support for the Attention part, which had already been implemented in Diffusers. Kohya-ss style LoRA is not only extended to Attention, but also to `ClipMLP`, `Transformer2DModel`'s `proj_in`, `proj_out`, and so on. This PR will support these remaining issues.
>
> [Support to load Kohya-ss style LoRA file format (without restrictions)](https://github.com/huggingface/diffusers/pull/3756)

> [failed to use the feature of supporting for A1111 LoRA](https://github.com/huggingface/diffusers/issues/3725)

```python
import torch
from torch import autocast
from diffusers import EulerAncestralDiscreteScheduler,StableDiffusionPipeline
from transformers import CLIPImageProcessor, CLIPProcessor, CLIPTextModel,CLIPTokenizer
from compel import Compel
from diffusers.utils import get_class_from_dynamic_module


stablepipeline = StableDiffusionPipeline.from_ckpt(
    "/data/stable-diffusion-webui/models/Stable-diffusion/majicmixRealistic_v6.safetensors"
).to("cuda")
LPWStableDiffusionPipeline = get_class_from_dynamic_module("lpw_stable_diffusion", module_file="lpw_stable_diffusion.py")
pipeline = LPWStableDiffusionPipeline(**stablepipeline.components)

pipeline.scheduler = EulerAncestralDiscreteScheduler.from_config(pipeline.scheduler.config)
pipeline.feature_extractor = CLIPImageProcessor.from_dict(pipeline.feature_extractor.to_dict())
pipeline.safety_checker = None
pipeline.load_lora_weights("/data/stable-diffusion-webui/models/Lora", weight_name="moxinv1.khzz.safetensors")

prompt = "1girl"
generator = torch.Generator(device="cuda").manual_seed(1432216924)


with autocast("cuda"):
    images = pipeline(
        prompt0,
        width = 512,
        height = 512,
        num_images_per_prompt = 1,
        generator=generator,
        num_inference_steps=30,
        cross_attention_kwargs={"scale": 1},
        guidance_scale=7).images
    

images[0]
```

<center>
<figure>
<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-07-13-030013.png" alt="before-face-store-03" style="zoom:67%;" />
<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-07-13-025959.png" alt="00018-1432216924" style="zoom:67%;" />
</figure>
</center>
<center>左图为diffusers[v0.18.1]的效果，右图为应用pr之后的效果(与sd webui完全一致)</center>

#### 多LoRA支持

尚未支持、[Support for adding multiple LoRA layers to Diffusers](https://github.com/huggingface/diffusers/issues/2613#top)

### 面部修复 - CodeFormer

[CodeFormer repo](https://github.com/sczhou/CodeFormer)

```python
# Clone CodeFormer and enter the CodeFormer folder
!git clone https://github.com/sczhou/CodeFormer.git
%cd  CodeFormer

# Set up the environment
# Install python dependencies
!pip install -r requirements.txt
# Install basicsr
!python basicsr/setup.py develop

# Download the pre-trained model
!python scripts/download_pretrained_models.py facelib
!python scripts/download_pretrained_models.py CodeFormer
```

将需要修复的图像放入inputs目录，执行命令后得到修复后的图片

```python
# Inference the uploaded images
#@markdown `CODEFORMER_FIDELITY`: Balance the quality (lower number) and fidelity (higher number)<br>
# you can add '--bg_upsampler realesrgan' to enhance the background
CODEFORMER_FIDELITY = 0.7 #@param {type:"slider", min:0, max:1, step:0.01}
#@markdown `BACKGROUND_ENHANCE`: Enhance background image with Real-ESRGAN<br>
BACKGROUND_ENHANCE = True #@param {type:"boolean"}
#@markdown `FACE_UPSAMPLE`: Upsample restored faces for high-resolution AI-created images<br>
FACE_UPSAMPLE = False #@param {type:"boolean"}
if BACKGROUND_ENHANCE:
  if FACE_UPSAMPLE:
    !python inference_codeformer.py -w $CODEFORMER_FIDELITY --input_path inputs --bg_upsampler realesrgan --face_upsample
  else:
    !python inference_codeformer.py -w $CODEFORMER_FIDELITY --input_path inputs --bg_upsampler realesrgan
else:
  !python inference_codeformer.py -w $CODEFORMER_FIDELITY --input_path inputs
```


### Hires.fix
diffusers中的高清修复并不像sd webui那样唾手可得，需要我们了解每个步骤的原理，然后组合使用。

[does diffusers have the equivalent to hires fix from A1111?](https://github.com/huggingface/diffusers/issues/3429)

[Real-ESRGAN](https://github.com/xinntao/Real-ESRGAN)

#### sd webui是如何实现Hires.fix的?

txt2img的核心逻辑位于

- 文件: processing.py

- 类:  `StableDiffusionProcessingTxt2Img(StableDiffusionProcessing)` 

- 函数: `sample(self, conditioning, unconditional_conditioning, seeds, subseeds, subseed_strength, prompts`

![image-20230712152457898](http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-07-13-054317.png)

如代码所示，其实text2img + Hires.fix其实就是text2img + upscaler + img2img。

其中特别注意Upscaler为latent时，是将图像通过`torch.nn.functional.interpolate`处理获得samples；

> `torch.nn.functional.interpolate` 是一个 PyTorch 函数，用于调整（放大或缩小）张量（通常是图像或特征映射）的尺寸。这个函数在计算机视觉和深度学习任务中很常见，尤其是在需要调整图像或特征映射大小以适应模型输入要求、进行上采样或下采样操作时。   
>
> ---GPT4生成内容

而其他情况则是调用对应的模型预测得到对应图像，然后vae encode获得samples。Upscaler阶段产生的samples为img2img阶段的输入；详细代码如下：

![image-20230712210733311](http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-07-13-054302.png)

其中upscaler用到了[Real-ESRGAN](https://github.com/xinntao/Real-ESRGAN),模型列表如下：

```python
def get_realesrgan_models(scaler):
    try:
        from basicsr.archs.rrdbnet_arch import RRDBNet
        from realesrgan.archs.srvgg_arch import SRVGGNetCompact
        models = [
            UpscalerData(
                name="R-ESRGAN General 4xV3",
                path="https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.5.0/realesr-general-x4v3.pth",
                scale=4,
                upscaler=scaler,
                model=lambda: SRVGGNetCompact(num_in_ch=3, num_out_ch=3, num_feat=64, num_conv=32, upscale=4, act_type='prelu')
            ),
            UpscalerData(
                name="R-ESRGAN General WDN 4xV3",
                path="https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.5.0/realesr-general-wdn-x4v3.pth",
                scale=4,
                upscaler=scaler,
                model=lambda: SRVGGNetCompact(num_in_ch=3, num_out_ch=3, num_feat=64, num_conv=32, upscale=4, act_type='prelu')
            ),
            UpscalerData(
                name="R-ESRGAN AnimeVideo",
                path="https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.5.0/realesr-animevideov3.pth",
                scale=4,
                upscaler=scaler,
                model=lambda: SRVGGNetCompact(num_in_ch=3, num_out_ch=3, num_feat=64, num_conv=16, upscale=4, act_type='prelu')
            ),
            UpscalerData(
                name="R-ESRGAN 4x+",
                path="https://github.com/xinntao/Real-ESRGAN/releases/download/v0.1.0/RealESRGAN_x4plus.pth",
                scale=4,
                upscaler=scaler,
                model=lambda: RRDBNet(num_in_ch=3, num_out_ch=3, num_feat=64, num_block=23, num_grow_ch=32, scale=4)
            ),
            UpscalerData(
                name="R-ESRGAN 4x+ Anime6B",
                path="https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.2.4/RealESRGAN_x4plus_anime_6B.pth",
                scale=4,
                upscaler=scaler,
                model=lambda: RRDBNet(num_in_ch=3, num_out_ch=3, num_feat=64, num_block=6, num_grow_ch=32, scale=4)
            ),
            UpscalerData(
                name="R-ESRGAN 2x+",
                path="https://github.com/xinntao/Real-ESRGAN/releases/download/v0.2.1/RealESRGAN_x2plus.pth",
                scale=2,
                upscaler=scaler,
                model=lambda: RRDBNet(num_in_ch=3, num_out_ch=3, num_feat=64, num_block=23, num_grow_ch=32, scale=2)
            ),
        ]
        return models
    except Exception:
        print("Error making Real-ESRGAN models list:", file=sys.stderr)
        print(traceback.format_exc(), file=sys.stderr)
```

#### diffusers中应该如何实现

其实很简单，跟sd webui一样，把文生图的结果upscaler然后放到图生图再跑一下就好了。

```python
import torch
from torch import autocast
from diffusers import EulerAncestralDiscreteScheduler,StableDiffusionPipeline,StableDiffusionImg2ImgPipeline
from transformers import CLIPImageProcessor, CLIPProcessor, CLIPTextModel,CLIPTokenizer
from diffusers.utils import get_class_from_dynamic_module


stablepipeline = StableDiffusionPipeline.from_ckpt(
    "/data/stable-diffusion-webui/models/Stable-diffusion/majicmixRealistic_v6.safetensors",
).to("cuda")

LPWStableDiffusionPipeline = get_class_from_dynamic_module("lpw_stable_diffusion", module_file="lpw_stable_diffusion.py")
text2img = LPWStableDiffusionPipeline(**stablepipeline.components)
text2img.scheduler = EulerAncestralDiscreteScheduler.from_config(text2img.scheduler.config)
text2img.safety_checker = None

text2img.to("cuda")
prompt = "1girl"

generator = torch.Generator(device="cuda").manual_seed(1432216924)

img2img = StableDiffusionImg2ImgPipeline(**text2img.components)

with autocast("cuda"):
    images = text2img(
        prompt,
        width = 512,
        height = 512,
        num_images_per_prompt = 1,
        generator=generator,
        #negative_prompt = negative_prompt,
        num_inference_steps=20,
        guidance_scale=7).images

# 这里偷懒了,只是把图片拉伸了，其实完全可抄一下sd webui的代码
images[0] = images[0].resize((512*2, 512*2))

with autocast("cuda"):
        images = img2img(
                prompt="1girl",
                image=images[0],
                strength=0.7,
                num_inference_steps=20,
                guidance_scale=7,
                num_images_per_prompt=1,
                generator=generator
        ).images

images[0]
```

### 相对于sd webui的优缺点

代码结构简单、清晰、二次开发简单、支持[多卡分布式推理](https://huggingface.co/docs/diffusers/v0.18.2/en/training/distributed_inference#accelerate)

[sd webui多显卡支持情况](https://github.com/AUTOMATIC1111/stable-diffusion-webui/discussions/1621)

不如sd webui 功能齐全、很多功能需要自行组合，非开箱即用 比如面部修复、Hires.fix等

如果要用diffusers复现sd webui的功能肯定是可行的，不过要有些工作量。好的方面是目前diffusers社区非常活跃、更新速度很快。

## 写在最后

以上算是对于stable diffusion以及diffusers一个短期的学习总结，由于本人是非算法背景，很多地方还在学习中，难免有很多不正确的地方，有任何问题欢迎随时指出。

另外本文有很多地方没有提到，比如controlnet、unclip、部署、训练等，有需要或者有精力再填坑。


## 附1 stable diffusion webui && diffusers安装

```bash
yum install -y git python3-opencv
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b

mkdir /data
mount /dev/vdb /data

conda create -n sd-webui python=3.10
conda activate sd-webui
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui.git
cd stable-diffusion-webui
sed -i 's/can_run_as_root=0/can_run_as_root=1/g' webui.sh && ./webui.sh 

conda create -n py3.8 python==3.8
conda activate py3.8
conda update -y conda
conda list
conda install -y numpy matplotlib pandas 
conda install pytorch torchvision torchaudio pytorch-cuda=11.7 -c pytorch -c nvidia
conda install -c huggingface transformers diffusers
conda install -y jupyter notebook requests
```

https://pytorch.org/

https://raw.githubusercontent.com/CompVis/stable-diffusion/main/configs/stable-diffusion/v1-inference.yaml

## 附2 开发环境

以上实例代码都是在jupter notebook中可运行的。如以其他方式运行，需要做些变更。

## 附3 一些参考链接

- [不知道Stable Diffusion Web UI应该怎么选？ 看这一篇就够了！](https://afoo.me/posts/2023-04-29-stablediffusion-webui-alternatives.html)
- https://huggingface.co/blog/controlnet
- https://pytorch.org/blog/accelerated-diffusers-pt-20/
- https://huggingface.co/docs/diffusers/v0.18.2/en/training/distributed_inference#accelerate
- [https://huggingface.co/blog/stable_diffusion#how-does-stable-diffusion-work](https://huggingface.co/blog/stable_diffusion#how-does-stable-diffusion-work)