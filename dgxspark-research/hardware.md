# DGX Spark 하드웨어 (LLM 추론 관점)

## GB10 Grace Blackwell Superchip

| 항목 | 값 |
|---|---|
| CPU | 20 코어 (10× Cortex-X925 + 10× Cortex-A725) |
| GPU 아키텍처 | Blackwell, 5세대 텐서 코어 |
| AI 성능 | **1 PFLOP FP4 sparse**, ~1000 AI TOPS |
| 통합 메모리 | **122 GB LPDDR5X** (CPU/GPU 공유 주소 공간) |
| 메모리 대역폭 | **273 GB/s** |
| 네트워킹 | 2× QSFP, 200 Gbps aggregate (2대 클러스터링 가능) |
| 최대 모델 | ~200B 파라미터 (단일), 600B+ (2대 연결) |

## LLM 추론 관점 의미

### 1. 디코딩은 메모리 대역폭 제약

자기회귀 디코딩(autoregressive)은 토큰당 모델 파라미터 한 번 전부 읽어야 하므로:

```
디코딩 tok/s ≈ (메모리 대역폭) / (활성 파라미터 × 바이트/파라미터)
```

DGX Spark의 273 GB/s 기준 이론 최대값:

| 모델 종류 | 양자화 | 활성 메모리 | 이론 tok/s |
|---|---|---|---|
| 27B Dense | BF16 | 54 GB | 5 |
| 27B Dense | Q4_K_M | 13.5 GB | 20 |
| 35B-A3B MoE | BF16 | 6 GB (활성 3B) | 45 |
| 35B-A3B MoE | Q4_K_M | 1.5 GB | **180** |

→ **MoE가 Dense보다 압도적으로 빠름**. 본 연구의 실측에서도 35B-A3B (~57 tok/s)가 27B Dense (~11 tok/s)보다 5배 빠름.

### 2. Prefill은 컴퓨트 제약 (1 PFLOP FP4)

긴 프롬프트 처리(prefill)는 행렬 곱 → 컴퓨트 바운드. Blackwell의 1 PFLOP FP4가 빛나는 구간. 다만 Ollama Linux는 GGUF만 사용해 NVFP4 가속을 못 받음.

### 3. NVFP4 5세대 텐서 코어

- BF16 대비 4배 처리량
- 단, **소프트웨어가 받쳐줘야** 활용 가능
- Ollama Linux: ❌ (GGUF만 지원)
- vLLM ≥ 0.6: ✅
- TensorRT-LLM: ✅ (NVIDIA 최적)

### 4. 통합 메모리 (Unified Memory)

- CPU↔GPU 데이터 복사 불필요 → 모델 로드 빠름
- 122 GB로 단일 GPU 워크스테이션이 못 다루는 모델 가능
- 다만 **VRAM 전용 GPU 대비 대역폭은 1/10** (HBM3은 ~3 TB/s vs Spark 273 GB/s)
- → "큰 모델 굴릴 수 있지만 빠르진 않다"

## 메모리 예산 계획

122 GB 안에서 모델 + KV 캐시를 모두 담아야:

| 컨텍스트 | 27B Dense Q4 KV 캐시 | 31B Dense Q4 KV 캐시 |
|---|---|---|
| 4K | ~0.5 GB | ~0.6 GB |
| 32K | ~4 GB | ~5 GB |
| 128K | ~16 GB | ~20 GB |
| 256K | **~33 GB** | **~40 GB** |
| 1M | ~130 GB ❌ | ~160 GB ❌ |

**경험칙**: 모델 사이즈 + (컨텍스트 × ~80 KB) ≤ 100 GB로 유지.

본 연구 중 발견한 함정: gemma4:31b를 기본 256K 컨텍스트로 띄우면 KV 캐시가 메모리 초과 → 500 에러. **`num_ctx`를 워크로드에 맞게 명시 권장** (예: 코딩은 4K~32K로 충분).

## 클러스터링 (선택적)

- 200 Gbps QSFP × 2 → 두 대 연결 시 600B+ 모델 가능 (예: Qwen3-Coder-480B)
- 분산 추론 latency가 추가되므로 단일 노드 대비 처리량 손실 있음
- 출처: [DGX Spark Playbooks - Connect Two Sparks](https://github.com/NVIDIA/dgx-spark-playbooks)

## 출처

- [DGX Spark Hardware Overview — NVIDIA](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)
- [DGX Spark In-Depth Review — LMSYS](https://www.lmsys.org/blog/2025-10-13-nvidia-dgx-spark/)
- [How DGX Spark Enables Intensive AI Tasks — NVIDIA Blog](https://developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks/)
- 본 연구의 실측 (`ttft-results.md`)
