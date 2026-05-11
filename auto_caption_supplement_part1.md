# Auto Caption Whitepaper — Phụ lục Nghiên cứu Chuyên sâu (Part 1)
## Bổ sung cho auto_caption_whitepaper.md

**Phiên bản:** 2.0 | **Ngày:** 11/05/2026
**Mục đích:** Bổ sung chi tiết kỹ thuật còn thiếu trong whitepaper gốc

---

## PHỤ LỤC A: Kiến trúc Chi tiết Whisper Model

### A.1 Encoder-Decoder Transformer Architecture

Whisper sử dụng kiến trúc **Encoder-Decoder Transformer** chuẩn, theo mô hình sequence-to-sequence từ paper "Attention Is All You Need" (Vaswani et al., 2017).

#### Encoder — Audio Feature Extraction

```
Raw Audio (16kHz) → 80-ch Log-Mel Spectrogram → Conv1D Stem → Transformer Encoder Blocks → Encoded Features
```

**Convolutional Stem (Pre-Transformer):**
- **Conv1D Layer 1:** filter_width=3, GELU activation
- **Conv1D Layer 2:** filter_width=3, stride=2 → downsampling time dimension 2x
- Mục đích: Giảm sequence length trước khi vào Transformer, giảm compute cost

**Positional Embeddings:** Fixed sinusoidal (không trainable)

**Mỗi Encoder Block gồm:**
1. **Pre-Layer Normalization** (Pre-LN) — normalize trước attention, stabilize deep training
2. **Multi-Head Self-Attention** — relate different time steps trong audio
3. **Residual Connection**
4. **Pre-LN → Feed-Forward Network (MLP)** → Residual Connection

#### Decoder — Autoregressive Text Generation

**Positional Embeddings:** Learned (trainable) — khác encoder

**Mỗi Decoder Block gồm:**
1. **Masked Self-Attention** — chỉ attend tokens trước đó (causal)
2. **Cross-Attention** — "nhìn" vào encoder output, quyết định phần audio nào liên quan đến token đang generate
3. **Feed-Forward Network (MLP)**

#### Bảng So sánh Encoder vs Decoder

| Đặc điểm | Encoder | Decoder |
|---|---|---|
| **Vai trò** | Trích xuất features từ audio | Sinh text token-by-token |
| **Positional Embed** | Fixed (sinusoidal) | Learned |
| **Attention** | Self-attention (bidirectional) | Masked self-attention + Cross-attention |
| **Input** | Log-Mel spectrogram frames | Token IDs (previous tokens) |
| **Output** | Contextualized audio representations | Next token probability |

### A.2 Multitask Token Format

Whisper xử lý nhiều task trong cùng 1 model thông qua **special tokens**:

```
<|startoftranscript|> → <|language|> → <|task|> → <|timestamps?|> → [transcript tokens] → <|endoftranscript|>
```

| Token | Ví dụ | Chức năng |
|---|---|---|
| `<|startoftranscript|>` | — | Bắt đầu sequence |
| `<|language|>` | `<|en|>`, `<|vi|>` | Xác định ngôn ngữ |
| `<|task|>` | `<|transcribe|>`, `<|translate|>` | Transcribe hoặc dịch sang English |
| `<|notimestamps|>` | — | Tắt timestamp prediction |
| Timestamp tokens | `<|0.00|>`, `<|0.50|>` | Đánh dấu thời gian (bội 20ms) |

**Ý nghĩa thiết kế:** Một model duy nhất xử lý ASR, translation, language ID, và timestamp prediction mà không cần model riêng.

### A.3 Whisper Model Variants — Chi tiết đầy đủ

| Model | Params | Encoder Layers | Decoder Layers | d_model | Attention Heads | Mel Bins |
|---|---|---|---|---|---|---|
| **tiny** | 39M | 4 | 4 | 384 | 6 | 80 |
| **base** | 74M | 6 | 6 | 512 | 8 | 80 |
| **small** | 244M | 12 | 12 | 768 | 12 | 80 |
| **medium** | 769M | 24 | 24 | 1024 | 16 | 80 |
| **large-v1/v2** | 1.55B | 32 | 32 | 1280 | 20 | 80 |
| **large-v3** | 1.55B | 32 | 32 | 1280 | 20 | **128** |
| **large-v3-turbo** | 809M | 32 | **4** | 1280 | 20 | 128 |

**Ghi chú quan trọng:**
- large-v3 nâng Mel bins từ 80→128, cải thiện frequency resolution
- large-v3-turbo giữ nguyên encoder 32 layers nhưng prune decoder 32→4 layers → 8x faster
- `.en` variants (tiny.en, base.en, small.en, medium.en) chỉ English, accuracy tốt hơn multilingual cho English
- Turbo **không hỗ trợ translation task**, chỉ transcribe

