# 추론 런타임 옵션 비교 (DGX Spark)

DGX Spark에서 LLM을 돌리는 방법은 여러 가지. 각자 트레이드오프가 다름.

## 한눈에 비교

| 런타임 | 양자화 지원 | NVFP4 HW 가속 | 셋업 난이도 | 적합 워크로드 |
|---|---|---|---|---|
| **Ollama** | GGUF (Q4/Q8/BF16) | ❌ | 매우 쉬움 (`ollama pull`) | 개발/에이전트, 빠른 시작 |
| **llama.cpp** | GGUF | ❌ | 쉬움 (직접 빌드 가능) | 임베디드, 커스텀 백엔드 |
| **vLLM** ≥ 0.6 | FP16/BF16/FP8/INT8/**NVFP4** | ✅ | 중간 (Python 환경) | 서빙, 배칭, 처리량 |
| **TensorRT-LLM** | FP16/INT8/FP8/**NVFP4** | ✅ (최적) | 어려움 (모델 변환) | 프로덕션, 최고 처리량 |
| **HuggingFace + bitsandbytes** | NF4/FP4 (제한적) | 일부 | 매우 쉬움 | 실험, 파인튜닝 |

## Ollama (현재 셋업)

✅ 본 연구에서 사용 — 가장 빠르게 코딩 어시스턴트 돌리기 좋음

```bash
ollama pull qwen3.6:35b
ollama run qwen3.6:35b "Write a Python function..."
```

**장점**:
- 한 줄로 설치/실행
- OpenAI 호환 API (`http://localhost:11434/v1`)
- IDE/에이전트 통합 풍부 (Cursor, Cline, Continue, Zed, OpenClaw 등)

**단점**:
- GGUF만 → NVFP4 미지원 → Blackwell 5세대 텐서코어 활용 못 함
- 배칭 약함 (단일 요청 위주)
- 멀티 GPU 분산 어려움

## vLLM (NVFP4 하드웨어 가속을 원할 때)

```bash
pip install vllm
vllm serve nvidia/Qwen3.6-35B-A3B-NVFP4 --max-model-len 32768
```

**장점**:
- NVFP4/FP8 등 Blackwell 네이티브 양자화 지원
- PagedAttention → 동시 요청 처리량 매우 높음
- OpenAI 호환 API

**단점**:
- 모델 가중치를 직접 받아야 (HF Hub의 `*-NVFP4`, `*-FP8` 변형)
- Python 환경 관리 필요
- Cold start 느림

**언제 쓰나**: 여러 사용자/에이전트를 동시에 서빙할 때, 또는 처리량(throughput)이 핵심일 때.

## TensorRT-LLM (최고 처리량)

```bash
git clone https://github.com/NVIDIA/TensorRT-LLM
# 모델 변환 → Engine 빌드 → 서빙
```

**장점**:
- DGX Spark Blackwell 최대 활용 (FP4 PFLOP 풀 사용)
- 최고 처리량/저지연
- NVIDIA 공식 지원

**단점**:
- 모델 변환 단계 필요 (HF → TRT engine)
- 빌드 시간 30분~수 시간
- 디버깅 어려움
- 모델 업데이트 시 재빌드

**언제 쓰나**: 단일 모델로 프로덕션 서빙할 때, 응답 지연이 결정적일 때.

## llama.cpp (Ollama의 기반)

**Ollama와 동일 백엔드**. 직접 빌드해서 더 세밀한 제어 가능:
- `--n-gpu-layers` 등 명시적 오프로드
- 실험적 기능(MoE 라우팅 디버깅 등) 사용 가능
- 컨테이너/임베디드에 가벼움

대부분의 사용자는 Ollama로 충분. llama.cpp는 디버깅/연구 시.

## 실용 가이드: DGX Spark에서 어떻게 시작?

### Phase 1: 빠른 시작 (현재 단계)
- **Ollama**로 코딩 모델 운용
- IDE/에이전트 통합 (Cursor, OpenClaw, Cline 등)
- 11~58 tok/s 정도로 충분 (단일 사용자)

### Phase 2: 처리량 필요 시
- **vLLM**으로 마이그레이션
- HF에서 NVFP4 가중치 받기 (예: `nvidia/Qwen3.6-35B-A3B-NVFP4`)
- 5세대 텐서코어 가속 활용 → 처리량 2~3배

### Phase 3: 프로덕션 서빙
- **TensorRT-LLM** 엔진 빌드
- Triton Inference Server와 결합
- 다중 클라이언트 동시 서빙

## 본 연구의 선택: Ollama

이유:
1. DGX Spark를 **개인 워크스테이션**으로 사용 → 단일 사용자 가정
2. **OpenClaw 에이전트 통합**이 핵심 사용 사례 → OpenAI API 호환 필수
3. 모델 교체/실험이 잦음 → `ollama pull` 한 줄이 최적
4. NVFP4 가속 못 받지만 Q4_K_M GGUF로도 코딩 워크로드 충분 (실측 11~58 tok/s)

향후 처리량 한계 도달 시 vLLM으로 점진적 이행 가능.

## 출처

- [vLLM Documentation](https://docs.vllm.ai/)
- [TensorRT-LLM GitHub](https://github.com/NVIDIA/TensorRT-LLM)
- [llama.cpp DGX Spark Discussion](https://github.com/ggml-org/llama.cpp/discussions/16578)
- [DGX Spark 활용 NVIDIA Blog](https://developer.nvidia.com/blog/how-nvidia-dgx-sparks-performance-enables-intensive-ai-tasks/)
