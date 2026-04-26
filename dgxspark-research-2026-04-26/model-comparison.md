# 오픈소스 코딩 LLM 비교: Qwen3.6 vs Gemma 4

**작성일**: 2026-04-26
**최종 수정**: 2026-04-26

> Ollama 라이브러리에서 로컬 실행 가능한 최신 코딩 특화 오픈 모델 비교 정리.

## 1. 후보 모델 개요

| 항목 | Qwen3.6 | Gemma 4 | DeepSeek-V4-Flash |
|---|---|---|---|
| 개발사 | Alibaba | Google DeepMind | DeepSeek |
| 라이선스 | Apache 2.0 | Apache 2.0 | MIT |
| 출시 | 2026-04 | 2026-04-02 | 2026-04-24 |
| 로컬 실행 | ✅ | ✅ | ❌ (Ollama 클라우드 전용) |
| 본 비교 대상 | ✅ | ✅ | ❌ |

DeepSeek-V4-Flash는 284B/13B MoE로 Ollama에 `:cloud` 태그만 존재 → **로컬 비교에서 제외**.

## 2. 베이스 모델 사양

### Qwen3.6
| 모델 | 총 파라미터 | 활성 파라미터 | 아키텍처 | 컨텍스트 |
|---|---|---|---|---|
| Qwen3.6-27B | 27B | 27B | Dense | 256K (~1M 확장) |
| Qwen3.6-35B-A3B | 35B | **3B** | MoE | 256K |

### Gemma 4
| 모델 | 총 파라미터 | 활성 파라미터 | 아키텍처 | 컨텍스트 |
|---|---|---|---|---|
| Gemma 4 E2B | 2B | 2B | Dense (모바일) | 128K |
| Gemma 4 E4B | 4B | 4B | Dense (엣지) | 128K |
| Gemma 4 26B-A4B | 26B | **4B** | MoE | 256K |
| Gemma 4 31B | 31B | 31B | Dense | 256K |

## 3. 코딩 벤치마크 (BF16 풀 프리시전 기준)

| 벤치마크 | Qwen3.6-27B | Qwen3.6-35B-A3B | Gemma 4 31B | Gemma 4 26B-A4B |
|---|---|---|---|---|
| **SWE-bench Verified** | **77.2%** | 73.4% | ~64% | ~58%* |
| **SWE-bench Pro** | **53.5%** | ~50%* | ~40%* | ~38%* |
| **LiveCodeBench v6** | ~75%* | ~72%* | **80.0%** | 77.1% |
| **HumanEval** | ~89%* | ~87%* | ~88% | ~85%* |
| **Codeforces Elo** | ~2050* | ~1950* | **2150** | ~1900* |
| **Terminal-Bench 2.0** | **59.3%** | ~55%* | ~45%* | ~42%* |

\* 별표 = 미공개 추정치

**패턴**:
- Agentic / 실전 코딩(SWE-bench, Terminal-Bench) → **Qwen3.6** 압승
- 알고리즘 / 경쟁 프로그래밍(LiveCodeBench, Codeforces) → **Gemma 4** 우위
- 함수 단위 코딩(HumanEval) → 비슷

## 4. 양자화별 스토리지 환산

| 양자화 | 비트/param | 27B 이론 | 31B 이론 | 비고 |
|---|---|---|---|---|
| BF16 | 16 bit | 54 GB | 62 GB | 풀 프리시전 |
| Q8_0 / MXFP8 | 8 bit | 27 GB | 31 GB | 거의 무손실 |
| Q4_K_M | ~4.5 bit | ~15 GB | ~17 GB | 일반 GGUF |
| NVFP4 | 4 bit | 13.5 GB | 15.5 GB | Blackwell 가속 |

> 실측 Ollama 사이즈는 메타데이터/임베딩으로 1~6GB 더 큼.

## 5. GPU 예산별 추천 (스토리지 구간 비교)

### 24GB GPU (RTX 4090/3090) — 격전지

| 태그 | 베이스 (총/활성) | 양자화 | 스토리지 | SWE-bench | LiveCodeBench |
|---|---|---|---|---|---|
| `qwen3.6:27b` | 27B / 27B | Q4_K_M | **17 GB** | **~74%** | ~72% |
| `gemma4:26b-nvfp4` | 26B / **4B** | NVFP4 | 17 GB | ~55% | ~74% |
| `qwen3.6:27b-coding-nvfp4` | 27B / 27B | NVFP4 | 20 GB | **~75%** | ~73% |
| `gemma4:31b-nvfp4` | 31B / 31B | NVFP4 | 20 GB | ~61% | **~77%** |
| `qwen3.6:35b-a3b-coding-nvfp4` | 35B / **3B** | NVFP4 | 22 GB | ~70% | ~70% |

