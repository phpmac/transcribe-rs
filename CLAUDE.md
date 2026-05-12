# CLAUDE.md

此文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导.

Read @AGENTS.md

## 工具使用规则

- 禁止使用 bash 命令操作文件 (cat/sed/awk/head/tail/grep/find 等), 必须使用 Read/Edit/Write 工具
- 代码定位优先使用 LSP (定义/引用/符号/大纲)
- GitHub 仓库信息必须使用 gh 命令, 禁止使用 web reader 爬取 GitHub 页面

## 提交规范

- commit message 必须使用中文, 禁止英文
- 遵循约定式提交, 使用中文前缀: 添加(新功能)/修复(bug)/重构(代码)/文档(变更)/优化(性能)/测试(用例)/发布(版本)

## 开发命令

```bash
cargo check --features onnx          # 检查 ONNX 模型编译
cargo check --features whisper-cpp   # 检查 Whisper 编译
cargo check --all-features           # 检查所有 feature

cargo test --features onnx           # 运行 ONNX 模型测试
cargo test --features whisper-cpp    # 运行 Whisper 测试
cargo test --all-features            # 运行所有测试

cargo run --example parakeet --features onnx
cargo run --example qwen3 --features onnx
```

开发别名 (定义在 `.cargo/config.toml`):
- `cargo check-all` -> `cargo check --all-features`
- `cargo build-all` -> `cargo build --all-features`
- `cargo test-all` -> `cargo test --all-features`

测试需要本地模型文件, 模型不存在时测试会跳过.

## 架构

多引擎语音转文字 Rust 库. 所有本地引擎实现 `SpeechModel` trait, 远程引擎实现 `RemoteTranscriptionEngine` trait.

### 核心结构

- `src/lib.rs` - `SpeechModel` trait 定义, `TranscribeOptions`, `ModelCapabilities`
- `src/error.rs` - `TranscribeError` 统一错误类型
- `src/audio.rs` - WAV 读取, 重采样到 16kHz
- `src/accel.rs` - GPU 加速器全局设置 (ORT/Whisper)

### ONNX 引擎 (`src/onnx/`)

所有 ONNX 模型共享 `session.rs` (会话创建), `audio-features/` (mel 频谱/CTC 解码):

| 模块 | 引擎 | 备注 |
|------|------|------|
| `parakeet/` | Parakeet TDT | 支持 timestamp |
| `canary/` | Canary | 支持翻译, PnC, ITN |
| `cohere/` | Cohere Transcribe | 大模型, 高精度 |
| `qwen3/` | Qwen3-ASR | 多语言, 支持语言提示词 |
| `sense_voice/` | SenseVoice | 中英日韩粤 |
| `moonshine/` | Moonshine + Streaming | 英语为主 |
| `gigaam/` | GigaAM v3 | 俄语 |

### Whisper 引擎 (`src/whisper_cpp/`)

通过 whisper-rs 绑定 whisper.cpp, GPU 由 feature flag 控制 (Metal/Vulkan/CUDA).

### 关键设计模式

- `SpeechModel` trait: `transcribe(&samples, &opts)` + `transcribe_file(&path, &opts)` 统一接口
- 每个 ONNX 模型内部结构: `engine.rs` (trait impl) -> `model.rs` (ONNX 推理) -> `mel.rs` (特征提取) -> `tokenizer.rs` (解码)
- GPU 加速通过全局原子变量 (`accel` 模块) 控制, 在加载模型前设置
- `Quantization` enum 控制加载 Int8/Int4/FP16/FP32 模型文件变体

### 音频要求

所有引擎: 16kHz, mono, 16-bit PCM WAV (f32 samples 输入).

### 各模型官方音频时长限制

| 模型 | 官方限制 | 来源 |
|------|---------|------|
| Qwen3-ASR | 20 分钟 | arxiv 技术报告 |
| SenseVoice | 30 秒 (需 VAD 分段) | HuggingFace 模型卡 |
| Canary/Parakeet | ~24 分钟 (全注意力) | HuggingFace nvidia/parakeet-tdt-0.6b-v3 |
| Cohere | 35 秒/片段 (自动分块) | HuggingFace transformers 文档 |
| Moonshine | 无硬限制, 推荐 <30 秒 | GitHub moonshine-ai |
| GigaAM | 无明确限制 | sherpa-onnx 文档 |
| Whisper | 30 秒/窗口 (内部自动分块) | OpenAI 文档 |
