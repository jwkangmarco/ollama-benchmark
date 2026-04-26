# DGX Spark 연구 노트

**작성일**: 2026-04-26
**대상 하드웨어**: NVIDIA DGX Spark (GB10 Grace Blackwell, 121 GB unified memory, Linux)

DGX Spark에서 로컬 코딩 LLM을 효과적으로 운용하기 위한 조사·실측·구성 자료 모음.

## 핵심 발견

1. **하드웨어 특성** — 통합 메모리 122 GB / 대역폭 273 GB/s / NVFP4 5세대 텐서코어. 디코딩은 **메모리 대역폭 제약** → MoE가 Dense보다 압도적으로 빠름.
2. **Ollama 양자화 함정** — `*-nvfp4`, `*-mxfp8`, `*-mlx-*` 태그는 **Apple Silicon MLX 전용**이며 DGX Spark Linux에서 풀 시 `412: this model requires macOS` 오류. GGUF (`q4_K_M`, `q8_0`) 또는 기본 태그를 써야 함.
3. **NVFP4 HW 가속의 진짜 경로** — Ollama Linux는 GGUF만 지원하므로 Blackwell NVFP4 가속을 받지 못함. **TensorRT-LLM** 또는 **vLLM**이 필요.
4. **모델 선택** — Ollama 기준 추천 단일 모델: **`qwen3.6:35b`** (35B-A3B MoE, 활성 3B). 정확도 우선이면 `qwen3.6:27b`.
5. **실측 TTFT/속도** — `qwen3.6:35b`가 `qwen3.6:27b`보다 **~5배 빠른 디코딩** (57.6 vs 11.5 tok/s) — 대역폭 제약 + MoE의 효과를 실증.

## 파일 인덱스

| 파일 | 내용 |
|---|---|
| [`hardware.md`](hardware.md) | DGX Spark GB10 스펙과 LLM 추론 관점의 의미 |
| [`quantization-platforms.md`](quantization-platforms.md) | Ollama 태그 접미사 의미 / MLX vs Linux 호환성 / 디버깅 로그 |
| [`runtime-options.md`](runtime-options.md) | Ollama vs vLLM vs TensorRT-LLM 비교 (DGX Spark 관점) |
| [`model-comparison.md`](model-comparison.md) | Qwen3.6 vs Gemma 4 일반 비교 (벤치마크/스토리지/구간별 추천) |
| [`recommendations.md`](recommendations.md) | DGX Spark 최종 모델 추천 + 설치 명령 |
| [`installed-setup.md`](installed-setup.md) | 실제 설치 결과 (Ollama 모델, OpenClaw 통합) |
| [`ttft-results.md`](ttft-results.md) | TTFT/생성 속도 실측 결과 |

## 빠른 시작

```bash
# Linux/Ollama 호환 모델 3종
ollama pull qwen3.6:27b   # 17GB, agentic 코딩 정확도 1위
ollama pull qwen3.6:35b   # 23GB, MoE A3B 빠른 추론
ollama pull gemma4:31b    # 19GB, 멀티모달 (Text+Image)
```

Primary 모델: `qwen3.6:35b` (속도/품질 균형, OpenClaw 기본).

## 주의

- 본 연구는 2026-04-26 시점 정보 (Qwen3.6, Gemma 4 출시 직후)
- 양자화별 정확도 점수에는 추정치가 포함됨 — 실제는 본 레포(`ollama-benchmark`)로 측정 권장
- Ollama 0.21.2 + GB10 환경에서 검증
