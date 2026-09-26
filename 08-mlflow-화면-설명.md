# MLflow 화면 — 데모에서 무엇을 보여줄 것인가

MLflow는 메뉴가 많지만 이 데모에서 쓰는 화면은 **experiment 하나, 그 안의 세 뷰**입니다.
이 문서는 실제 인스턴스를 조회해 **무엇이 채워져 있고 무엇이 비어 있는지** 확인한 결과입니다.

---

## 먼저 알아야 할 것 — 처음엔 비어 있었고, 그래서 채웠습니다

**EvalHub는 MLflow에 run을 만들고 `evaluation-card.json` 아티팩트만 올립니다.**
metric도 param도 기록하지 않고 run을 `RUNNING`으로 남깁니다. 그 상태로 Overview를
열면 아무것도 없습니다 — 실제로 그렇게 보였습니다.

그래서 **카드의 숫자를 metric으로 옮겨 적는 단계**를 넣었습니다.

| 어디서 | 무엇을 |
|---|---|
| 파이프라인 `measure` 단계 | 평가가 끝나면 metric 6개·param·tag를 기록하고 run을 `FINISHED`로 닫음 |
| `scripts/16-mlflow-record.sh` | 이미 끝난 EvalHub 작업을 소급 기록 (데모 전 한 번 실행) |

기록 후 실제로 보이는 것:

| run | 상태 | tok/s | TTFT | gate |
|---|---|---|---|---|
| `qwen3-32b-awq · run1` | FINISHED | 36.87 | 105.0 ms | 1.0 (통과) |
| `qwen3-32b-awq · run2` | FINISHED | 36.87 | 105.0 ms | 1.0 |
| `qwen3-32b-awq · run3 rate8` | FINISHED | 36.89 | 105.5 ms | 1.0 |
| `qwen3-32b-awq · qwen3-32b-awq-demo-1530` | FINISHED | 36.90 | 105.1 ms | 1.0 |
| `qwen36-35b-a3b-fp8 · pipeline` | FINISHED | 9.87 | 311.3 ms | **0.0 (기각)** |
| `qwen36-35b-a3b-fp8 · pipeline · FAILED` | FAILED | — | — | — |

> ⚠️ **데모 전에 `16-mlflow-record.sh`를 한 번 돌리세요.** 그 사이 EvalHub로 새 평가를
> 돌렸다면 그 run은 다시 빈 상태입니다. 스크립트는 metric이 없는 완료 run만 채우므로
> 여러 번 실행해도 안전합니다.

---

## ⚠️ 먼저 — experiment 상단의 `GenAI | Model training` 토글

experiment를 열면 이름 옆에 **`GenAI` / `Model training`** 토글이 있습니다. 기본이
`GenAI`이고, 그 화면은 **Traces · Sessions · Judges · Evaluation runs · Prompts** —
에이전트 관측용입니다. 우리는 trace를 만들지 않으므로 Overview의 Traces/Latency/Errors가
전부 "No data available"입니다. **비어 있는 게 정상입니다. 데이터가 없는 게 아닙니다.**

**`Model training`을 누르세요.** run 목록·metric·차트가 그쪽에 있습니다.

> 실제로 이 토글 때문에 "Overview에 아무것도 없다"고 오해했습니다. 데모 전에 토글을
> `Model training`으로 두고 시작하세요.

메뉴 위치(스크린샷 기준): 왼쪽 메뉴 **Develop & train → Experiments**. MLflow는 별도
앱이 아니라 여기 임베드되어 있고, `Go to RHSummit` 링크로 워크스페이스를 오갑니다.

## 보여줄 것 — 세 뷰, 약 2분

### ① Develop & train → Experiments → `rhsummit-model-validation` → **Model training** (15초)

experiment가 세 개 보입니다. **`rhsummit-model-validation`만 여세요.** `AIP-default`와
`MLflow Demo`는 기본 생성물·샘플입니다.

*멘트:* 평가를 돌릴 때마다 결과가 여기 한 곳에 모입니다.

### ② 실행 목록 — 표로 비교 (45초) ★ 핵심

run 목록에서 metric 열을 켭니다: `output_tokens_per_second`, `mean_ttft_ms`, `gate_pass`.
**같은 표에 32B 네 줄과 35B 한 줄이 나란히** 보입니다.

*멘트:* 32B 네 번이 36.87 · 36.87 · 36.89 · 36.90 — 측정이 재현됩니다. 그 옆 35B는 9.87.
`gate_pass`가 1과 0으로 갈립니다. **파이프라인이 기각한 근거가 이 열입니다.**

**확대할 곳**: `output_tokens_per_second` 열과 `gate_pass` 열.

### ③ 차트 뷰 — 막대로 비교 (30초)

실행 목록 상단에서 차트 뷰로 전환하고 `output_tokens_per_second`를 고릅니다.
32B 막대 넷이 같은 높이, 35B 막대 하나가 1/4 높이로 섭니다.

*멘트:* 숫자보다 이 그림이 빠릅니다. 3.7배 차이가 한눈에 보입니다.

### ④ run 하나 → Overview (30초, 선택)

`qwen36-35b-a3b-fp8 · pipeline`을 엽니다. Overview에 **Parameters**와 **Metrics**가 표로
있습니다. `vllm_args`(OOM 회피 인자)와 `gpu` 파라미터를 짚으세요 — **조건이 기록되어
있어야 나중에 비교가 의미 있습니다.**

Artifacts 탭의 `evaluation-card.json`은 원본 증거입니다. 필요하면 열되, 숫자는 이미
Overview에 있으니 시간이 없으면 건너뛰세요.

---

## 열지 말아야 할 메뉴

| 메뉴 | 왜 |
|---|---|
| **Models** | 비어 있음. **모델 버전 관리는 OpenShift AI 모델 레지스트리에서** 하고 MLflow 쪽은 안 씁니다 |
| Prompts | 비어 있음 |
| Traces / Evaluations | 비어 있음 |
| System metrics 탭 | 기록 안 함 |

**Models 메뉴는 특히 헷갈립니다.** 앞에서 모델 레지스트리를 보여준 뒤라 같은 이름이
또 나옵니다. 질문이 나오면: *"모델 버전 관리는 OpenShift AI의 모델 레지스트리에서 하고,
MLflow는 측정 기록을 담당합니다. 둘 다 쓸 필요는 없어서 한쪽만 씁니다."*

---

## 주의 — 무대에서 걸릴 수 있는 것

**run 이름.** 소급 기록한 run은 `모델 · run1` 식으로, 파이프라인이 만든 run은
`모델 · pipeline · <작업ID 앞 8자>`로 이름이 붙습니다. 이름이 겹치지 않는지 미리 보세요.

**`FAILED` run 하나.** 캐시 문제로 모델 없이 측정을 시도했던 실행입니다. 지우지 마세요 —
"실패도 기록에 남는다"는 좋은 한 마디가 됩니다. 질문이 나오면 그렇게 답하세요.

**한 문장으로:**
> **"평가를 돌릴 때마다 무엇을 어떤 조건으로 재서 얼마가 나왔는지가 여기 한 곳에 쌓이고,
> 버전을 바꾸면 같은 표에서 비교합니다."**

---

## 데모 전 체크리스트

- [ ] `scripts/16-mlflow-record.sh` 실행 — 빈 run이 없는지
- [ ] 실행 목록에서 metric 열 3개 켜 두기 (브라우저가 기억합니다)
- [ ] 차트 뷰에서 `output_tokens_per_second` 선택해 두기
- [ ] Overview → Parameters에 `vllm_args`, `gpu`가 보이는지
- [ ] Models 메뉴 질문 답변 준비
