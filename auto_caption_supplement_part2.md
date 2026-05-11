# Auto Caption Whitepaper — Phụ lục Nghiên cứu Chuyên sâu (Part 2)

---

## PHỤ LỤC G: Conformer Architecture & CTC vs Attention-based ASR

### G.1 Conformer — Kiến trúc SOTA cho ASR

Conformer kết hợp **CNN** (local features) + **Transformer** (global context) trong thiết kế "sandwich":

```
Input → Macaron FFN (½) → Multi-Head Self-Attention → Convolution Module → Macaron FFN (½) → Output
```

**Conformer Block chi tiết:**
1. **Macaron FFN (½ step):** Half-step feed-forward, pre-process features
2. **MHSA:** Global context, long-range dependencies
3. **Convolution Module:** Depthwise-separable convolution → local patterns
4. **Macaron FFN (½ step):** Post-process features
5. **LayerNorm + Residual connections** throughout

**Fast-Conformer (2024-2025):**
- 8x depthwise-separable subsampling → 2.8x faster
- Kernel size 31→9 → fewer params, same accuracy
- Linear/chunked attention → handle longer audio
- Nền tảng cho NVIDIA Parakeet/Canary models

### G.2 CTC vs Attention-based vs RNN-T

| Tiêu chí | CTC | Attention (Seq2Seq) | RNN-Transducer |
|---|---|---|---|
| **Decoding** | Non-autoregressive | Autoregressive | Autoregressive |
| **Speed** | **Nhanh nhất** (parallelizable) | Chậm (sequential) | Trung bình |
| **Accuracy** | Tốt | **Tốt nhất** | Rất tốt |
| **Streaming** | ✅ Dễ | ❌ Khó | **✅ Tốt nhất** |
| **Conditional independence** | Có (weakness) | Không | Không |
| **Use case** | Batch/offline, low-latency | Offline, high accuracy | **Real-time streaming** |

**CTC (Connectionist Temporal Classification):**
- Output frame-level predictions → collapse repeated tokens + remove blanks
- **Conditional independence assumption:** Mỗi frame predict independently → miss inter-token dependencies
- Nhanh vì không cần sequential decoding

**Attention-based (Whisper):**
- Encoder-Decoder với cross-attention
- Autoregressive: predict token dựa trên all previous tokens + audio
- Chậm hơn nhưng chính xác hơn

**RNN-Transducer:**
- Kết hợp ưu điểm CTC (streaming) + Attention (context)
- Standard cho Google/NVIDIA streaming ASR
- Prediction network giữ language model context

**Hybrid CTC/Attention:** Shared encoder, dual decoder → train faster, switch strategy at runtime.

---

## PHỤ LỤC H: On-device ASR Alternatives (Ngoài Whisper)

### H.1 Sherpa-ONNX (k2-fsa/sherpa-onnx)

| Đặc điểm | Chi tiết |
|---|---|
| **Mô tả** | Cross-platform speech toolkit, ONNX Runtime |
| **Models** | Zipformer, Paraformer, Whisper, SenseVoice, TeleSpeech |
| **Platforms** | iOS, Android, Raspberry Pi, WebAssembly, Windows, Linux, macOS |
| **License** | Apache 2.0 |
| **Streaming** | ✅ Real-time streaming ASR |
| **Offline** | ✅ Hoàn toàn offline |
| **Languages** | Multi-model: English, Chinese, Vietnamese, etc. |

**Ưu điểm so với whisper.cpp:**
- Hỗ trợ **streaming** (whisper.cpp chỉ batch/offline)
- Nhiều model architectures (không chỉ Whisper)
- WebAssembly support → chạy trong browser
- Tích hợp VAD, keyword spotting, TTS

### H.2 SenseVoice (Alibaba/FunASR)

| Đặc điểm | SenseVoice-Small | SenseVoice-Large |
|---|---|---|
| **Architecture** | Non-autoregressive | Autoregressive |
| **Speed vs Whisper** | **5-15x faster** than Whisper-small | Comparable |
| **Languages** | 5 (ZH, EN, Cantonese, JA, KO) | 50+ |
| **Capabilities** | ASR + Emotion + Audio Events | ASR + Emotion + Audio Events |
| **Export** | ONNX, libtorch | ONNX, libtorch |
| **License** | MIT (model), Apache 2.0 (toolkit) | MIT |

**Unique Features:**
- **Emotion Recognition (SER):** Detect happy, sad, angry, neutral
- **Audio Event Detection (AED):** Music, applause, laughter, crying, coughing
- **Language ID** tự động
- Non-autoregressive → **cực kỳ nhanh** cho real-time

### H.3 Vosk (alphacephei/vosk)

