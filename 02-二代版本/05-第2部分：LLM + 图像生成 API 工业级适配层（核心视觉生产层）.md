**第2部分：LLM + 图像生成 API 工业级适配层（核心视觉生产层）** 

这一层是整个“短剧生产系统”的关键瓶颈层，因为它直接决定：

> 🎯 角色是否稳定  
> 🎯 画风是否统一  
> 🎯 分镜是否可控  
> 🎯 是否能批量工业化生产

---

# 🧠 一、设计目标（这一层的本质）

我们要解决 6 件事：

### 1️⃣ 多厂商统一接口（Kling / Runway / Flux / DALL·E / Midjourney）

---

### 2️⃣ 同时支持：

- 文生图
    
- 图生图
    
- 图生视频
    
- 视频续写
    

---

### 3️⃣ 统一“短剧参数体系”

- 9:16
    
- seed
    
- reference image
    
- motion strength
    
- duration
    

---

### 4️⃣ 同步 + 异步统一返回结构

---

### 5️⃣ 角色一致性接口预埋（为第4部分 LoRA / FaceSwap 做接口准备）

---

### 6️⃣ 失败自动降级链路（关键）

---

# 🏗 二、视觉 API 统一抽象层

## base_visual_adapter.py

```python
import abc
import asyncio
from typing import Dict, Any, Optional


class BaseVisualAdapter(abc.ABC):
    """
    图像/视频生成统一抽象层
    """

    def __init__(self, config: Dict):
        self.config = config
        self.max_retries = config.get("max_retries", 3)

    # =========================
    # 核心统一接口
    # =========================

    @abc.abstractmethod
    async def text_to_image(self, prompt: str, **kwargs) -> Dict:
        pass

    @abc.abstractmethod
    async def image_to_image(self, prompt: str, image_url: str, **kwargs) -> Dict:
        pass

    @abc.abstractmethod
    async def image_to_video(self, prompt: str, image_url: str, **kwargs) -> Dict:
        pass

    @abc.abstractmethod
    async def text_to_video(self, prompt: str, **kwargs) -> Dict:
        pass

    # =========================
    # 异步任务统一轮询
    # =========================

    async def poll_task(self, task_id: str, interval=5, timeout=600):
        """
        统一轮询逻辑（视频生成核心）
        """

        elapsed = 0

        while elapsed < timeout:
            result = await self.check_status(task_id)

            if result.get("status") == "completed":
                return await self.fetch_result(task_id)

            if result.get("status") == "failed":
                raise Exception("task failed")

            await asyncio.sleep(interval)
            elapsed += interval

        raise TimeoutError("visual generation timeout")

    @abc.abstractmethod
    async def check_status(self, task_id: str) -> Dict:
        pass

    @abc.abstractmethod
    async def fetch_result(self, task_id: str) -> Dict:
        pass
```

---

# 🎨 三、Kling（可灵）适配器（重点工业级）

```python
import aiohttp
from base_visual_adapter import BaseVisualAdapter


class KlingAdapter(BaseVisualAdapter):

    def __init__(self, config):
        super().__init__(config)
        self.api_key = config["api_key"]
        self.base_url = "https://api.kling.ai/v1"

    # =========================
    # 文生图
    # =========================

    async def text_to_image(self, prompt: str, **kwargs):

        payload = {
            "prompt": prompt,
            "aspect_ratio": "9:16",
            "seed": kwargs.get("seed", 42),
            "reference_image": kwargs.get("ref_image"),
        }

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.base_url}/image/generate",
                json=payload,
                headers={"Authorization": self.api_key}
            ) as resp:

                data = await resp.json()
                return await self._handle_async(data)

    # =========================
    # 图生视频（核心）
    # =========================

    async def image_to_video(self, prompt: str, image_url: str, **kwargs):

        payload = {
            "prompt": prompt,
            "image": image_url,
            "duration": kwargs.get("duration", 4),
            "motion_strength": kwargs.get("motion", "medium"),
            "aspect_ratio": "9:16"
        }

        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.base_url}/video/generate",
                json=payload,
                headers={"Authorization": self.api_key}
            ) as resp:

                data = await resp.json()
                task_id = data["task_id"]

                return await self.poll_task(task_id)

    async def check_status(self, task_id):
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.base_url}/task/{task_id}",
                headers={"Authorization": self.api_key}
            ) as resp:
                return await resp.json()

    async def fetch_result(self, task_id):
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.base_url}/task/{task_id}/result",
                headers={"Authorization": self.api_key}
            ) as resp:
                return await resp.json()

    async def _handle_async(self, data):
        if "task_id" in data:
            return await self.poll_task(data["task_id"])
        return data
```

