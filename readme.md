# 项目文档

## 项目概述
本项目是一个基于 Flask 框架的 Web 应用程序，主要用于视频分析和处理。项目实现了视频上传、视频摘要生成、视频行为分析等功能，并将分析结果存储在 Milvus 数据库中，以便后续检索和查询。

## 文档
- [API 文档](docs/api.md)：详细的 API 接口说明
- [数据流图](docs/data_flow.md)：系统组件和数据流向说明

## 功能模块

### 1. 视频上传与处理
- **视频上传**：用户可以通过 API 上传视频文件，并将其存储在 MinIO 对象存储中。
- **视频处理**：上传的视频文件会被提取帧并转换为 Base64 编码，用于后续的分析和摘要生成。

### 2. 视频摘要生成
- **摘要生成**：通过调用 OpenAI 的 API，对视频内容进行分析并生成摘要。生成的摘要信息包括视频中的关键行为和时间范围。

### 3. 视频行为分析
- **行为分析**：通过分析视频帧，识别视频中的常见驾驶行为和其他交通参与者的行为，并将分析结果以 JSON 格式输出。

### 4. 数据库管理
- **Milvus 数据库**：使用 Milvus 数据库存储视频的元数据和分析结果，支持通过标签或文本进行视频检索。

## 技术栈
- **Flask**：Web 框架用于构建 API 和处理请求。
- **MinIO**：对象存储服务，用于存储视频文件。
- **OpenAI**：用于视频内容分析和摘要生成。
- **Milvus**：向量数据库，用于存储和检索视频分析结果。

## 安装与运行

### 1. 环境准备
确保已安装以下依赖：
- Python 3.8+
- Flask
- MinIO
- OpenAI
- Milvus

### 2. 下载模型
项目需要以下模型文件，请下载并放置在对应目录：
```
models/                      # 模型文件目录
├── embedding/              # 向量嵌入模型
│   ├── cn-clip/           # 中文CLIP模型
│   └── bge-small-zh-1.5/  # BGE中文向量模型
```
注意：模型文件较大，不包含在代码仓库中，请从以下地址下载：