| Đặc điểm | Chi tiết |
|---|---|
| **Models** | Kaldi-based, pre-trained cho 20+ ngôn ngữ |
| **Size** | 50MB (lightweight) hoặc 1.5GB (accurate) |
| **Streaming** | ✅ Native streaming |
| **Offline** | ✅ Hoàn toàn offline |
| **Platforms** | Android, iOS, Raspberry Pi, Desktop |
| **Accuracy** | Kém hơn Whisper đáng kể |
| **License** | Apache 2.0 |

### H.4 So sánh tổng hợp On-device Options

| Engine | Accuracy | Speed | Size | Streaming | Languages | Maturity |
|---|---|---|---|---|---|---|
| **whisper.cpp** | ★★★★ | ★★★ | 40-466MB | ❌ | 99+ | ★★★★★ |
| **Sherpa-ONNX** | ★★★★ | ★★★★ | 10-300MB | ✅ | Depends | ★★★★ |
| **SenseVoice** | ★★★★ | ★★★★★ | ~200MB | ✅ | 5-50+ | ★★★ |
| **Vosk** | ★★★ | ★★★★ | 50MB-1.5GB | ✅ | 20+ | ★★★★ |

**Khuyến nghị:** whisper.cpp cho accuracy-first offline. Sherpa-ONNX nếu cần streaming.

---

## PHỤ LỤC I: NVIDIA Canary & Parakeet (Next-gen ASR)

### I.1 Parakeet-TDT-0.6B-v3 (2025)

- **Architecture:** Fast-Conformer encoder + Token-and-Duration Transducer (TDT)
- **Params:** 600M
- **Training:** Granary dataset (~1M hours, 25 European languages)
- **Strengths:** High throughput, optimized for batch/real-time
- **Deployment:** NVIDIA Riva / NIM microservices
- **Limitation:** Requires NVIDIA GPU (CUDA), không phù hợp mobile

### I.2 Canary-1B-v2 (2025)

- **Architecture:** Fast-Conformer encoder-decoder
- **Params:** 1B
- **Languages:** 25 European languages (transcription + translation)
- **Use case:** High accuracy, complex multilingual scenarios
- **Deployment:** Server-side GPU

### I.3 Relevance cho Auto Caption

| Scenario | Parakeet/Canary | Whisper |
|---|---|---|
| **Cloud backend** | ✅ Excellent (nếu có NVIDIA GPU) | ✅ Excellent |
| **On-device mobile** | ❌ Không phù hợp | ✅ (via whisper.cpp) |
| **Cost** | Cần NVIDIA infrastructure | Flexible |
| **Open-source** | ✅ (HuggingFace) | ✅ |

**Kết luận:** Canary/Parakeet là candidates cho **Cloud fallback** thay thế/bổ sung Whisper, đặc biệt nếu đã có NVIDIA GPU infrastructure.

---

## PHỤ LỤC J: Speaker Diarization

### J.1 pyannote.audio Pipeline (2025)

```
Audio → pyannote VAD → Speaker Segmentation → Embedding Extraction → Clustering → Diarization Output
                                                                                          │
                                                                                          ▼
                                                                        "Speaker A: 0.0s-2.5s"
                                                                        "Speaker B: 2.6s-5.1s"
                                                                        "Speaker A: 5.2s-8.0s"
```

**Current best model:** `pyannote/speaker-diarization-community-1`

**Key Features (2025):**
- **Exclusive mode:** Non-overlapping segments → dễ align với ASR
- Improved speaker counting và assignment
- Requires HuggingFace access token

### J.2 Cascaded ASR + Diarization Pipeline

```
1. Diarization: pyannote → "who spoke when" timestamps
2. Segmentation: Slice audio theo speaker segments  
3. ASR: Whisper transcribe mỗi segment
4. Reconciliation: Map text → speaker identity
```

### J.3 Relevance cho Auto Caption

| Use Case | Cần Diarization? | Lý do |
|---|---|---|
| Solo creator (1 speaker) | ❌ | Không cần phân biệt |
| Interview/podcast (2+ speakers) | ✅ | Hiển thị tên speaker |
| Multi-speaker content | ✅ | Caption attribution |

**Khuyến nghị:** Không cần cho PoC (phần lớn short-form = 1 speaker). Thêm như premium feature sau.

---

## PHỤ LỤC K: Caption Rendering Engine — Kiến trúc Kỹ thuật

### K.1 Rendering Architecture

#### iOS — AVFoundation + CoreAnimation

```
Word timestamps + Style config
        │
        ▼
CATextLayer (per word) → CAKeyframeAnimation (highlight timing)
        │
        ▼
AVVideoCompositionCoreAnimationTool → Composite lên video track
        │
        ▼
Export via AVAssetExportSession
```