### A.4 Training Data — 680,000 Hours Weak Supervision

**Composition:**
| Loại | Giờ | Tỷ lệ | Mô tả |
|---|---|---|---|
| English ASR | ~438,000 | 65% | English audio → English text |
| X→English Translation | ~125,000 | 18% | Non-English audio → English text |
| Multilingual ASR | ~117,000 | 17% | Non-English audio → native text |

**"Weak Supervision" có nghĩa:**
- Transcripts thu thập tự động từ internet (subtitles, closed captions)
- **Không** human-verified — chất lượng không đồng đều
- Filtered bằng heuristics: punctuation, capitalization, language ID
- Loại bỏ machine-generated transcripts (auto-generated YouTube captions)

**Large-v3 mở rộng:**
- ~1M hours weakly labeled audio
- +4M hours pseudo-labeled (generated bởi Whisper versions trước)
- Tổng: ~5M hours training data

---

## PHỤ LỤC B: Audio Feature Extraction Chi tiết

### B.1 Log-Mel Spectrogram Pipeline

```
Raw Audio → Resample 16kHz → Frame (25ms window, 10ms hop) → Hann Window → STFT → |Magnitude|² → Mel Filter Bank → log() → 80/128-channel Log-Mel Spectrogram
```

#### Các bước chi tiết:

**1. Framing (Windowing):**
- Chia audio thành frames 25ms (400 samples @ 16kHz)
- Overlap: hop_size = 10ms (160 samples) → 63% overlap
- Áp dụng Hann window function để giảm spectral leakage

**2. STFT (Short-Time Fourier Transform):**
- FFT size: N_FFT = 400 (hoặc 512 với zero-padding)
- Output: Complex-valued matrix [n_fft/2 + 1, n_frames]
- Lấy magnitude squared: |X(f)|²

**3. Mel Filter Bank:**
- 80 triangular filters (Whisper v1/v2) hoặc 128 (v3)
- Phân bố theo **Mel scale**: f_mel = 2595 × log₁₀(1 + f/700)
- Nhấn mạnh dải tần thấp (nơi speech intelligibility cao nhất)
- Human ear phân biệt tần số thấp tốt hơn tần số cao → Mel scale mô phỏng điều này

**4. Logarithmic Compression:**
- log(mel_energies + 1e-10)
- Nén dynamic range, phù hợp với cách tai người cảm nhận loudness (logarithmic)

#### Tại sao Log-Mel Spectrogram?

| Lựa chọn | Ưu điểm | Nhược điểm |
|---|---|---|
| Raw waveform | Không mất thông tin | Sequence rất dài, khó train |
| MFCC | Compact, decorrelated | Mất phase info, ít phù hợp DL |
| **Log-Mel** | **Biomimetic, compact, proven** | **Mất phase** |
| Complex spectrogram | Giữ phase | Dimension lớn |

---

## PHỤ LỤC C: Silero VAD — Kiến trúc Chi tiết

### C.1 Architecture Internals

```
16kHz PCM Audio → STFT Feature Extraction → Encoder → RNN Decoder (stateful) → Speech Probability [0.0, 1.0]
```

**Core Components:**
1. **Feature Extraction:** STFT → frequency-domain representation
2. **Encoder:** Trích xuất acoustic representations từ spectral features
3. **RNN Decoder:** Stateful — duy trì hidden states (h, c) shape (2, batch, 64)
4. **Output:** Probability 0.0→1.0 cho mỗi chunk

### C.2 Operational Parameters

| Parameter | Giá trị | Ý nghĩa |
|---|---|---|
| Input sample rate | 16kHz hoặc 8kHz | 16kHz recommended |
| Chunk size | 512 samples (32ms @ 16kHz) | Mỗi inference call xử lý 1 chunk |
| Model size | ~2MB (ONNX) | Siêu nhẹ cho mobile |
| Inference latency | <1ms/chunk (single CPU) | Real-time capable |
| Hidden state shape | (2, batch, 64) | Phải pass giữa các calls |

### C.3 Hysteresis State Machine (Production Implementation)

**Vấn đề:** Raw probability output bị "flickering" — dao động nhanh giữa speech/silence trong natural pauses.

**Giải pháp: Dual-threshold hysteresis**

```
                    speech_threshold = 0.5
                         ┌─────────┐
          ┌──────────────│ SPEAKING │◄────────────────┐
          │              └─────────┘                  │
          │ prob < silence_threshold                   │ prob > speech_threshold
          │ for min_silence_duration                   │
          ▼                                           │
     ┌──────────┐                                ┌────────┐
     │ SILENCE  │────────────────────────────────►│ ONSET  │
     └──────────┘   prob > speech_threshold       └────────┘
                    silence_threshold = 0.35
```

