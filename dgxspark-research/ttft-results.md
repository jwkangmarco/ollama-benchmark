# TTFT 실측 결과 (DGX Spark)

**측정일**: 2026-04-26
**환경**: NVIDIA DGX Spark (GB10, 122 GB unified memory) / Ollama 0.21.2 / Linux
**프롬프트**: `"Write a one-line Python function that returns the sum of two numbers."`
**컨텍스트**: `num_ctx=4096`, `num_predict=64`

## 측정 방법

각 모델당:
- **Cold**: `keep_alive=0`으로 메모리 언로드 후 첫 요청 (모델 로드 시간 포함)
- **Warm × 3**: 연속 3회 측정 후 평균 (모델이 메모리에 상주하는 상태)

TTFT 측정은 streaming HTTP 요청 시 wall-clock으로 첫 응답 chunk 도달까지의 시간. Ollama 내부 timing(`load_duration`, `prompt_eval_duration`)도 함께 기록.

## 결과 요약

| 모델 | 사이즈 | Cold TTFT | Warm TTFT (평균) | 생성 속도 |
|---|---|---|---|---|
| **qwen3.6:35b** (35B-A3B MoE Q4) | 23 GB | 5,556 ms | **222 ms** | **57.3 tok/s** ⚡ |
| qwen3.6:27b (Dense Q4) | 17 GB | 6,221 ms | 325 ms | 11.6 tok/s |
| gemma4:31b (Dense Q4) | 19 GB | 11,335 ms | 6,448 ms ⚠️ | 10.5 tok/s |

## 상세 데이터

### qwen3.6:35b (35B-A3B MoE, Q4_K_M)

```
COLD   TTFT=5556ms  load=5428ms  prompt_eval=101ms  gen=57.0 tok/s
WARM-1 TTFT= 236ms  load= 151ms  prompt_eval= 80ms  gen=57.7 tok/s
WARM-2 TTFT= 224ms  load= 129ms  prompt_eval= 92ms  gen=57.0 tok/s
WARM-3 TTFT= 206ms  load= 125ms  prompt_eval= 78ms  gen=57.2 tok/s
→ avg WARM TTFT: 222ms, avg gen: 57.3 tok/s
```

✨ **종합 1위**:
- Cold start 가장 빠름 (5.5s)
- Warm TTFT 222ms — 인터랙티브 응답성 우수
- 생성 속도 57.3 tok/s — 다른 모델 대비 5배 빠름

### qwen3.6:27b (Dense, Q4_K_M)

```
COLD   TTFT=6221ms  load=6041ms  prompt_eval=149ms  gen=11.6 tok/s
WARM-1 TTFT= 314ms  load= 161ms  prompt_eval=148ms  gen=11.6 tok/s
WARM-2 TTFT= 337ms  load= 188ms  prompt_eval=145ms  gen=11.6 tok/s
WARM-3 TTFT= 325ms  load= 178ms  prompt_eval=144ms  gen=11.6 tok/s
→ avg WARM TTFT: 325ms, avg gen: 11.6 tok/s
```

✅ **agentic 코딩 정확도 1위 (SWE-bench 추정 74%)**:
- 안정적인 ~325 ms warm TTFT
- 생성 속도 11.6 tok/s — Dense라 느리지만 한 번의 정확한 응답이 더 가치 있는 워크로드(PR 작성 등)에 적합

### gemma4:31b (Dense, Q4_K_M)

```
COLD   TTFT=11335ms  load=4949ms  prompt_eval=133ms  gen=10.5 tok/s
WARM-1 TTFT= 6489ms  load= 247ms  prompt_eval=102ms  gen=10.5 tok/s
WARM-2 TTFT= 6404ms  load= 168ms  prompt_eval=100ms  gen=10.5 tok/s
WARM-3 TTFT= 6451ms  load= 207ms  prompt_eval=100ms  gen=10.5 tok/s
→ avg WARM TTFT: 6448ms, avg gen: 10.5 tok/s
```

⚠️ **Warm TTFT 이상치**:
- `load_duration`(200 ms) + `prompt_eval_duration`(100 ms) ≈ 300 ms이지만 wall-clock TTFT는 ~6.4s
- 추정 원인: **멀티모달 비전 인코더 초기화 오버헤드** — Gemma 4는 텍스트 전용 프롬프트에서도 vision tower를 초기화할 가능성
- 생성 속도 10.5 tok/s는 정상

📌 **추천**: 멀티모달이 꼭 필요한 워크로드(스크린샷 분석)가 아니면 다른 모델 선호.

## 핵심 인사이트

### 1. MoE의 압도적 디코딩 속도 — 이론 검증

DGX Spark의 273 GB/s 메모리 대역폭은 디코딩의 병목:

```
디코딩 tok/s ≈ 273 GB/s ÷ (활성 파라미터 × 바이트/파라미터)
```

- qwen3.6:35b (활성 3B × 0.5 byte ≈ 1.5 GB/token) → 이론 180 tok/s, 실측 **57 tok/s** (효율 32%)
- qwen3.6:27b (활성 27B × 0.5 byte ≈ 13.5 GB/token) → 이론 20 tok/s, 실측 **11.6 tok/s** (효율 58%)

→ MoE는 이론치 대비 효율은 낮지만 절대치는 5배 빠름. **DGX Spark에선 MoE 선택이 결정적**.

### 2. Cold TTFT는 모델 사이즈가 아닌 디스크 로드 속도

| 모델 | 사이즈 | Cold load |
|---|---|---|
| qwen3.6:35b | 23 GB | 5.4 s |
| qwen3.6:27b | 17 GB | 6.0 s |
| gemma4:31b | 19 GB | 4.9 s |

순서가 사이즈와 반비례 — 캐시 상태/ I/O 동시성 영향. 일관된 Cold 측정 어려움.

### 3. Warm TTFT는 ~200~325 ms (멀티모달 모델 제외)

코딩 어시스턴트 인터랙션에 충분한 응답성. 다만 thinking 모드가 켜지면 첫 사용자 가시 토큰까지 더 길어질 수 있음 (별도 측정 필요).

### 4. Gemma 4 멀티모달 오버헤드

텍스트 전용 프롬프트에도 ~6초 warm latency. 이미지 입력 없는 워크로드에선 단순 텍스트 모델이 선호됨.

## 권장 사용 패턴

| 워크로드 | 추천 모델 | 이유 |
|---|---|---|
| IDE 자동완성 / 인라인 채팅 | **qwen3.6:35b** | 222 ms TTFT + 57 tok/s |
| 에이전트 multi-turn (OpenClaw 등) | **qwen3.6:35b** | MoE 빠른 응답 |
| Agentic PR 작성 (Aider 등) | **qwen3.6:27b** | SWE-bench 정확도 우선 |
| UI 스크린샷 디버깅 | **gemma4:31b** | 멀티모달 (단, 응답 지연 감수) |

## 재현 방법

테스트 스크립트는 [`/tmp/ttft_test.py`](#) 에 위치. 핵심 옵션:

```python
PROMPT = "Write a one-line Python function that returns the sum of two numbers."
WARM_RUNS = 3
options = {"num_predict": 64, "num_ctx": 4096}
```

## 주의

- 1회 실측이라 표본이 작음 — 변동성 평가하려면 여러 번 반복 권장
- `num_ctx=4096`은 짧은 프롬프트용. 대형 코드베이스 입력 시 prompt_eval이 길어짐
- gemma4:31b는 256K 컨텍스트 디폴트 사용 시 KV 캐시 OOM (500 에러). 명시적으로 `num_ctx` 줄여야 함