**Real-time Preview:** CALayer hierarchy overlay trên AVPlayerLayer
**Export:** AVVideoCompositionCoreAnimationTool composite animations vào video

#### Android — Media3 Transformer + Canvas/OpenGL

```
Word timestamps + Style config
        │
        ▼
Custom TextOverlayEffect (Media3 Effect API)
        │
        ▼
OpenGL ES / Vulkan shader → Text texture overlay
        │
        ▼
Media3 Transformer → Export video với overlay
```

**Real-time Preview:** SurfaceView + Canvas hardware-accelerated drawing
**Export:** Media3 Transformer API với custom Effect

### K.2 Karaoke Effect Implementation

**Core Logic (Pseudocode):**

```
function renderCaptionFrame(currentTime, words[], style):
    for each word in words:
        if currentTime < word.start:
            // Chưa đến → render inactive style (dim color)
            drawText(word.text, style.inactiveColor, style.font)
        else if currentTime >= word.start AND currentTime <= word.end:
            // Đang nói → render active style (highlight + animation)
            progress = (currentTime - word.start) / (word.end - word.start)
            drawText(word.text, style.activeColor, style.font, scale=1.0 + 0.1*progress)
            // Optional: bounce, glow, color sweep animation
        else:
            // Đã qua → render completed style
            drawText(word.text, style.completedColor, style.font)
```

### K.3 Text Rendering Techniques

| Technique | Performance | Quality | Flexibility |
|---|---|---|---|
| **Native Canvas/CALayer** | ★★★★ | ★★★★ | ★★★ |
| **SDF (Signed Distance Field)** | ★★★★★ | ★★★★★ | ★★★★★ |
| **Texture Atlas** | ★★★★★ | ★★★ | ★★ |
| **Skia (cross-platform)** | ★★★★ | ★★★★ | ★★★★ |

**SDF Text Rendering (Recommended cho high-quality):**
- Pre-rasterize glyphs thành SDF texture
- GPU fragment shader render text với outlines, glows, shadows
- Resolution-independent: zoom không bị pixelated
- Single draw call cho entire text → excellent performance

### K.4 Caption Style Templates

| Style | Visual | Technical |
|---|---|---|
| **Classic** | White text, black outline | Stroke + fill, drop shadow |
| **Karaoke Highlight** | Word-by-word color change | Per-word timing, color interpolation |
| **Pop/Bounce** | Scale animation per word | Spring animation, overshoot |
| **Typewriter** | Character-by-character appear | Character-level timing |
| **Gradient Sweep** | Color gradient moves across text | Shader-based gradient mask |
| **Background Box** | Colored box behind each word | Dynamic width box, rounded corners |

---

## PHỤ LỤC L: Accessibility & Compliance

### L.1 Regulatory Landscape (2025-2026)

| Regulation | Region | Effective | Caption Requirement |
|---|---|---|---|
| **ADA Title III** | USA | Ongoing | Required for public accommodations |
| **CVAA** | USA | 2012+ | Required for online video (broadcast origin) |
| **Section 508** | USA (Federal) | Ongoing | Required for federal content |
| **EAA (EU 2019/882)** | EU 27 | **28 June 2025** | Mandatory for digital services |
| **EN 301 549** | EU (Technical std) | Ongoing | Technical standard for EAA compliance |
| **AODA** | Ontario, Canada | Ongoing | Required for large organizations |

### L.2 WCAG 2.1 Requirements cho Captions

| Guideline | Level | Requirement |
|---|---|---|
| **1.2.2** | A | Captions (Prerecorded) — synchronized captions cho prerecorded audio |
| **1.2.4** | AA | Captions (Live) — captions cho live audio |
| **1.4.3** | AA | Contrast — text contrast ratio ≥ 4.5:1 |
| **1.4.6** | AAA | Enhanced Contrast — ratio ≥ 7:1 |

### L.3 EAA Impact cho Auto Caption Feature

**European Accessibility Act (có hiệu lực 28/6/2025):**
- Bắt buộc captioning cho **tất cả** audiovisual content trong EU
- Áp dụng cho **private companies** bán dịch vụ digital tại EU
- Caption phải **chính xác, đồng bộ, dễ đọc**
- Phải cung cấp **bằng ngôn ngữ thị trường** (không chỉ English)
- Vi phạm → **phạt tài chính** theo từng Member State

**Cơ hội kinh doanh:**
- Auto Caption trở thành **compliance tool**, không chỉ creative tool
- Creators/businesses **bắt buộc** phải có caption → tăng demand
- App hỗ trợ multi-language auto caption = competitive advantage lớn trong EU market

---

## PHỤ LỤC M: Vietnamese ASR — Benchmark & Considerations

### M.1 Vietnamese ASR Performance (2024-2025)