---

# 🎬 四、Runway / Luma / Seedance（统一异步模型）

```python
class AsyncVideoAdapter(BaseVisualAdapter):

    async def text_to_video(self, prompt: str, **kwargs):

        payload = {
            "prompt": prompt,
            "duration": kwargs.get("duration", 4),
            "ratio": "9:16"
        }

        task_id = await self._submit(payload)

        return await self.poll_task(task_id)

    async def _submit(self, payload):
        raise NotImplementedError
```

👉 Runway / Luma / Seedance 都继承这个

---

# 🖼 五、Flux / DALL·E / Midjourney（图像层统一）

```python
class ImageAdapter(BaseVisualAdapter):

    async def text_to_image(self, prompt: str, **kwargs):

        payload = {
            "prompt": prompt,
            "size": "1080x1920",
            "seed": kwargs.get("seed")
        }

        return await self._request(payload)

    async def image_to_image(self, prompt, image_url, **kwargs):

        payload = {
            "prompt": prompt,
            "image": image_url,
            "strength": kwargs.get("strength", 0.7)
        }

        return await self._request(payload)
```

---

# 🧩 六、统一视觉调度器（核心）

```python
class VisualGateway:

    def __init__(self):
        self.adapters = {}

    def register(self, name: str, adapter):
        self.adapters[name] = adapter

    async def text_to_image(self, provider, prompt, **kwargs):
        return await self.adapters[provider].text_to_image(prompt, **kwargs)

    async def image_to_video(self, provider, prompt, image_url, **kwargs):
        return await self.adapters[provider].image_to_video(
            prompt, image_url, **kwargs
        )
```

---

# 🔁 七、自动降级链（非常关键）

```python
FALLBACK_CHAIN = {
    "kling": ["runway", "luma", "seedance"],
    "midjourney": ["dalle3", "flux"],
}
```

```python
async def safe_generate(gateway, provider, func, *args, **kwargs):

    try:
        return await getattr(gateway.adapters[provider], func)(*args, **kwargs)

    except Exception as e:

        for fallback in FALLBACK_CHAIN.get(provider, []):
            try:
                return await getattr(gateway.adapters[fallback], func)(*args, **kwargs)
            except:
                continue

        raise e
```

---

# 🧠 八、短剧“统一参数体系”（关键设计）

所有模型统一输入：

```python
VISUAL_SCHEMA = {
    "aspect_ratio": "9:16",
    "resolution": "1080x1920",
    "seed": 42,
    "motion_strength": "low|medium|high",
    "duration": 4,
    "reference_image": None,
    "lora_id": None
}
```

👉 这是整个系统“标准化生产能力”的关键

---

# 🎯 九、这一部分已经实现了什么

现在你已经拥有：

### ✔ 多厂商视觉统一接口

### ✔ 文生图 / 图生视频 / 图生图统一

### ✔ 异步任务系统

### ✔ 轮询机制

### ✔ fallback降级链

### ✔ 短剧参数标准化体系

---

# 🚀 下一步（第3部分）

下一部分我会给你：

## 👉 视频 + TTS + 音乐 API 工业级系统

包括：

- ElevenLabs / CosyVoice / Edge-TTS
    
- Suno / Udio / MusicGen
    
- SSML 情绪控制系统
    
- 角色 voice binding（角色音色绑定系统）
    
- 音画同步结构设计
    