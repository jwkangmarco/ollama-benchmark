# Latency / Throughput 실측 결과 (DGX Spark)

**측정일**: 2026-04-26
**환경**: NVIDIA DGX Spark (GB10, 122 GB unified memory) / Ollama 0.21.2 / Linux
**프롬프트**: `"Write a one-line Python function that returns the sum of two numbers."`
**컨텍스트**: `num_ctx=4096`, `num_predict=64`

> 📝 본 측정은 처음에 "TTFT 테스트"로 시작했지만, **streaming 요청을 끝까지 받으면 TTFT 외에 throughput / TPOT도 자연스럽게 같이 캡처**되므로 종합 latency 프로파일 측정이 됨. 파일명/제목을 그에 맞게 수정.

## 측정 지표 정의

| 지표 | 정의 | 계산 |
|---|---|---|
| **TTFT** (Time To First Token) | 요청 전송 → 첫 출력 토큰 도달 wall-clock | streaming 첫 chunk timestamp |
| **TPOT** (Time Per Output Token, = ITL Inter-Token Latency) | 첫 토큰 이후 평균 토큰당 생성 시간 | `eval_duration / eval_count` |
| **Throughput** | 초당 생성 토큰 수 | `eval_count / eval_duration` (= 1/TPOT) |
| Cold/Warm | 모델이 메모리에 없는 상태(Cold) / 적재된 상태(Warm) | `keep_alive=0` 후 첫 요청 / 연속 요청 |

```
요청 전송 ─┬─ load (cold만) ─┬─ prompt_eval ─┬─ 첫 토큰 ───────── 마지막 토큰
          │                 │               │                    │
          └─────────── TTFT (wall-clock) ───┘                    │
                                            └── eval_duration ──┘
                                                  ↑
                                          throughput / TPOT 산출 구간
```

**총 응답 시간** ≈ TTFT + (N-1) × TPOT (N토큰 응답 기준)

## 측정 방법

각 모델당:
- **Cold**: `keep_alive=0`으로 메모리 언로드 후 첫 요청 (모델 로드 시간 포함)
- **Warm × 3**: 연속 3회 측정 후 평균

TTFT는 streaming HTTP 요청 시 wall-clock으로 첫 응답 chunk 도달까지의 시간. Throughput / TPOT은 Ollama 응답의 `eval_count`, `eval_duration` 필드에서 산출.

## 결과 요약

| 모델 | 사이즈 | Cold TTFT | **Warm TTFT** | **TPOT** | **Throughput** |
|---|---|---|---|---|---|
| **qwen3.6:35b** (35B-A3B MoE Q4) | 23 GB | 5,556 ms | **222 ms** | **17.5 ms/tok** | **57.3 tok/s** ⚡ |
| qwen3.6:27b (Dense Q4) | 17 GB | 6,221 ms | 325 ms | 86.2 ms/tok | 11.6 tok/s |
| gemma4:31b (Dense Q4) | 19 GB | 11,335 ms | 6,448 ms ⚠️ | 95.2 ms/tok | 10.5 tok/s |

### 64-토큰 응답 완료까지 wall-clock 추정

```
총 시간 ≈ TTFT + (64 - 1) × TPOT
```

| 모델 | TTFT | TPOT × 63 | **합계** |
|---|---|---|---|
| qwen3.6:35b | 222 ms | 1,103 ms | **1.32 s** |
| qwen3.6:27b | 325 ms | 5,431 ms | **5.76 s** |
| gemma4:31b | 6,448 ms | 5,998 ms | **12.45 s** |

→ qwen3.6:35b가 64-토큰 응답 완성에 27B 대비 **4.4배 빠름**, gemma4 대비 **9.4배 빠름**.

## 상세 데이터

### qwen3.6:35b (35B-A3B MoE, Q4_K_M)