**Tuning Parameters:**
| Parameter | Default | Mô tả |
|---|---|---|
| speech_threshold | 0.5 | Prob > threshold → detect speech |
| silence_threshold | 0.35 | Prob < threshold → detect silence |
| min_speech_duration | 250ms | Bỏ qua speech segments < 250ms |
| min_silence_duration | 300ms | Bỏ qua silence gaps < 300ms |
| speech_pad | 30ms | Padding trước/sau speech segment |

**Pre-speech padding:** Buffer 30ms audio trước trigger point → tránh cắt mất đầu từ.

---

## PHỤ LỤC D: Word-level Timestamp — Chi tiết Kỹ thuật 3 Phương pháp

### D.1 Phương pháp 1: Cross-Attention + DTW (Whisper Native)

**Nguyên lý:** Cross-attention weights trong Decoder cho biết model đang "nhìn" vào phần nào của audio khi predict mỗi token.

**Pipeline chi tiết:**

```
1. Inference Whisper → Thu thập cross-attention maps từ decoder layers
2. Chọn "alignment heads" (subset attention heads có correlation cao với timing)
3. Average attention maps across selected heads
4. Column-normalize attention matrix
5. Tạo cost matrix (high attention → low cost)
6. Áp dụng Dynamic Time Warping (DTW) → optimal alignment path
7. Extract start/end timestamps cho mỗi token từ DTW path
```

**DTW (Dynamic Time Warping):**
- Input: Cost matrix [n_tokens × n_audio_frames]
- Output: Optimal path mapping tokens → audio frames
- Constraint: Path phải monotonically increasing (tokens follow temporal order)
- Complexity: O(n_tokens × n_frames)

**Libraries implement:**
- `whisper-timestamped` (linto-ai) — popular, VAD integration
- `stable-ts` — robust long-form handling
- `CrisperWhisper` (nyrahealth) — fine-tuned for precision

**Accuracy:** ±100-200ms — đủ cho most caption applications

### D.2 Phương pháp 2: Forced Alignment bằng wav2vec 2.0 (WhisperX)

**Nguyên lý:** Dùng CTC-based phoneme model (wav2vec 2.0) để align transcript text với audio waveform.

```
1. Whisper transcribe → text output (segment-level timestamps)
2. wav2vec 2.0 phoneme model load (language-specific)
3. Text → phoneme sequence
4. Audio → wav2vec 2.0 encoder → frame-level phoneme probabilities
5. CTC forced alignment: map phoneme sequence → audio frames
6. Aggregate phoneme-level → word-level timestamps
```

**Tại sao chính xác hơn:**
- wav2vec 2.0 được train specifically cho phoneme recognition
- CTC alignment là deterministic, không phụ thuộc vào attention quality
- Frame-level resolution: 20ms per frame → accuracy ±20-50ms

**Hạn chế:**
- Cần model wav2vec 2.0 riêng cho mỗi ngôn ngữ
- Thêm ~300MB model size
- Inference time tăng ~30-50%

### D.3 Phương pháp 3: E2E Timestamp Tokens (Whisper Built-in)

**Nguyên lý:** Whisper có thể emit timestamp tokens trực tiếp trong output sequence.

```
Output: <|0.00|> Hello <|0.78|> world <|1.15|> <|1.50|> how <|1.82|> are <|2.10|> you <|2.45|>
```

- Timestamp tokens là bội số của 20ms
- Resolution: 20ms granularity
- **Segment-level** mặc định, **word-level** kém chính xác hơn
- Không cần model phụ
- Accuracy: ±200-500ms (kém nhất trong 3 phương pháp)

### D.4 So sánh 3 phương pháp

| Tiêu chí | Cross-Attn + DTW | Forced Alignment (WhisperX) | E2E Timestamps |
|---|---|---|---|
| **Accuracy** | ±100-200ms | **±20-50ms** | ±200-500ms |
| **Thêm model** | Không | wav2vec 2.0 (~300MB) | Không |
| **Compute overhead** | Moderate | High (+30-50%) | **Minimal** |
| **Multilingual** | Tốt | Cần model/ngôn ngữ | Tốt |
| **Production-ready** | ✅ | ✅ (WhisperX) | ⚠️ Kém chính xác |
| **Best for** | General use | **Karaoke/precision** | Quick & dirty |

**Khuyến nghị cho Auto Caption:**
- **PoC:** Cross-Attention + DTW (đơn giản, không cần model phụ)
- **Production:** Forced Alignment via WhisperX (precision cho karaoke effect)

---

## PHỤ LỤC E: Whisper Hallucination — Vấn đề & Giải pháp

### E.1 Root Cause Analysis

Whisper là **generative model** — khi gặp silence/noise, nó cố "fill in" bằng text phổ biến từ training distribution.