### 32~48GB GPU

| 태그 | 베이스 (총/활성) | 양자화 | 스토리지 | SWE-bench | LiveCodeBench |
|---|---|---|---|---|---|
| `qwen3.6:27b-coding-mxfp8` | 27B / 27B | MXFP8 | 31 GB | **~76%** | ~74% |
| `gemma4:31b-mxfp8` | 31B / 31B | MXFP8 | 32 GB | ~63% | **~79%** |

### ≤16GB GPU / 엣지 (Gemma 4 단독)

| 태그 | 베이스 | 양자화 | 스토리지 |
|---|---|---|---|
| `gemma4:e4b-nvfp4` | 4B Dense | NVFP4 | **9.6 GB** |
| `gemma4:e2b-nvfp4` | 2B Dense | NVFP4 | **7.1 GB** |

> Qwen3.6는 이 구간에 모델 없음.

## 6. 워크로드별 최종 추천

| 워크로드 | 추천 |
|---|---|
| 🔧 Agentic 코딩 (PR 작성, repo 수정) | `qwen3.6:27b-coding-nvfp4` |
| 🏆 알고리즘 / 경쟁 코딩 | `gemma4:31b-nvfp4` |
| 🖼️ 멀티모달 (UI 스크린샷 → 코드) | `gemma4:31b-nvfp4` |
| 📚 롱컨텍스트 (>500K) | `qwen3.6:27b` (1M 확장) |
| ⚡ IDE 자동완성 (낮은 latency) | `qwen3.6:35b-a3b-coding-nvfp4` (활성 3B) |
| 📱 모바일 / 엣지 | `gemma4:e4b-nvfp4` |
| 🖥️ 24GB GPU 단일 | `qwen3.6:27b-coding-nvfp4` |
| 🏢 워크스테이션 (48GB+) | `qwen3.6:27b-coding-mxfp8` |

## 7. 핵심 인사이트

1. **파라미터 ≠ 스토리지**: 27B Dense라도 BF16(55GB) vs Q4(17GB)는 3배 차이
2. **MoE 메모리 함정**: 활성 파라미터가 작아도 **총 파라미터 전부 메모리 로드** 필요
3. **24GB GPU 최강**: `qwen3.6:27b-coding-nvfp4` (20GB, SWE 75%) — agentic 워크로드 기준
4. **Gemma 4의 다양성**: 2B~31B 풀 라인업 + 멀티모달 + 알고리즘 코딩 강점
5. **Qwen3.6의 깊이**: SWE-bench 13%p 우위 + 1M 컨텍스트 + thinking 모드

## 8. 한 줄 결론

> **24GB GPU 단일 환경에서 코딩 한 모델만 고른다면 → `qwen3.6:27b-coding-nvfp4` (20GB)**
> 멀티모달 / 16GB 이하 환경 / 알고리즘 워크로드라면 → **Gemma 4 라인업**
> 두 모델 다 받아두고 워크로드별로 스위치하는 것이 베스트 (총 ~30GB)

## 9. 주의사항

- 양자화별 점수는 **공식 미공개 추정치 다수 포함** → 본 레포(`ollama-benchmark`)로 직접 측정 권장
- NVFP4는 NVIDIA Blackwell(RTX 50, GB200)에서만 하드웨어 가속
- Qwen3.6는 기본 thinking 모드 → latency-sensitive 워크로드는 별도 고려
- 모든 데이터는 **2026-04-26** 기준, 모델 업데이트 시 재검증 필요

## 출처

- [Qwen3.6 · Ollama](https://ollama.com/library/qwen3.6)
- [Gemma 4 · Ollama](https://ollama.com/library/gemma4)
- [Qwen3.6-27B 리뷰 — buildfastwithai](https://www.buildfastwithai.com/blogs/qwen3-6-27b-review-2026)
- [Gemma 4 Benchmarks — Medium](https://medium.com/@moksh.9/heres-a-tighter-benchmark-focused-blog-post-501c5ea829f4)
- [SWE-bench & LiveCodeBench Leaderboard](https://benchlm.ai/coding)
- [Welcome Gemma 4 — HuggingFace](https://huggingface.co/blog/gemma4)