```
COLD   TTFT=5556ms  load=5428ms  prompt_eval=101ms  TPOT=17.5ms  throughput=57.0 tok/s
WARM-1 TTFT= 236ms  load= 151ms  prompt_eval= 80ms  TPOT=17.3ms  throughput=57.7 tok/s
WARM-2 TTFT= 224ms  load= 129ms  prompt_eval= 92ms  TPOT=17.5ms  throughput=57.0 tok/s
WARM-3 TTFT= 206ms  load= 125ms  prompt_eval= 78ms  TPOT=17.5ms  throughput=57.2 tok/s
→ avg WARM: TTFT=222ms, TPOT=17.4ms, throughput=57.3 tok/s
```

✨ **종합 1위** — Warm TTFT 222ms로 인터랙티브 응답성 확보 + TPOT 17.5ms로 빠른 토큰 흐름.

### qwen3.6:27b (Dense, Q4_K_M)

```
COLD   TTFT=6221ms  load=6041ms  prompt_eval=149ms  TPOT=86.2ms  throughput=11.6 tok/s
WARM-1 TTFT= 314ms  load= 161ms  prompt_eval=148ms  TPOT=86.2ms  throughput=11.6 tok/s
WARM-2 TTFT= 337ms  load= 188ms  prompt_eval=145ms  TPOT=86.2ms  throughput=11.6 tok/s
WARM-3 TTFT= 325ms  load= 178ms  prompt_eval=144ms  TPOT=86.2ms  throughput=11.6 tok/s
→ avg WARM: TTFT=325ms, TPOT=86.2ms, throughput=11.6 tok/s
```

✅ **agentic 코딩 정확도 1위 (SWE-bench 추정 74%)** — 안정적이지만 TPOT 86ms로 느림. 한 번의 정확한 응답이 가치 있는 워크로드(PR 작성)에 적합.

### gemma4:31b (Dense, Q4_K_M)

```
COLD   TTFT=11335ms  load=4949ms  prompt_eval=133ms  TPOT=95.2ms  throughput=10.5 tok/s
WARM-1 TTFT= 6489ms  load= 247ms  prompt_eval=102ms  TPOT=95.2ms  throughput=10.5 tok/s
WARM-2 TTFT= 6404ms  load= 168ms  prompt_eval=100ms  TPOT=95.2ms  throughput=10.5 tok/s
WARM-3 TTFT= 6451ms  load= 207ms  prompt_eval=100ms  TPOT=95.2ms  throughput=10.5 tok/s
→ avg WARM: TTFT=6448ms, TPOT=95.2ms, throughput=10.5 tok/s
```

⚠️ **Warm TTFT 이상치**:
- `load_duration`(200 ms) + `prompt_eval_duration`(100 ms) ≈ 300 ms이지만 wall-clock TTFT는 ~6.4s
- 추정 원인: **멀티모달 비전 인코더 초기화 오버헤드** — 텍스트 전용 프롬프트에서도 vision tower 매번 초기화 가능성
- TPOT/Throughput은 정상 (다른 Dense 모델과 비슷)

## 핵심 인사이트

### 1. MoE의 압도적 디코딩 속도 — 이론 검증

DGX Spark의 273 GB/s 메모리 대역폭이 디코딩의 병목:

```
이론 throughput ≈ 273 GB/s ÷ (활성 파라미터 × 바이트/파라미터)
이론 TPOT ≈ 1 / 이론 throughput
```

| 모델 | 활성 파라미터 메모리 | 이론 throughput | 실측 throughput | 효율 |
|---|---|---|---|---|
| qwen3.6:35b | 활성 3B × 0.5B = 1.5 GB/token | 180 tok/s | 57.3 tok/s | 32% |
| qwen3.6:27b | 활성 27B × 0.5B = 13.5 GB/token | 20 tok/s | 11.6 tok/s | 58% |

→ MoE는 이론 효율은 낮지만 절대치 5배 빠름. **DGX Spark에선 MoE 선택이 결정적**.

### 2. TPOT 분석으로 본 사용자 경험

| TPOT 범위 | 사용자 인지 | 본 측정에서 |
|---|---|---|
| < 30 ms/tok | "마치 GPT-4o 같다" — 자연스러운 흐름 | qwen3.6:35b (17.5) ✅ |
| 30~60 ms/tok | 약간 느린 streaming, 가독성 OK | — |
| 60~100 ms/tok | 단어 단위로 끊겨 보임, 답답함 | qwen3.6:27b (86), gemma4 (95) |
| > 150 ms/tok | 한 글자씩 떠오름, 사용 어려움 | — |

