# DGX Spark 코딩 모델 추천

**작성일**: 2026-04-26
**대상 하드웨어**: NVIDIA DGX Spark (GB10 Grace Blackwell, 128GB unified memory)

## 1. DGX Spark 핵심 스펙 (LLM 관점)

| 항목 | 값 | 영향 |
|---|---|---|
| GPU 아키텍처 | Blackwell (GB10) | **NVFP4 하드웨어 가속** ⭐ |
| 통합 메모리 | 122 GB LPDDR5X | 35B BF16 풀 로드 가능 |
| 메모리 대역폭 | **273 GB/s** | **MoE가 Dense 압승** |
| AI 성능 | 1 PFLOP FP4 sparse | NVFP4에서 풀 스피드 |
| 최대 모델 크기 | ~200B 파라미터 | 200B 이하 자유롭게 |
| 클러스터링 | 200 Gbps QSFP × 2 | 두 대 연결 시 400B+ 가능 |

**3가지 결정적 포인트**:
1. **NVFP4 네이티브 가속** → NVFP4 양자화가 최적
2. **273 GB/s 대역폭 제약** → 디코딩 속도 ∝ 활성 파라미터 → MoE 압승
3. **122GB 메모리** → 여러 모델 동시 로드 가능

## 2. 디코딩 속도 추정 (273 GB/s 기준)

| 모델 | 양자화 | 활성 파라미터 메모리 | 이론 tok/s | 실측 추정 |
|---|---|---|---|---|
| Qwen3.6-27B | BF16 | 54 GB | 5 | ~4-6 |
| Qwen3.6-27B | MXFP8 | 27 GB | 10 | ~8-12 |
| Qwen3.6-27B | NVFP4 | 13.5 GB | 20 | ~18-25 |
| **Qwen3.6-35B-A3B** | **NVFP4** | **1.5 GB (활성 3B)** | **180** | **~80-120** ⚡ |
| Gemma 4 31B | NVFP4 | 15.5 GB | 18 | ~15-20 |
| Gemma 4 26B-A4B | NVFP4 | 2 GB (활성 4B) | 137 | ~60-90 ⚡ |

→ **MoE + NVFP4 조합이 5~10배 빠름**

## 3. 추천 모델 (워크로드별)

### 🥇 1순위: `qwen3.6:35b-a3b-coding-nvfp4` (22 GB)

- ✅ NVFP4 하드웨어 가속 (Blackwell 네이티브)
- ✅ 활성 3B MoE → 디코딩 80~120 tok/s
- ✅ SWE-bench 70%, LiveCodeBench 70%
- ✅ 22GB만 점유 → 100GB로 1M 컨텍스트 KV 캐시 가능
- ✅ Agentic + 자동완성 모두 빠르게 처리

### 🥈 2순위: `qwen3.6:27b-coding-mxfp8` (31 GB)

- ✅ **SWE-bench 76% — 오픈소스 최강 agentic 코딩**
- ✅ MXFP8 Blackwell 가속
- ✅ 31GB → KV 캐시/배치/멀티모델 여유
- ❌ Dense 27B → 디코딩 8~12 tok/s

### 🥉 3순위: `gemma4:31b-nvfp4` (20 GB)

- ✅ NVFP4 하드웨어 가속
- ✅ 이미지/비디오 입력 지원 (UI 스크린샷 디버깅)
- ✅ LiveCodeBench 77% (알고리즘)
- ❌ SWE-bench 61% (agentic 약함)

## 4. 추천 구성: 3-모델 동시 로드

128GB 메모리 활용 극대화:

| 역할 | 모델 | 스토리지 | 용도 |
|---|---|---|---|
| 🔧 Agentic (PR 작성) | `qwen3.6:27b-coding-mxfp8` | 31 GB | 정확도 최우선 |
| ⚡ 자동완성 (IDE) | `qwen3.6:35b-a3b-coding-nvfp4` | 22 GB | 100+ tok/s |
| 🖼️ 멀티모달 | `gemma4:31b-nvfp4` | 20 GB | 스크린샷 디버깅 |
| **합계** | | **73 GB** | **49GB KV 캐시 여유** |

### 설치 명령

```bash
ollama pull qwen3.6:35b-a3b-coding-nvfp4
ollama pull qwen3.6:27b-coding-mxfp8
ollama pull gemma4:31b-nvfp4
```

## 5. 한 줄 결론

> **단일 추천 → `qwen3.6:35b-a3b-coding-nvfp4` (22GB)**
>
> Spark의 강점(NVFP4 HW 가속 + 통합 메모리)을 최적 활용. MoE 활성 3B + NVFP4 네이티브로 디코딩 100+ tok/s, agentic 코딩 SWE-bench 70% 확보.
>
> **여유 있다면 + `qwen3.6:27b-coding-mxfp8`** (31GB) 추가. 정확도 최우선 워크로드 위임.

## 6. DGX Spark 활용 팁

1. **2대 클러스터링**: 200 Gbps QSFP로 두 대 연결 → 600B+ 모델 (Qwen3-Coder-480B 등)
2. **배치 추론**: 단일 쿼리 5~10 tok/s지만 배치 시 처리량 5~10배 → 백엔드 서버용
3. **롱컨텍스트**: 122GB로 1M 토큰 KV 캐시 → 대규모 코드베이스 통째 분석
4. **NVFP4 우선**: 다른 양자화 대비 2~3배 빠름. 일반 GPU와 Spark의 차이는 NVFP4 HW 가속

## 7. 주의

- `:cloud` 태그(예: `deepseek-v4-flash:cloud`)는 Spark에서 의미 없음 — 로컬 자원 활용 못 함
- BF16 풀 프리시전(55~70GB)은 메모리는 들어가지만 **디코딩 매우 느림** (5 tok/s)
- DGX Spark는 **대역폭 제약** > 컴퓨트 제약 → MoE/양자화 선택이 결정적
- 모델/양자화 성능은 본 레포(`ollama-benchmark`)로 환경에서 직접 측정 권장

## 출처

- [DGX Spark Hardware Overview — NVIDIA](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)
- [DGX Spark In-Depth Review — LMSYS](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark/)
- [Gemma 4 on DGX Spark — NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/gemma-4-day-1-inference-on-nvidia-dgx-spark-preliminary-benchmarks/365503)
- [DGX Spark llama.cpp Performance](https://github.com/ggml-org/llama.cpp/discussions/16578)
- [Gemma 4 vs Qwen 3.5 on DGX Spark](https://github.com/nabe2030/gemma4-vs-qwen35-dgx-spark)
