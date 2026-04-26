# 설치된 셋업 (DGX Spark)

본 연구를 통해 실제로 설치/구성된 상태 기록.

## Ollama

| 항목 | 값 |
|---|---|
| 버전 | 0.21.2 |
| API | `http://127.0.0.1:11434/v1` (OpenAI 호환) |
| Native API | `http://127.0.0.1:11434/api/*` |

### 설치된 모델

```
NAME           ID              SIZE     MODIFIED
gemma4:31b     6316f0629137    19 GB    [installed]
qwen3.6:35b    07d35212591f    23 GB    [installed]
qwen3.6:27b    a50eda8ed977    17 GB    [installed]
```

총 **59 GB** (122 GB unified memory의 ~48%).

### 다운로드 명령 (재현용)

```bash
ollama pull qwen3.6:27b   # Dense Q4, agentic 정확도 1위
ollama pull qwen3.6:35b   # MoE A3B Q4, 빠른 추론
ollama pull gemma4:31b    # Dense Q4, 멀티모달
```

### 검증

```bash
ollama list
ollama ps                                # 현재 메모리 적재 상태
curl http://127.0.0.1:11434/api/tags     # API 동작 확인
```

## OpenClaw 통합

설정 파일: `~/.openclaw/openclaw.json`

### Provider 등록 (변경된 부분만)

```json
"ollama": {
  "baseUrl": "http://127.0.0.1:11434/v1",
  "apiKey": "ollama-local",
  "api": "ollama",
  "models": [
    {
      "id": "qwen3.6:27b",
      "name": "Qwen 3.6 27B (Dense, Agentic Coding)",
      "reasoning": true,
      "input": ["text"],
      "contextWindow": 262144,
      "maxTokens": 8192
    },
    {
      "id": "qwen3.6:35b",
      "name": "Qwen 3.6 35B-A3B (MoE, Fast Coding)",
      "reasoning": true,
      "input": ["text"],
      "contextWindow": 262144,
      "maxTokens": 8192
    },
    {
      "id": "gemma4:31b",
      "name": "Gemma 4 31B (Multimodal Coding)",
      "reasoning": false,
      "input": ["text", "image"],
      "contextWindow": 262144,
      "maxTokens": 8192
    }
  ]
}
```

### Agents Default

```json
"agents": {
  "defaults": {
    "model": {
      "primary": "ollama/qwen3.6:35b"
    },
    "models": {
      "moonshot/kimi-k2.5": {},
      "google/gemini-3-flash-preview": {},
      "ollama/qwen3.6:27b": {},
      "ollama/qwen3.6:35b": {},
      "ollama/gemma4:31b": {}
    },
    ...
  }
}
```

**Primary**: `ollama/qwen3.6:35b` 선택 이유 — TTFT 222ms, 생성 57 tok/s로 에이전트 multi-turn 워크로드에 가장 적합.

### Gateway

- 포트: 18789
- 모드: local / loopback
- 인증: token-based

```bash
curl http://127.0.0.1:18789/  # Control UI HTML 응답 확인
```

### 백업

본 연구 중 수정 전 스냅샷:
- `~/.openclaw/openclaw.json.bak.pre-2026-04-26`
- 기존 자동 백업: `*.bak`, `*.bak.1` ~ `*.bak.13`

## 실측 메모리 사용량 (`ollama ps`)

기본 컨텍스트 (262144) 사용 시:

```
NAME           SIZE     PROCESSOR    CONTEXT    UNTIL
qwen3.6:35b    34 GB    100% GPU     262144     Forever
qwen3.6:27b    42 GB    100% GPU     262144     Forever
```

⚠️ **두 모델 동시 적재 시 76 GB 점유**. gemma4:31b 추가 적재 시 OOM 위험.

### 권장 운용

- **단일 모델 상주**: 메모리 편안 (<50 GB), KV 캐시 여유
- **멀티 모델 동시**: `num_ctx`를 작업별로 명시 (예: 4K~32K)
- **컨텍스트 축소 명령** (Ollama):
  ```bash
  curl http://localhost:11434/api/generate -d '{
    "model":"qwen3.6:35b","prompt":"hi","options":{"num_ctx":4096}
  }'
  ```

## 참고: 다른 GitHub 레포 작업

본 셋업 검증 과정에서 SSH 기반 git 푸시 사용 (HTTPS 401 회피):

```bash
git remote set-url origin git@github.com:jwkangmarco/ollama-benchmark.git
```

`gh auth status`가 SSH 프로토콜로 설정된 환경에선 SSH가 안정적.

## 다음 단계

본 연구 결과를 토대로 다음 활동 가능:

1. **`ollama-benchmark` 레포의 실제 벤치마크 코드로 SWE-bench / HumanEval 측정**
2. **OpenClaw 에이전트로 코딩 태스크 실행 → 시간 + 품질 측정**
3. **vLLM 병행 셋업 → NVFP4 가속 후 처리량 비교**
4. **클러스터링 (DGX Spark 2대) → Qwen3-Coder-480B 등 대형 모델 시도**

## 출처 및 관련 파일

- 벤치마크 데이터: [`ttft-results.md`](ttft-results.md)
- 모델 추천 근거: [`recommendations.md`](recommendations.md)
- 양자화 호환성: [`quantization-platforms.md`](quantization-platforms.md)
