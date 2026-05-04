# Auto Caption cho Short-form Video Editor
## Cáo bạch Kỹ thuật & Chiến lược (Technical & Strategic Whitepaper)

**Phiên bản:** 1.0 | **Ngày:** 03/05/2026  
**Tác giả:** Principal Video AI Architect & Strategic Research Director

---

## Executive Summary

Auto Caption không còn là "Nice-to-have" — nó là **tính năng sống còn** quyết định sự thành bại của bất kỳ ứng dụng Short-form Video Editor nào. Dữ liệu cho thấy **~85% video trên mạng xã hội được xem ở chế độ tắt tiếng**, và video có caption tăng **12–40% thời gian xem**, **80% tỷ lệ xem hết**, và **26% tăng CTA click-through**. Các nền tảng TikTok, Reels, Shorts đều sử dụng ASR và OCR để quét nội dung video phục vụ thuật toán phân phối, biến caption thành yếu tố ảnh hưởng trực tiếp đến organic reach.

Báo cáo này phân tích toàn diện 4 tầng: **(1)** Chiến lược kinh doanh, **(2)** Pipeline thuật toán ASR từ Silero VAD đến Word-level Timestamp Alignment, **(3)** Kiến trúc triển khai Cloud vs. Edge AI với phân tích chi phí scale 1M users, **(4)** Intelligence đối thủ (CapCut, Wink, Premiere Rush) và đề xuất Tech Stack cho PoC 1 tháng.

**Kết luận chiến lược:** Kiến trúc Hybrid (On-device cho short-form ≤3 phút, Cloud fallback cho long-form/accuracy-critical) sử dụng whisper.cpp + Silero VAD là phương án tối ưu về chi phí, trải nghiệm và khả năng cạnh tranh.

---

## Mục lục