→ **에이전트/IDE 통합용으로는 35B-A3B가 유일하게 자연스러운 UX**. Dense 27B / Gemma 4는 스트리밍 체감이 답답.

### 3. Cold TTFT는 모델 사이즈와 비례하지 않음

| 모델 | 사이즈 | Cold load | 비고 |
|---|---|---|---|
| gemma4:31b | 19 GB | 4.9 s | 가장 빠른 로드 |
| qwen3.6:35b | 23 GB | 5.4 s | |
| qwen3.6:27b | 17 GB | 6.0 s | 가장 느린 로드 |

순서가 사이즈와 반비례 — 디스크 캐시 상태 / I/O 동시성 영향. 1회 측정으론 신뢰성 낮음.

### 4. Warm TTFT ≈ Load + Prompt Eval (정상 모델)

| 모델 | warm load | warm prompt_eval | 합계 추정 | 실측 TTFT |
|---|---|---|---|---|
| qwen3.6:35b | 135 ms | 83 ms | 218 ms | **222 ms** ✅ |
| qwen3.6:27b | 176 ms | 146 ms | 322 ms | **325 ms** ✅ |
| gemma4:31b | 207 ms | 101 ms | 308 ms | **6,448 ms** ❌ |

→ Gemma 4는 Ollama가 보고하는 timing 외 ~6초가 어디론가 사라짐. 비전 모델 초기화 가설.

## 권장 사용 패턴

| 워크로드 | 추천 모델 | 핵심 지표 |
|---|---|---|
| IDE 자동완성 / 인라인 채팅 | **qwen3.6:35b** | TTFT 222ms + TPOT 17.5ms |
| 에이전트 multi-turn (OpenClaw) | **qwen3.6:35b** | 빠른 turn 회전율 |
| Agentic PR 작성 (Aider 등) | **qwen3.6:27b** | SWE-bench 정확도 우선, TPOT는 양보 |
| UI 스크린샷 디버깅 | **gemma4:31b** | 멀티모달 (단, ~6s 응답 지연 감수) |

## 추가 측정으로 확장하려면

본 측정은 단일 프롬프트 / 짧은 출력 / 1회 측정이라 표본 부족. 더 신뢰성 있는 결과를 원하면:

| 측정 항목 | 방법 | 가치 |
|---|---|---|
| **TPOT 분포 (jitter)** | streaming chunk마다 timestamp 기록 | 사용자 체감 평가 |
| **Long context TPOT** | `num_ctx=32K, 128K`로 KV 캐시 영향 측정 | 대형 코드베이스 워크로드 |
| **Concurrent throughput** | 동시 N개 요청 → 총 tok/s | 멀티 사용자 / 배치 |
| **Prompt-eval throughput** | 긴 프롬프트로 prefill 속도 측정 | 컴퓨트 vs 대역폭 분석 |
| **Real-world prompts** | SWE-bench / HumanEval 실제 프롬프트 | 정확도 + 지연 동시 평가 |

→ 본 레포(`ollama-benchmark`)의 벤치마크 도구로 후속 측정 권장.

## 재현 방법

테스트 스크립트: `/tmp/ttft_test.py` (외부 의존성 없음, urllib만)

```python
# 핵심 옵션
PROMPT = "Write a one-line Python function that returns the sum of two numbers."
WARM_RUNS = 3
options = {"num_predict": 64, "num_ctx": 4096}
```

## 주의

- 1회 측정 — 표본 작음. 변동성 평가하려면 여러 번 반복 권장
- `num_ctx=4096`은 짧은 프롬프트용. 대형 코드베이스 입력 시 prompt_eval 길어짐 → TTFT 증가, TPOT는 거의 동일
- gemma4:31b는 256K 기본 컨텍스트 시 KV 캐시 OOM (500 에러). 명시적으로 `num_ctx` 줄여야 함
- Ollama 0.21.2 / GB10 환경 — 다른 환경에서 결과 다를 수 있음