| Model/Service | WER (VLSP 2020) | WER (FLEURS) | Ghi chú |
|---|---|---|---|
| **PhoWhisper-large** | ~13.75% | — | SOTA open-source Vietnamese |
| **Zipformer-30M (6000h)** | ~12.29% | — | Nguyen 2025, competitive |
| **Speechmatics Enhanced** | — | ~7.14% | Commercial, best accuracy |
| **Google Chirp 2** | — | ~8.38% | Commercial cloud API |
| **Amazon Transcribe** | — | ~9.32% | Commercial cloud API |
| **Azure Speech** | — | ~10.08% | Commercial cloud API |
| **OpenAI Whisper large-v3** | — | ~10.25% | Open-source |
| **Deepgram Nova-2** | — | ~11.36% | Commercial cloud API |

### M.2 Thách thức đặc thù Vietnamese

1. **Tonal language:** 6 tones → dấu thanh critical cho nghĩa (ma/má/mà/mả/mã/mạ)
2. **Regional accents:** Bắc/Trung/Nam phát âm khác biệt lớn
3. **Code-switching:** Trộn Vietnamese-English phổ biến trong content creator
4. **Syllable-based:** Mỗi syllable = 1 từ → word segmentation khác English
5. **Diacritics:** Unicode normalization (NFC vs NFD) gây issues

### M.3 Khuyến nghị cho Vietnamese Support

| Approach | Pros | Cons |
|---|---|---|
| **Whisper multilingual** | Sẵn có, decent accuracy | WER ~10-14%, tonal errors |
| **PhoWhisper (fine-tuned)** | Best open-source Vietnamese | Cần self-host |
| **Google Chirp 2 API** | WER ~8.38%, production-ready | Cloud-only, cost |
| **Custom fine-tune** | Best possible accuracy | Cần data + compute |

**PoC Strategy:** Whisper multilingual cho MVP → Fine-tune hoặc PhoWhisper cho production.

---

## PHỤ LỤC N: faster-whisper & CTranslate2

### N.1 faster-whisper

**Mô tả:** Reimplementation của Whisper sử dụng CTranslate2 inference engine.

| Đặc điểm | faster-whisper | openai-whisper |
|---|---|---|
| **Speed** | **~4x faster** | 1x baseline |
| **Memory** | **~2-3x less** | High |
| **Backend** | CTranslate2 (C++) | PyTorch |
| **Quantization** | INT8, FP16 native | FP16/FP32 |
| **Batched inference** | ✅ | Limited |
| **Word timestamps** | ✅ (cross-attention) | ✅ |
| **VAD integration** | ✅ Silero VAD built-in | ❌ |

**CTranslate2 optimizations:**
- Weight quantization (INT8/FP16)
- Layer fusion
- Batch reordering
- KV cache optimization
- Multi-GPU support

### N.2 Vai trò trong Auto Caption Architecture

```
Cloud Backend Architecture:
    faster-whisper (GPU server) → 4x throughput vs vanilla Whisper
                                → INT8 quantization → fit smaller GPU
                                → Silero VAD built-in → reduce hallucination
                                → Word-level timestamps → karaoke ready
```

**Cost Impact:** 4x faster = 4x fewer GPU-hours = **~75% cost reduction** vs vanilla Whisper deployment.

---

## References (Bổ sung)

### Research Papers (Bổ sung)
- Gulati, A., et al. "Conformer: Convolution-augmented Transformer for Speech Recognition." INTERSPEECH, 2020.
- Graves, A., et al. "Connectionist Temporal Classification." ICML, 2006.
- He, Y., et al. "Streaming End-to-End Speech Recognition for Mobile Devices." ICASSP, 2019.
- An, D., et al. "FunASR: A Fundamental End-to-End Speech Recognition Toolkit." 2023.
- Nguyen, V.N. "PhoWhisper: Automatic Speech Recognition for Vietnamese." 2024.
- Bredin, H. "pyannote.audio: Neural Building Blocks for Speaker Diarization." ICASSP, 2023.

### Technical Resources (Bổ sung)
- linto-ai/whisper-timestamped — GitHub. Word-level timestamps via DTW.
- NVIDIA NeMo-text-processing — Inverse Text Normalization.
- k2-fsa/sherpa-onnx — Cross-platform speech toolkit.
- FunAudioLLM/SenseVoice — Alibaba speech understanding.
- SYSTRAN/faster-whisper — CTranslate2-based Whisper.

### Regulatory
- European Accessibility Act (EU) 2019/882 — effective 28 June 2025.
- W3C WCAG 2.1 — Web Content Accessibility Guidelines.
- EN 301 549 — European standard for ICT accessibility.

---

*Tài liệu bổ sung này được biên soạn dựa trên nghiên cứu tính đến 11/05/2026.*