1. [TẦNG 1: Phân tích Chiến lược](#tầng-1-phân-tích-chiến-lược)
2. [TẦNG 2: Lõi Thuật toán ASR Pipeline](#tầng-2-lõi-thuật-toán-asr-pipeline)
3. TẦNG 3: Kiến trúc Triển khai — xem Part 2
4. TẦNG 4: Intelligence Đối thủ & Tech Stack PoC — xem Part 2
5. References — xem Part 2

---

## TẦNG 1: Phân tích Chiến lược

### 1.1 Hành vi Tiêu thụ Video (User Consumption Behavior)

#### Muted Viewing — Thực tế không thể bỏ qua

| Chỉ số | Số liệu | Nguồn |
|---|---|---|
| Video mạng xã hội xem tắt tiếng | **~85%** | Nhiều báo cáo ngành 2024–2025 |
| Video Facebook xem tắt tiếng | **~85%** | Facebook Internal Data |
| Video mobile xem tắt tiếng | **~92%** | Industry Reports 2024 |
| Gen Z dùng caption dù có âm thanh | **~80%** | Kapwing Research 2024 |

**Nguyên nhân gốc rễ:**
- **Bối cảnh sử dụng:** Người dùng lướt mạng xã hội ở nơi công cộng, văn phòng — nơi bật âm thanh là không phù hợp.
- **Platform Default:** Hầu hết nền tảng auto-play video ở chế độ mute.
- **Thói quen thế hệ:** Gen Z coi caption như một phần trải nghiệm content, không chỉ là công cụ hỗ trợ.

#### Tâm lý học Kinetic Typography

Kinetic Typography (chữ động) là **công cụ giữ chân người dùng**:

- **Dual-coding Theory (Paivio, 1971):** Thông tin trình bày đồng thời qua cả kênh thị giác (text) và thính giác (audio), khả năng ghi nhớ tăng gấp đôi.
- **Visual Saliency:** Chữ động (highlight, bounce, scale) tạo **visual anchoring points** — mắt người dùng bị "khóa" vào text đang chuyển động, giảm khả năng scroll.
- **Karaoke Effect:** Highlight từng từ theo nhịp nói tạo cảm giác **synchrony**, kích hoạt reward circuits trong não.

#### Tác động của Caption đến Engagement

| Chỉ số | Tác động | Nguồn |
|---|---|---|
| Tăng thời gian xem | **+12% đến +40%** | Nhiều nghiên cứu 2024 |
| Tăng tổng lượt xem | **+7.32%** | Discovery Digital Networks |
| Tăng views 14 ngày đầu | **+13.48%** | Discovery Digital Networks |
| Tăng watch time | **+25%** | 3PlayMedia Study |
| Xem hết video | **80%** khả năng cao hơn | Industry Reports |
| Tăng engagement | **lên đến +80%** | Industry Reports |
| Tăng CTA click | **+26%** | SubCap/Industry Data |

### 1.2 Thuật toán Phân phối (Algorithm Impact)

#### TikTok sử dụng 3 lớp công nghệ index nội dung:

1. **ASR:** Transcribe toàn bộ lời nói → mỗi từ = keyword để categorize.
2. **OCR:** Quét frame-by-frame đọc text trên video, bao gồm caption burn-in.
3. **Visual Analysis:** Nhận diện objects, scenes → "semantic fingerprint".

**Hệ quả:** Caption trực tiếp cung cấp keyword signals cho thuật toán. Video có caption = nhiều text data = thuật toán hiểu tốt hơn = phân phối chính xác hơn.

#### Hai Hệ thống Khám phá

| Hệ thống | Cơ chế | Vai trò Caption |
|---|---|---|
| **For You Page** | User interaction signals (retention, engagement) | Caption tăng retention → tăng xác suất FYP |
| **Search Discovery** | Relevance matching (ASR+OCR+metadata) | Caption = searchable text → SEO video |

### 1.3 Business ROI

#### Nỗi đau cốt lõi của Creator

1. **Time-to-Publish:** Gõ caption thủ công video 60s mất 15–30 phút. Auto caption: <1 phút.
2. **Reach Anxiety:** Không caption = giảm reach, nhưng tạo thủ công quá tốn thời gian.
3. **Multi-platform:** Mỗi nền tảng format khác nhau. Auto caption + templates giải quyết.
4. **Accessibility Compliance:** Xu hướng luật (ADA, EAA) yêu cầu caption.

#### Must-have Analysis

| Tiêu chí | Đánh giá |
|---|---|
| Tần suất sử dụng | Mỗi video, mỗi ngày |
| Ảnh hưởng KPI Creator | Trực tiếp: reach, views, retention |
| Switching cost nếu thiếu | Rất cao — creator chuyển app |
| Competitive table stakes | CapCut đã có → không có = thua |

**Kết luận:** Auto Caption là **retention feature** — thiếu nó, user rời app.

---

## TẦNG 2: Lõi Thuật toán ASR Pipeline

### 2.1 Pipeline ASR End-to-End

```
Video → Audio Extract → VAD → Feature Extract → Acoustic Model → Language Model → Word Timestamp Align
```

**Bước 1 - Audio Extraction:** Tách audio từ MP4/MOV → WAV mono 16kHz 16-bit. Dùng FFmpeg hoặc native API (AVFoundation iOS / MediaExtractor Android). Lưu ý: FFmpegKit đã retired (2025), cần custom wrapper hoặc native API.

**Bước 2 - VAD (Silero VAD v5+):**

| Thông số | Giá trị |
|---|---|
| Kích thước | **~2 MB** (ONNX) |
| Latency/chunk | **<1 ms** (single CPU) |
| Training data | 6,000+ ngôn ngữ |
| Window size | 256/512 samples |

**Bước 3 - Feature Extraction:** Raw audio → 80-channel log-Mel spectrogram (25ms window, 10ms hop).

**Bước 4&5 - Acoustic + Language Model:** Trong E2E models (Whisper, SeamlessM4T), hợp nhất trong Encoder-Decoder Transformer.

### 2.2 SOTA Models So sánh

| Tiêu chí | Whisper large-v3 | large-v3-turbo | Distil-Whisper | SeamlessM4T v2 | Google USM |
|---|---|---|---|---|---|
| **Params** | 1.55B | ~809M | ~756M | ~2.3B | 2B |
| **Ngôn ngữ** | 99+ | 99+ | English only | 100+ | 300+ |
| **WER (en)** | ~5–7% | ~7.75–9.5% | Within 1% of v3 | Competitive | SOTA |
| **Speed vs v3** | 1x | **6x** | **6.3x** | N/A | Cloud-only |
| **Disk** | ~3.1 GB | ~1.6 GB | ~1.5 GB | ~4.5 GB | Cloud-only |
| **Open-source** | ✅ | ✅ | ✅ | ✅ | ❌ API only |

#### Whisper large-v3-turbo
- Pruned decoder 32→4 layers. ~6x faster, WER chỉ +1–2%.
- ~809M params, ~6GB VRAM. Phù hợp cloud/GPU, không mobile.

#### whisper.cpp — Edge Deployment

| Model | Disk | RAM | Mobile |
|---|---|---|---|
| tiny | ~75 MB | ~270–390 MB | ✅ Tốt |
| base | ~142 MB | ~390–500 MB | ✅ Khá |
| small | ~466 MB | ~850 MB–1 GB | ⚠️ Flagship only |

- Hỗ trợ CoreML (Apple Neural Engine) + Metal
- Quantization Q4_0/Q5_0 = sweet spot mobile

#### Distil-Whisper
- Knowledge Distillation: encoder nguyên vẹn, decoder 32→2 layers
- **6.3x faster**, WER within 1% teacher, giảm hallucination long-form
- **Hạn chế: English only**

#### Meta SeamlessM4T v2
- Encoder: w2v-BERT 2.0 (4.5M hours training)
- Multimodal: S2S, S2T, T2S. Robust noise/speaker variation
- Quá lớn (~4.5GB) cho on-device

#### Google USM/Chirp
- Paper: arXiv:2303.01037. Conformer, 2B params, 300+ languages
- Training: 12M hours unlabeled + 28B sentences
- **Cloud-only**, không open-source

### 2.3 Word-level Timestamp

| Kỹ thuật | Cách hoạt động | Accuracy |
|---|---|---|
| **Forced Alignment** | Model ngoài (wav2vec 2.0) align transcript→audio | Rất cao |
| **Attention Extraction** | Cross-attention weights Whisper + DTW | Cao |
| **E2E Timestamping** | Model emit timestamp tokens trực tiếp | Cao |

#### WhisperX — Production-ready

1. Transcribe bằng Whisper
2. Forced Alignment bằng wav2vec 2.0 → timestamp ms-level cho mỗi từ

```json
{"words": [
  {"word": "Hello", "start": 0.42, "end": 0.78, "score": 0.95},
  {"word": "world", "start": 0.82, "end": 1.15, "score": 0.92}
]}
```

Khi playback time = `start` → trigger highlight animation → Karaoke effect.

---

*Tiếp tục Part 2: Tầng 3 & 4*
# Auto Caption Whitepaper — Part 2
## TẦNG 3: Kiến trúc Triển khai & TẦNG 4: Intelligence Đối thủ

---

## TẦNG 3: Kiến trúc Triển khai (Cloud vs. Edge AI)

### 3.1 Kiến trúc Cloud-based

#### Ưu điểm
- **Accuracy cao nhất:** Chạy large model (Whisper large-v3, USM) không giới hạn bởi device
- **Multilingual:** Full 99–300+ languages
- **Không phụ thuộc hardware:** Mọi device đều dùng được
- **Cập nhật model:** Deploy model mới không cần update app

#### Nhược điểm
- **Latency:** Upload audio + inference + download = 3–15 giây cho video 60s
- **Phụ thuộc internet:** Không hoạt động offline
- **Chi phí bandwidth:** Upload audio cho mỗi video
- **Privacy:** Audio gửi lên server → concern GDPR/privacy

#### Chi phí Cloud API — So sánh

| Provider | Giá/phút | Billing | Ghi chú |
|---|---|---|---|
| **Google Cloud STT** | ~$0.016 | Tiered volume | Dynamic batch ~75% rẻ hơn |
| **AWS Transcribe** | ~$0.024 | Per-second, min 15s | Volume discount |
| **Azure Speech** | ~$0.006–$0.017 | Batch vs Real-time | Batch rẻ hơn nhiều |

#### Mô hình Chi phí Scale 1M Users

**Giả định:**
- 1M MAU, mỗi user tạo trung bình 3 video/tuần
- Video trung bình 60 giây = 1 phút audio
- Tổng: 1M × 3 × 4 = **12M phút/tháng**

| Provider | Chi phí/tháng (12M phút) |
|---|---|
| Google Cloud (standard) | **~$192,000** |
| Google Cloud (dynamic batch) | **~$48,000** |
| AWS Transcribe | **~$288,000** |
| Azure (batch) | **~$72,000–$204,000** |

**Chi phí ẩn bổ sung:**
- Storage (S3/GCS): ~$2,000–5,000/tháng
- Compute (Lambda/Cloud Functions): ~$3,000–8,000/tháng
- Data egress: ~$1,000–3,000/tháng
- **Tổng ước tính thực tế: $75,000–$300,000/tháng** tùy provider

**Self-hosted alternative:** Deploy Whisper large-v3-turbo trên GPU cluster:
- 10–20 NVIDIA A100 instances: ~$15,000–30,000/tháng
- Infra + DevOps overhead: ~$5,000–10,000/tháng
- **Tổng: ~$20,000–40,000/tháng** — rẻ hơn 3–7x so với managed API

### 3.2 Kiến trúc On-device (Edge AI)

#### 3.2.1 Kỹ thuật Nén Model (Model Compression)

##### Quantization

| Kỹ thuật | Nguyên lý | Kết quả | Trade-off |
|---|---|---|---|
| **FP16→INT8** | Giảm precision từ 16-bit float → 8-bit integer | ~2–4x nhỏ hơn, ~2–4x nhanh hơn | WER tăng <0.5% |
| **INT8→INT4** | Tiếp tục giảm xuống 4-bit | ~4–8x nhỏ hơn so với FP16 | WER tăng 1–3% |
| **AWQ** | Activation-aware Weight Quantization | Giảm accuracy loss ở extreme quantization | Cần calibration data |
| **GPTQ** | Post-training quantization cho Transformers | INT4 với accuracy gần INT8 | One-time compute cost |

**Ví dụ thực tế:** Whisper small (466MB FP16) → INT4 quantization → **~120MB** — fit cho mobile.

##### Pruning

- **Structured Pruning:** Loại bỏ entire filters/channels/attention heads → tạo dense model nhỏ hơn → trực tiếp nhanh hơn trên mobile NPU/CPU mà không cần hardware đặc biệt.
- **Unstructured Pruning:** Loại bỏ individual weights → sparsity → cần hardware hỗ trợ sparse computation mới thực sự nhanh hơn.

**Khuyến nghị mobile: Structured pruning** vì tạo ra model dense, tương thích mọi hardware.

##### Knowledge Distillation

Pipeline tối ưu cho mobile:
1. **Distill:** Whisper large-v3 (teacher) → compact student model
2. **Prune:** Structured pruning loại bỏ redundancy
3. **Quantize:** INT8 (stable) hoặc INT4 (aggressive)

Kết quả: Model 1.5GB → **<100MB** với accuracy chấp nhận được (WER +2–5%).

#### 3.2.2 Hardware Acceleration

##### iOS — Apple Neural Engine + CoreML

```
whisper.cpp model (GGML) → CoreML conversion → Apple Neural Engine (ANE)
```

- **Apple Neural Engine:** Dedicated NPU, 15.8 TOPS (A16), ~35 TOPS (M-series)
- **CoreML integration:** whisper.cpp hỗ trợ native CoreML backend
- **Kết quả:** Whisper tiny/base chạy **nhanh hơn real-time** trên iPhone 14+
- **Lợi ích:** CPU/GPU rảnh để xử lý UI, giảm nhiệt, tiết kiệm pin

##### Android — NNAPI + ONNX Runtime

```
Whisper model → ONNX export → ONNX Runtime Mobile → NNAPI delegation → NPU/DSP
```

- **NNAPI (Android Neural Networks API):** Abstraction layer → delegate inference đến NPU/DSP/GPU tùy device
- **ONNX Runtime Mobile:** Cross-platform inference engine, hỗ trợ NNAPI, QNN (Qualcomm), XNNPACK
- **Thách thức:** Fragmentation — hiệu năng NPU khác nhau lớn giữa Snapdragon/Exynos/MediaTek/Tensor
- **Giải pháp:** Feature detection at runtime → chọn backend tối ưu (NPU → GPU → CPU fallback)

##### Bảng So sánh Hardware Acceleration

| Platform | NPU | Framework | Backend | Whisper tiny RTF |
|---|---|---|---|---|
| iOS (A16+) | Apple Neural Engine | CoreML | ANE | ~0.3x (3x faster than RT) |
| iOS (A16+) | — | Metal | GPU | ~0.5x |
| Android (SD 8 Gen 3) | Hexagon DSP | ONNX/QNN | NPU | ~0.5–0.8x |
| Android (mid-range) | — | XNNPACK | CPU | ~1.0–1.5x |

*RTF = Real-Time Factor. <1.0 = faster than real-time*

#### 3.2.3 Memory & Thermal Management

##### Vấn đề: Video dài + Device yếu

- Whisper tiny load vào RAM: ~270–390 MB
- Whisper base: ~390–500 MB
- Device 3–4GB RAM tổng, OS + apps chiếm ~2GB → chỉ còn 1–2GB free

##### Giải pháp kiến trúc

**1. Chunked Processing (Streaming Architecture):**
```
Audio dài → Split thành chunks 30s → Process từng chunk → Merge results
```
- Không load toàn bộ audio vào memory
- Mỗi chunk xử lý xong → release memory → chunk tiếp theo
- Overlap 1–2s giữa chunks để tránh cắt giữa từ

**2. Memory-mapped Model Loading:**
- Dùng `mmap()` thay vì load toàn bộ model vào RAM
- OS quản lý page in/out → chỉ load phần model đang dùng
- whisper.cpp hỗ trợ native mmap

**3. Thermal Throttling Mitigation:**
- **Duty cycling:** Process 30s → pause 2–3s → process tiếp
- **Monitoring:** Đọc thermal sensors (iOS: `ProcessInfo.thermalState`, Android: `ThermalManager`)
- **Adaptive quality:** Khi nhiệt cao → tự động downgrade model (small → base → tiny)
- **Background processing:** Chạy inference ở background priority → OS tự throttle hợp lý

**4. Progressive UX:**
- Hiển thị caption cho chunks đã xong ngay lập tức
- User có thể bắt đầu edit trong khi chunks sau vẫn đang process
- Progress bar thể hiện % hoàn thành

### 3.3 Kiến trúc Hybrid — Đề xuất Tối ưu

```
                    ┌─────────────────┐
                    │   User Request   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Decision Engine │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              │              ▼
    ┌─────────────────┐      │    ┌─────────────────┐
    │   On-Device      │      │    │   Cloud API      │
    │   (whisper.cpp)  │      │    │   (Whisper API /  │
    │   tiny/base      │      │    │    self-hosted)   │
    └─────────────────┘      │    └─────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Decision Rules  │
                    └─────────────────┘
```

**Decision Rules:**
| Điều kiện | Route | Lý do |
|---|---|---|
| Video ≤3 phút + có internet | On-device | Nhanh, miễn phí, privacy |
| Video ≤3 phút + offline | On-device | Chỉ option khả dụng |
| Video >3 phút | Cloud | Accuracy + speed tốt hơn |
| User yêu cầu "HD accuracy" | Cloud | large-v3 cho kết quả tốt nhất |
| Ngôn ngữ rare (không trong tiny model) | Cloud | Multilingual support |
| Device low-end (<4GB RAM) | Cloud | Tránh OOM/thermal |

---

## TẦNG 4: Intelligence Đối thủ & Tech Stack PoC

### 4.1 Competitive Intelligence

#### CapCut (ByteDance)

| Khía cạnh | Phân tích |
|---|---|
| **Công nghệ** | **Cloud-based ASR** |
| **Bằng chứng** | Yêu cầu internet; tốc độ dao động theo server load; audio upload lên ByteDance servers |
| **Privacy** | Audio xử lý trên server ByteDance → concern privacy |
| **Ưu điểm** | Accuracy cao, đa ngôn ngữ, nhiều style template |
| **Nhược điểm** | Phụ thuộc internet, privacy concern, tốc độ không nhất quán |
| **Chiến lược** | Leverage hạ tầng cloud khổng lồ ByteDance, ưu tiên accuracy trên tất cả |

#### Wink (Video Enhancer)

| Khía cạnh | Phân tích |
|---|---|
| **Công nghệ** | **Cloud-based AI** |
| **Bằng chứng** | Yêu cầu internet bắt buộc; app nói rõ cần kết nối để generate subtitles |
| **Ưu điểm** | UX tốt cho social media, built-in styling |
| **Nhược điểm** | Cloud dependency, tốc độ phụ thuộc network |

#### Adobe Premiere Rush

| Khía cạnh | Phân tích |
|---|---|
| **Công nghệ** | **KHÔNG CÓ** native auto caption |
| **Context** | Adobe Sensei-powered Speech-to-Text chỉ có trong Premiere Pro (bản full) |
| **Tương lai** | Adobe thông báo sẽ discontinue Rush → chuyển sang tools mới |
| **Cơ hội** | Khoảng trống thị trường: users Rush không có auto caption → target acquisition |

#### Bảng tổng hợp

| App | Auto Caption | Processing | Offline | Privacy |
|---|---|---|---|---|
| **CapCut** | ✅ | Cloud | ❌ | ⚠️ ByteDance servers |
| **Wink** | ✅ | Cloud | ❌ | ⚠️ Cloud processing |
| **Premiere Rush** | ❌ | N/A | N/A | N/A |
| **Đề xuất (Ours)** | ✅ | **Hybrid** | ✅ (on-device) | ✅ (on-device mode) |

**Competitive Advantage:** Kiến trúc Hybrid cho phép chúng ta là app **duy nhất** cung cấp:
1. Auto caption **offline** (privacy-first)
2. Cloud fallback cho accuracy/multilingual
3. Nhanh hơn cloud-only competitors cho short-form content

### 4.2 Tech Stack PoC — Đề xuất Chi tiết

#### Core Libraries

| Component | Library | License | Platform | Vai trò |
|---|---|---|---|---|
| **ASR Engine** | [whisper.cpp](https://github.com/ggerganov/whisper.cpp) | MIT | iOS/Android/Desktop | On-device inference |
| **VAD** | [Silero VAD](https://github.com/snakers4/silero-vad) | MIT | Cross-platform (ONNX) | Voice detection pre-filter |
| **Timestamp Align** | [WhisperX](https://github.com/m-bain/whisperX) | BSD | Python (server-side) | Word-level timestamp |
| **Inference Runtime** | [ONNX Runtime Mobile](https://onnxruntime.ai/) | MIT | iOS/Android | Cross-platform ML runtime |
| **Audio Extract** | AVFoundation (iOS) / MediaExtractor (Android) | Platform | Native | Audio từ video |
| **Audio Processing** | Custom FFmpeg wrapper hoặc native API | LGPL/Native | Cross-platform | Resample, format convert |
| **Cloud Fallback** | OpenAI Whisper API hoặc Self-hosted faster-whisper | - | Server | Large model inference |

#### Architecture Diagram — PoC

```
┌──────────────────────────────────────────────────┐
│                    Mobile App                     │
│  ┌────────────┐  ┌──────────┐  ┌──────────────┐ │
│  │ Video Input │→ │ Audio    │→ │ Silero VAD   │ │
│  │ (Camera/    │  │ Extract  │  │ (ONNX, 2MB)  │ │
│  │  Gallery)   │  │ (Native) │  │ Speech detect│ │
│  └────────────┘  └──────────┘  └──────┬───────┘ │
│                                       │          │
│                          ┌────────────▼────────┐ │
│                          │  Decision Engine     │ │
│                          │  (length/device/net) │ │
│                          └───┬─────────────┬───┘ │
│                              │             │     │
│                    ┌─────────▼──┐   ┌──────▼───┐ │
│                    │ whisper.cpp│   │ Cloud API │ │
│                    │ tiny/base  │   │ (REST)    │ │
│                    │ CoreML/    │   │           │ │
│                    │ ONNX RT    │   │           │ │
│                    └─────┬─────┘   └─────┬────┘ │
│                          │               │      │
│                    ┌─────▼───────────────▼────┐ │
│                    │  Timestamp Merge +        │ │
│                    │  Caption Renderer         │ │
│                    │  (Karaoke/Highlight/Style)│ │
│                    └──────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

#### PoC Timeline — 4 Tuần

| Tuần | Deliverable | Chi tiết |
|---|---|---|
| **Tuần 1** | Audio Pipeline + VAD | FFmpeg/native audio extract, Silero VAD integration, chunking logic |
| **Tuần 2** | On-device ASR | whisper.cpp tiny model integration, CoreML/ONNX setup, basic transcription |
| **Tuần 3** | Timestamp + Rendering | Word-level timestamp (attention-based hoặc simplified forced align), Caption overlay UI, basic karaoke effect |
| **Tuần 4** | Cloud Fallback + Polish | Cloud API integration, Decision Engine logic, Error handling, UX polish, Performance profiling |

#### Model Selection Guide cho PoC

| Scenario | Model | Size | Speed | Accuracy |
|---|---|---|---|---|
| **PoC Demo** | whisper.cpp tiny (Q5_0) | ~40 MB | Very fast | Acceptable |
| **Production v1** | whisper.cpp base (Q5_0) | ~80 MB | Fast | Good |
| **Production v2** | whisper.cpp small (Q4_0) | ~120 MB | Moderate | Very Good |
| **Cloud fallback** | faster-whisper large-v3-turbo | Server-side | Fast (GPU) | Excellent |

---

## Trade-off Matrix Tổng hợp

| Tiêu chí | Cloud-only | On-device only | Hybrid (Đề xuất) |
|---|---|---|---|
| **Accuracy** | ★★★★★ | ★★★☆☆ | ★★★★☆ |
| **Speed (short-form)** | ★★★☆☆ | ★★★★★ | ★★★★★ |
| **Offline** | ❌ | ✅ | ✅ |
| **Privacy** | ⚠️ | ✅ | ✅ |
| **Cost at scale** | $$$$$ | $ (device cost) | $$ |
| **Multilingual** | ★★★★★ | ★★☆☆☆ | ★★★★☆ |
| **Device compatibility** | ★★★★★ | ★★★☆☆ | ★★★★☆ |
| **Maintenance** | ★★★☆☆ | ★★★★☆ | ★★★☆☆ |

---

## References

### Research Papers
1. Radford, A., et al. "Robust Speech Recognition via Large-Scale Weak Supervision." OpenAI, 2022. (Whisper paper)
2. Zhang, Y., et al. "Google USM: Scaling Automatic Speech Recognition Beyond 100 Languages." arXiv:2303.01037, 2023.
3. Barrault, L., et al. "SeamlessM4T: Massively Multilingual & Multimodal Machine Translation." Meta AI, 2023.
4. Gandhi, S., et al. "Distil-Whisper: Robust Knowledge Distillation via Large-Scale Pseudo Labelling." Hugging Face, 2023.
5. Bain, M., et al. "WhisperX: Time-Accurate Speech Transcription of Long-Form Audio." INTERSPEECH, 2023.
6. Baevski, A., et al. "wav2vec 2.0: A Framework for Self-Supervised Learning of Speech Representations." Meta AI, 2020.
7. Paivio, A. "Imagery and Verbal Processes." 1971. (Dual-coding theory)

### Technical Documentation & Blogs
8. ggerganov/whisper.cpp — GitHub Repository. MIT License.
9. snakers4/silero-vad — GitHub Repository. MIT License.
10. ONNX Runtime Mobile — Microsoft. https://onnxruntime.ai/
11. Apple Core ML Documentation — https://developer.apple.com/documentation/coreml
12. Android NNAPI — https://developer.android.com/ndk/guides/neuralnetworks

### Industry Data & Reports
13. "85% of social media videos watched on mute" — Multiple industry reports, 2024–2025.
14. Discovery Digital Networks / 3PlayMedia — Caption engagement study.
15. Kapwing — Gen Z caption usage research, 2024.
16. TikTok Algorithm Analysis — darkroomagency.com, almcorp.com, hootsuite.com.

### Cloud Pricing (verified Q1 2026)
17. Google Cloud Speech-to-Text Pricing — cloud.google.com
18. AWS Transcribe Pricing — aws.amazon.com
19. Azure AI Speech Pricing — azure.microsoft.com

### Competitive Analysis Sources
20. CapCut cloud-based ASR analysis — flowith.io, reddit.com discussions, 2024–2025.
21. Wink AI Subtitles — wnkapp.com official documentation.
22. Adobe Premiere Rush discontinuation — adobe.com, 2026.

---

*Tài liệu này được biên soạn dựa trên nghiên cứu tính đến ngày 03/05/2026. Các số liệu giá cả và benchmark có thể thay đổi. Khuyến nghị verify với nguồn chính thức trước khi ra quyết định kinh doanh.*