- BGE 中文向量模型：[BAAI/bge-small-zh-v1.5](https://huggingface.co/BAAI/bge-small-zh-v1.5)
  - 下载后放置在 `models/embedding/bge-small-zh-1.5/` 目录
- 中文 CLIP 模型：[OFA-Sys/chinese-clip-vit-large-patch14-336px](https://huggingface.co/OFA-Sys/chinese-clip-vit-large-patch14-336px)
  - 下载模型文件 `clip_cn_vit-l-14-336.pt`
  - 下载后放置在 `models/embedding/cn-clip/` 目录

### 3. 安装依赖
```bash
conda create -y -p ./venv python=3.12
conda activate ./venv
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 4. 配置环境变量
1. 复制环境变量示例文件：
```bash
cp .env_sample .env
```

2. 编辑 `.env` 文件，填写必要的环境变量：
```ini
# 服务器配置
SERVER_HOST=localhost        # 服务器主机地址
SERVER_PORT=30501           # 服务器端口

# API密钥
DASHSCOPE_API_KEY=your_api_key    # DashScope API密钥

# MinIO配置
OSS_BUCKET_NAME=your_bucket_name   # MinIO存储桶名称

# 场景挖掘算法配置
SCENE_MINING_CONFIG_PATH=app/algorithm/scene_mining/config-qwen-gemini.yaml
SCENE_MINING_OUTPUT_DIR=outputs/scene_mining
SCENE_MINING_VIDEO_CACHE_DIR=data/scene_mining_videos
SCENE_MINING_VIDEO_URL_PREFIX=file:///app/videos
SCENE_MINING_SUMMARY_MAX_TOKENS=4096

# Qwen3-VL-Embedding vLLM 服务与检索特征配置
EMBEDDING_MODEL=qwen3-vl
QWEN3_VL_EMBEDDING_BASE_URL=http://localhost:8575
QWEN3_VL_EMBEDDING_DIM=2048
QWEN3_VL_EMBEDDING_TIMEOUT=300
QWEN3_VL_EMBEDDING_RETRIES=2
QWEN3_VL_EMBEDDING_RETRY_BACKOFF=1
QWEN3_VL_GPU_MEMORY_UTILIZATION=0.1727
QWEN3_VL_MAX_MODEL_LEN=32768
MILVUS_VIDEO_TEXT_FEATURE_COLLECTION_NAME=video_text_features
MILVUS_VIDEO_VISUAL_FEATURE_COLLECTION_NAME=video_visual_features
FRAME_SAMPLE_FPS=1
FRAME_SAMPLE_MAX_FRAMES=20
FRAME_SAMPLE_EVENT_FPS=1
MEDIA_CACHE_DIR=data/media_cache
MEDIA_CACHE_MAX_AGE=3600
TASK_STATUS_DIR=data/task_status
ENABLE_LEGACY_VIDEO_PROXY=false

# 场景挖掘 VLM 服务
SCENE_MINING_API_BASE_URL=http://localhost:8576/v1
SCENE_MINING_API_MODEL_NAME=/models/qwen-gemini
SCENE_MINING_VLM_PORT=8576
SCENE_MINING_VLM_GPU_MEMORY_UTILIZATION=0.7773
SCENE_MINING_VLM_MAX_MODEL_LEN=40960
```

注意：请将示例值替换为实际的配置值。

场景挖掘默认通过当前项目内置的 `app/algorithm/scene_mining/config-qwen-gemini.yaml` 连接当前算法端 `http://localhost:8574/v1`，模型名为 `/models/qwen-gemini`。启动前请确保算法端已可访问；若待分析视频不是默认 `/mnt/data/ai-ground/dataset/videos` 下的文件，服务会缓存到 `SCENE_MINING_VIDEO_CACHE_DIR` 后再送入算法流程。

### 5. 启动应用
```bash
python app.py
```

### Docker + vLLM 启动

`docker-compose.yml` 会启动三个服务：

- `qwen-gemini-vlm`：基于 `swr.cn-north-4.myhuaweicloud.com/ddn-k8s/docker.io/vllm/vllm-openai:v0.21.0-ubuntu2404` 挂载 `/mnt/data/checkpoints/Qwen3_5-9B-Gemini-Distill`，在 compose 内网提供 OpenAI-compatible `/v1` VLM 服务，并仅绑定宿主 `127.0.0.1:8576`。
- `qwen3-vl-embedding`：基于同一 vLLM 镜像挂载 `/mnt/data/checkpoints/Qwen3-VL-Embedding-2B`，在 compose 内网提供 `/embed` 特征服务，并仅绑定宿主 `127.0.0.1:8575`。
- `vision-perception`：Flask 应用，只通过 `QWEN3_VL_EMBEDDING_BASE_URL` 调用 embedding 服务，不在应用进程加载模型。

```bash
cp .env_sample .env
./scripts/start_docker_auto_gpu.sh
```

脚本会用 `nvidia-smi` 选择空闲显存最多的 GPU，并设置 `CUDA_VISIBLE_DEVICES` 后执行 `docker compose up --build`。默认挂载视频目录 `/mnt/data/ai-ground/dataset/videos:/app/videos:ro`，运行输出写入 `./outputs`，缓存写入 `./data`。

同卡共部署时，默认按总显存利用率 0.95 且 **VLM:Embedding = 9:2** 分配 vLLM `gpu-memory-utilization`：`SCENE_MINING_VLM_GPU_MEMORY_UTILIZATION=0.7773`，`QWEN3_VL_GPU_MEMORY_UTILIZATION=0.1727`。

由于当前云主机的 EIP 入口对 Docker bridge 端口转发不稳定，`vision-perception` 使用 host network 直接监听 `SERVER_PORT`；VLM 和 embedding 只绑定宿主 loopback，不暴露公网：

```ini
SCENE_MINING_API_BASE_URL=http://127.0.0.1:8576/v1
QWEN3_VL_EMBEDDING_BASE_URL=http://127.0.0.1:8575
```

Qwen3-VL-Embedding 检索会写入两个新增 Milvus collection：`video_text_features` 保存 `tags` 与 `summary` 两条文本特征，`video_visual_features` 保存每个视频一条由 ffmpeg 采样帧池化得到的全局视觉特征。

视频和缩略图对浏览器统一返回 `/media/<bucket>/<object>`。服务会先从 MinIO 拉取到 `MEDIA_CACHE_DIR`，再用 Flask `send_file(conditional=True)` 支持 Range 播放，避免公网代理长连接直连 MinIO 时偶发 502。媒体健康检查：

```bash
curl 'http://127.0.0.1:30501/api/media/health?url=/media/perception-mining/example.mp4'
curl -X POST http://127.0.0.1:30501/api/media/health \
  -H 'Content-Type: application/json' \
  -d '{"urls":["/media/perception-mining/example.mp4"],"warm_cache":true}'
```

长任务处理除 `/api/add/stream` 外，也支持任务化轮询，进度状态写入 `TASK_STATUS_DIR`，浏览器刷新或代理断开后可继续查询：

```bash
curl -X POST http://127.0.0.1:30501/api/add/task \
  -H 'Content-Type: application/json' \
  -d '{"video_url":"/media/perception-mining/example.mp4","action_type":3}'
curl http://127.0.0.1:30501/api/add/task/<task_id>
```

运行期临时文件清理：

```bash
python -m app.scripts.cleanup_runtime --dry-run
python -m app.scripts.cleanup_runtime --max-age-hours 24
```

默认清理媒体本地缓存、任务状态、分片上传残留、上传锁和 scene mining 裁切 clip；`SCENE_MINING_VIDEO_CACHE_DIR` 默认保留，确认不需要复用时可加 `--include-video-cache`。

初始化/补齐 feature collection：

```bash
python -m app.scripts.create_database
python -m app.scripts.backfill_features --limit 10        # 试跑
python -m app.scripts.backfill_features                   # 全量回填
python -m app.scripts.backfill_features --skip-visual     # 仅回填文本特征
```

## API 参考

详细的 API 文档请参考 [API 文档](docs/api.md)。

### 主要接口
- 视频上传：上传视频文件到系统
- 视频添加：添加视频并进行分析
- 视频挖掘：分析视频中的行为
- 摘要生成：生成视频内容摘要
- 视频搜索：基于文本搜索视频


## 开发说明

### 错误处理
项目使用统一的错误处理机制：
1. 参数验证错误通过 ValueError 抛出
2. 所有异常都会被转换为统一的 JSON 响应格式
3. 错误日志会自动记录到日志文件中

### 响应处理
- 使用 @api_handler 装饰器统一处理所有 API 响应
- 成功响应使用 api_response() 函数封装
- 错误响应使用 error_response() 函数封装
