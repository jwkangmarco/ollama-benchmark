# Ollama 양자화 태그 / 플랫폼 호환성

본 연구 중 가장 큰 함정. Ollama 라이브러리에 등장하는 양자화 태그 접미사가 **플랫폼별로 호환성이 다르다**는 점을 사전 인지 못하면 다운로드 시점에 막힘.

## TL;DR

| 태그 패턴 | 백엔드 | DGX Spark Linux | macOS Apple Silicon |
|---|---|---|---|
| `:tag` (no suffix) | GGUF Q4_K_M (대개) | ✅ | ✅ |
| `:tag-it-q4_K_M` | GGUF | ✅ | ✅ |
| `:tag-it-q8_0` | GGUF | ✅ | ✅ |
| `:tag-it-bf16` | GGUF | ✅ | ✅ |
| `:tag-mlx-bf16` | **MLX** | ❌ 412 오류 | ✅ |
| `:tag-mxfp8` | **MLX** | ❌ 412 오류 | ✅ |
| `:tag-nvfp4` | **MLX** | ❌ 412 오류 | ✅ |

⚠️ **`-nvfp4` 태그명에 속지 말 것**: NVIDIA NVFP4 형식이 아니라 MLX 프레임워크의 4비트 변형이며 Apple Silicon 전용.

## 실제로 만난 오류

Ollama 0.21.2 / DGX Spark Linux에서:

```bash
$ ollama pull qwen3.6:35b-a3b-coding-nvfp4
pulling manifest 
Error: pull model manifest: 412: this model requires macOS

$ ollama pull qwen3.6:27b-coding-mxfp8
Error: pull model manifest: 412: this model requires macOS

$ ollama pull gemma4:31b-nvfp4
Error: pull model manifest: 412: this model requires macOS
```

## 권장 태그 (DGX Spark Linux 기준)

### Qwen 3.6
```bash
ollama pull qwen3.6:27b              # 17GB, Q4_K_M, Dense
ollama pull qwen3.6:35b              # 23GB, Q4_K_M, 35B-A3B MoE
# 더 높은 정확도:
ollama pull qwen3.6:27b-it-q8_0      # ~28GB
ollama pull qwen3.6:35b-a3b-it-q8_0  # ~38GB
```

### Gemma 4
```bash
ollama pull gemma4:31b               # 19GB, Q4_K_M, Dense
ollama pull gemma4:26b               # 18GB, Q4_K_M, MoE 4B 활성
ollama pull gemma4:e4b               # 9.6GB, 4B Dense (엣지)
ollama pull gemma4:e2b               # 7.2GB, 2B Dense (모바일)
# 더 높은 정확도:
ollama pull gemma4:31b-it-q8_0       # 34GB
```

## NVFP4 하드웨어 가속을 진짜 받으려면

DGX Spark의 **Blackwell 5세대 텐서 코어 NVFP4 가속**은 Ollama Linux 경로로는 활용 못 함. 다음 중 하나 사용:

| 옵션 | 장점 | 단점 |
|---|---|---|
| **TensorRT-LLM** | NVIDIA 공식 최적, 최고 처리량 | 빌드 복잡, 모델 변환 필요 |
| **vLLM** ≥ 0.6 | NVFP4 지원, 배칭/서빙 우수 | NVFP4 가중치 직접 수급 필요 |
| **HF + bitsandbytes** | 손쉬운 NF4/FP4 | 일부 가속만, 처리량 낮음 |

자세한 비교는 [`runtime-options.md`](runtime-options.md) 참고.

## 왜 Ollama가 NVFP4를 지원 안 하나?

- Ollama 백엔드는 **llama.cpp** (GGUF) — 현재 NVFP4 미지원
- llama.cpp의 CUDA 백엔드는 BF16/FP16/Q8/Q4까지만 지원
- 향후 llama.cpp가 NVFP4 도입하면 Ollama도 자동 지원 예상
- 추적: [llama.cpp issue tracker](https://github.com/ggml-org/llama.cpp/issues)

## 양자화 ↔ 스토리지 환산 (참고)

| 양자화 | 비트/param | 27B 이론 | 31B 이론 |
|---|---|---|---|
| BF16 / FP16 | 16 | 54 GB | 62 GB |
| Q8_0 / FP8 | 8 | 27 GB | 31 GB |
| Q4_K_M | ~4.5 | ~15 GB | ~17 GB |
| Q3_K_M | ~3.4 | ~11 GB | ~13 GB |

실제 Ollama 다운로드 사이즈는 메타데이터/임베딩으로 1~6 GB 더 큼.

## 교훈

1. **`-nvfp4`/`-mxfp8`/`-mlx-*` 태그는 무시** (DGX Spark Linux에서)
2. 기본 태그(`qwen3.6:27b`) 또는 명시적 GGUF (`-it-q4_K_M`, `-it-q8_0`) 사용
3. NVFP4 HW 가속 원하면 TensorRT-LLM/vLLM으로 우회

## 출처

- 본 연구의 412 오류 실측 로그 (Ollama 0.21.2)
- [Ollama Library — qwen3.6](https://ollama.com/library/qwen3.6)
- [Ollama Library — gemma4](https://ollama.com/library/gemma4)
- [llama.cpp — DGX Spark 호환성 논의](https://github.com/ggml-org/llama.cpp/discussions/16578)