**Triệu chứng:**
1. **Repetition loops:** "Thank you. Thank you. Thank you..." lặp vô hạn
2. **Phantom text:** Generate text không tồn tại trong audio
3. **Common hallucinations:** "Thanks for watching", "Subscribe", "Like and share"
4. **Language switching:** Đang transcribe tiếng Việt bỗng chuyển sang English

**Nguyên nhân gốc:**
- Silence segments → decoder không có acoustic signal → dựa vào language model prior
- Training data chứa nhiều YouTube content → model memorize common phrases
- Low-confidence regions → decoder "drift" theo most-probable completion

### E.2 Mitigation Strategy — 4 Layers

#### Layer 1: Pre-processing (Phòng ngừa)
```
Audio → Silero VAD → Loại bỏ silence/noise segments → Chỉ feed speech segments vào Whisper
```
**Đây là biện pháp hiệu quả nhất.** VAD pre-filter ngăn Whisper tiếp xúc với silence.

#### Layer 2: Inference Parameter Tuning

| Parameter | Giá trị khuyến nghị | Tác dụng |
|---|---|---|
| `beam_size` | 1 (greedy) | Ngăn decoder commit vào repetitive sequences |
| `temperature` | 0.0 (deterministic) | Giảm randomness |
| `temperature_fallback` | [0.0, 0.2, 0.4, 0.6] | Retry với temperature tăng dần nếu fail |
| `compression_ratio_threshold` | 2.4 | Reject segments có compression ratio cao (repetitive) |
| `logprob_threshold` | -1.0 | Reject segments có avg log-prob thấp |
| `no_speech_threshold` | 0.6 | Probability threshold cho "no speech" detection |

#### Layer 3: Post-processing Heuristics
1. **Repetition detection:** Nếu cùng phrase lặp >3 lần liên tiếp → remove
2. **Hallucination blocklist:** Maintain list known hallucinations ("Thank you for watching", etc.)
3. **Duration sanity check:** Nếu segment duration / word count → words-per-second quá cao/thấp → flag
4. **Context reset:** Clear previous-text context khi detect loop

#### Layer 4: Research-level (2024-2025)
- **Calm-Whisper:** Fine-tune specific self-attention heads responsible cho noise-induced hallucinations
- **ALA (Adaptive Layer Attention):** Multi-objective knowledge distillation cho robust decoder
- **Prompt engineering:** System prompt hướng model output `(music)`, `(inaudible)` thay vì hallucinate

---

## PHỤ LỤC F: ASR Post-Processing Pipeline

### F.1 Pipeline hoàn chỉnh sau ASR

```
Whisper Output (raw text) → Punctuation Restoration → Capitalization → Inverse Text Normalization → Profanity Filter → Caption Segmentation → Final Output
```

### F.2 Punctuation Restoration

**Vấn đề:** Whisper large models output có punctuation, nhưng tiny/base thường thiếu hoặc sai.

**Giải pháp:**
- **Whisper built-in:** Large models khá tốt cho punctuation
- **Dedicated model:** `deepmultilingualpunctuation` (Oliver Guhr) — BERT-based, ~500MB
- **Rule-based fallback:** Heuristics dựa trên pause duration từ timestamps

### F.3 Inverse Text Normalization (ITN)

**Mục đích:** Chuyển spoken form → written form cho readability

| Spoken Form | Written Form | Semiotic Class |
|---|---|---|
| "one hundred twenty three" | "123" | Cardinal |
| "january fifth twenty twenty six" | "January 5, 2026" | Date |
| "ten dollars and fifty cents" | "$10.50" | Currency |
| "three point one four" | "3.14" | Decimal |
| "hashtag trending" | "#trending" | Verbatim |

**Tools:**
- **NVIDIA NeMo-text-processing:** WFST-based (rule-based, deterministic) — production standard
- **Neural ITN:** Transformer-based tagger + decoder (NeMo Duplex) — more flexible

**Cho Whisper:** Large models đã handle ITN khá tốt internally. Tiny/base cần external ITN.

### F.4 Caption Segmentation

**Vấn đề:** ASR output là continuous text. Caption cần chia thành segments phù hợp.

**Rules cho caption segmentation:**
| Rule | Giá trị | Lý do |
|---|---|---|
| Max characters/line | 42 (NTSC) hoặc 37 (PAL) | Đọc kịp |
| Max lines | 2 | Không che quá nhiều video |
| Min duration | 1 giây | Đủ thời gian đọc |
| Max duration | 7 giây | Tránh stale captions |
| Reading speed | 15-20 chars/second | Comfortable reading |
| Break tại | Clause boundaries, punctuation | Natural reading flow |

---

*Tiếp tục Part 2...*
