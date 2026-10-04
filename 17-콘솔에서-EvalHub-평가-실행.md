# 17. 콘솔에서 EvalHub 평가 실행하기 (데모용)

데모에서 **콘솔만으로** EvalHub 설정 내역을 보여주고, 평가를 실행하고, 결과를 보여주는 순서입니다.
2026-10-05에 OpenShift AI 3.5.1 콘솔에서 직접 따라가며 확인한 내용입니다.

## EvalHub 설명 대본

콘솔을 열기 전에 EvalHub가 무엇인지 1분 30초 정도로 설명할 때 쓰는 대본입니다.

### 먼저 정리: 도구, 창구, 절차

| | 무엇인가 | 비유 |
|---|---|---|
| GuideLLM | 실제로 부하를 걸고 속도를 재는 **도구** | 측정 장비 |
| EvalHub | 평가 도구들을 등록해 두고, 요청을 받아 실행하고, 결과를 기록하는 **서비스** | 검사 접수 창구 |
| 검증 파이프라인 | 모델을 올리고, 측정하고, 판정하고, 되돌리는 **절차 전체** | 검사 공정 |

콘솔의 Evaluations 메뉴가 EvalHub의 화면입니다.

```
콘솔 · 파이프라인 · API
        │ 평가 요청
        ▼
     EvalHub ──평가 작업 실행──▶ GuideLLM ──부하──▶ 모델
        │
        └─ 결과 기록 ──▶ MLflow (Experiments)
```

### 대본

**1. 왜 필요한가 (약 25초)**

> 모델을 검증하려면 도구가 여러 개 필요합니다. 속도는 GuideLLM으로 재고, 정확도는 LM Evaluation Harness로 봅니다. 도구마다 실행 방법도 다르고 결과 형식도 다릅니다. 팀마다 따로 돌리면 누가 어떤 조건으로 쟀는지 남지 않습니다.
>
> EvalHub는 이 도구들을 한곳에 모아 둔 평가 서비스입니다. 같은 방식으로 요청하고, 같은 곳에 기록이 남습니다.

**2. 어떻게 구성하나 (약 25초)**

> 구성은 세 가지를 정하면 됩니다. 어떤 평가 도구를 쓸지, 어떤 평가 묶음을 쓸지, 결과를 어디에 기록할지입니다.
>
> 지금 이 프로젝트에는 성능 측정용 GuideLLM과 정확도 평가용 LM Evaluation Harness가 등록돼 있고, 결과는 MLflow에 남도록 해 두었습니다. 이 화면에 보이는 평가 195개가 그렇게 등록된 항목입니다.

**3. 어떻게 실행되나 (약 25초)**

> 실행은 평가를 고르고, 모델 주소를 넣고, 시작을 누르면 됩니다.
>
> 그러면 EvalHub가 평가 작업을 하나 띄웁니다. 그 작업이 모델에 요청을 보내면서 측정하고, 끝나면 결과를 EvalHub에 돌려주고 사라집니다. 콘솔에서 누르든, 검증 파이프라인이 자동으로 부르든 같은 방식으로 실행됩니다.

**4. 어떤 결과가 나오나 (약 15초)**

> 결과는 두 곳에서 봅니다. Evaluations 화면에서는 통과 여부를, Experiments 화면에서는 초당 토큰 수와 첫 토큰 시간 같은 세부 수치를 봅니다. 이전 실행과 나란히 비교할 수도 있습니다.

### 대본을 뒷받침하는 사실

질문이 나왔을 때 답할 수 있도록 실제 구성을 적어 둡니다.

| 대본의 말 | 실제 구성 |
|---|---|
| 평가 도구를 정한다 | EvalHub 리소스의 `providers`: `guidellm`, `lm-evaluation-harness` |
| 평가 묶음을 정한다 | EvalHub 리소스의 `collections`: `standard-llm-evals-v1` |
| 결과 기록 위치를 정한다 | EvalHub 리소스의 환경 변수 `MLFLOW_TRACKING_URI`. 직접 넣어야 합니다 |
| 평가 작업을 띄운다 | 프로젝트 안에 평가 파드가 하나 생깁니다. 평가 도구 컨테이너와, 모델·EvalHub·MLflow로 요청을 중계하는 사이드카로 이루어집니다 |
| 모델에 요청을 보낸다 | 클러스터 내부 모델 주소로 직접 보냅니다. 외부 게이트웨이를 거치지 않습니다 |
| 통과 여부를 표시한다 | 평가마다 기본 지표와 기준이 정의돼 있습니다. GuideLLM은 `output_tokens_per_second`, 기본 기준 10 |

구성할 때 추가로 필요했던 것은 두 가지입니다.

- **권한**: 파이프라인이 EvalHub를 부르려면 실행 계정에 평가 권한을 묶어야 합니다. 없으면 403이 납니다.
- **모델 인증서**: 모델이 TLS로만 서빙하면 CA 인증서를 시크릿으로 만들어 평가 요청에 함께 넘겨야 합니다(`evalhub-model-service-ca`).

우리 기준(30 tok/s, 첫 토큰 200 ms)으로 판정하는 것은 EvalHub가 아니라 검증 파이프라인의 gate 단계입니다. EvalHub는 측정과 기록을 맡습니다. 파이프라인 단계는 [18-검증-파이프라인-단계-설명.md](18-검증-파이프라인-단계-설명.md)를 참고합니다.

## 결론: `rhsummit` 프로젝트의 EvalHub를 씁니다

클러스터에 EvalHub가 두 개 있습니다. 콘솔의 Evaluations 화면은 **선택한 프로젝트**에 따라 다른 EvalHub를 보여줍니다.

| | `rhsummit` 프로젝트 | `demo` 프로젝트 |
|---|---|---|
| 연결되는 EvalHub | `rhsummit` 네임스페이스의 EvalHub | `redhat-ods-applications`의 중앙 EvalHub (한방 설치 스크립트가 만든 것) |
| 실행 이력 | 9월 26일부터의 성능 측정 이력이 있음 | 없음 |
| 콘솔에서 실행 | **됨** (2026-10-05 확인, 2분 만에 완료) | **안 됨**. 작업이 Pending에서 멈춤 |
| 평가 도구 | GuideLLM, LM Evaluation Harness | 왼쪽 둘 + garak, lighteval |
| 평가 묶음 | Standard LLM Evaluation Suite v1 | Open LLM Leaderboard v2, Safety & Fairness |

`demo` 쪽이 멈추는 이유는 네트워크 정책입니다. 평가 작업은 `demo` 네임스페이스에서 돌고 결과를 중앙 EvalHub로 보내는데, `redhat-ods-applications`는 정해진 라벨이 붙은 네임스페이스의 연결만 받습니다. 그래서 결과 보고가 시간 초과로 끊깁니다.

데모는 **`rhsummit`** 으로 진행합니다. 이력이 있어 화면이 비어 보이지 않고, 검증 파이프라인도 같은 EvalHub를 씁니다.

## 데모 흐름 (약 3분)

### 1. 설정 내역 보여주기

OpenShift AI 콘솔에는 EvalHub 전용 설정 메뉴가 없습니다(Settings 아래에 평가 항목이 없습니다). 대신 **실행 마법사의 목록 화면**이 곧 설정 내역입니다.

1. 왼쪽 메뉴 **Develop & train → Evaluations**
2. 위쪽 **Project**를 `rhsummit`으로 선택
3. **Start evaluation run** 클릭
4. 두 카드가 나옵니다.
   - **Benchmark**: 단일 평가. 누르면 등록된 평가 195개가 카드로 나옵니다.
   - **Benchmark suite**: 평가 묶음. 누르면 "Standard LLM Evaluation Suite v1 (12 benchmarks)"가 나옵니다.
5. **Benchmark**를 누르고, 필터를 **Name**으로 바꿔 `steady`를 입력합니다.
   - "Steady-state load test" 카드 하나가 남습니다. 분류 표시는 **Performance**, 지표는 `requests_per_second`, `output_tokens_per_second`, `mean_ttft_ms` 등입니다.
   - 이것이 **GuideLLM**의 일정 부하 측정입니다.

말할 것: "플랫폼에 평가 도구와 평가 항목이 미리 등록돼 있습니다. 성능 측정은 GuideLLM, 정확도 평가는 LM Evaluation Harness입니다."

원본 설정을 보여주고 싶으면 OpenShift 콘솔에서 봅니다(메뉴 경로만 확인했고 화면은 열어보지 않았습니다).

- **Home → Search** → Resources에서 `EvalHub` 선택 → 프로젝트 `rhsummit` → `evalhub` → YAML 탭
  - `providers`(평가 도구), `collections`(평가 묶음), `MLFLOW_TRACKING_URI`(결과 기록 위치)가 보입니다.
- **Workloads → ConfigMaps** → `evalhub-provider-guidellm`
  - 평가별 기본 지표와 통과 기준이 들어 있습니다.

### 2. 실행하기

"Steady-state load test" 카드의 **Select benchmark**를 누르면 입력 화면이 나옵니다.

| 항목 | 넣을 값 | 설명 |
|---|---|---|
| Evaluation name | 예: `qwen3-32b-awq-console-demo` | 기본값은 날짜·시각 |
| MLflow Experiment | **Select existing experiment** → `rhsummit-model-validation` | 기본 선택이 다른 실험일 수 있으니 확인 |
| Source | `Model` | 기본값 |
| Model | **Other (External endpoint)** | 아래 주의 참고 |
| Model name | `qwen3-32b-awq` | 대소문자 구분 |
| Endpoint URL | `https://qwen3-32b-awq-kserve-workload-svc.rhsummit.svc.cluster.local:8000/v1` | 클러스터 내부 주소 |
| API key secret name | `evalhub-model-service-ca` | 이름은 API 키지만 **CA 인증서 시크릿**을 넣습니다 |
| Benchmark threshold | **건드리지 않음** | 아래 주의 참고 |
| Primary scorer metric | `output_tokens_per_second` | 기본값 |
| Benchmark parameters | 체크 후 `{"rate": 2, "max_seconds": 60}` | JSON 입력 |

**Start evaluation run**을 누르면 목록으로 돌아가고 "Evaluation started" 알림이 뜹니다. 상태는 Pending → Running → Complete로 바뀌며 약 2분 걸립니다.

기다리는 동안 보여줄 것: 목록에 쌓여 있는 이전 실행 이력, 또는 Observe & monitor의 GPU·llm-d 대시보드.

### 3. 결과 보여주기

**Evaluations 화면**

- 목록에서 완료된 실행의 이름을 누릅니다.
- 보이는 것: Evaluation score, **Pass** 표시, Primary metric, Benchmark threshold.
- 콘솔 결과 화면에는 이 네 가지만 나옵니다. 첫 토큰 시간 같은 세부 지표는 없습니다.

**Experiments 화면 (세부 지표)**

- 왼쪽 메뉴 **Develop & train → Experiments** → Project `RHSummit`
- 실험 이름(`rhsummit-model-validation`)을 눌러 방금 실행을 엽니다.
- 보이는 것: `output_tokens_per_second`, `mean_ttft_ms`, `mean_itl_ms`, `requests_per_second`.
- 화면 설명은 [08-mlflow-화면-설명.md](08-mlflow-화면-설명.md)를 참고합니다.

2026-10-05 콘솔 실행의 결과는 36.93 tok/s, 첫 토큰 평균 104.5 ms, Pass였습니다.

## 콘솔에서 걸리는 것 다섯 가지

실제로 따라가며 확인한 것들입니다. 데모 전에 알고 있어야 당황하지 않습니다.

1. **Model 목록에 운영 모델이 안 나옵니다.**
   `qwen3-32b-awq`는 LLMInferenceService인데, 목록에는 InferenceService만 나옵니다. 그래서 `qwen36-35b-a3b-community`(사용 불가 표시)와 "Other (External endpoint)"만 보입니다. **Other를 고르고 주소를 직접 넣습니다.**

2. **Validate connection은 실패로 나옵니다.**
   "Connection verification failed."가 뜨지만 실행은 정상입니다. 데모에서는 이 버튼을 누르지 않습니다.

3. **점수가 `3693%`처럼 표시됩니다.**
   실제 값은 36.93 tok/s입니다. 콘솔이 모든 점수를 0~1 비율로 보고 100을 곱해 퍼센트로 표시하기 때문입니다. "초당 토큰 36.93을 퍼센트로 잘못 표시하는 기술 미리보기 단계의 화면"이라고 말하고, 정확한 값은 Experiments에서 보여줍니다.

4. **통과 기준(threshold)을 콘솔에서는 30으로 넣을 수 없습니다.**
   입력 칸의 최대값이 100이고, 100을 넣으면 실제로는 1 tok/s로 저장됩니다(같은 퍼센트 문제). 기본값은 1000으로 표시되는데 이는 10 tok/s에 해당합니다. 값을 고치면 100으로 잘리므로 **건드리지 않습니다**. 건드리지 않았을 때 10이 그대로 들어가는지는 확인하지 못했습니다.
   우리 기준(30 tok/s, 첫 토큰 200 ms)으로 판정하는 것은 **검증 파이프라인의 gate 단계**입니다. 콘솔 실행은 "측정"을, 파이프라인은 "우리 기준 판정"을 보여주는 것으로 역할을 나눕니다.

5. **Evaluations는 Tech Preview 표시가 붙어 있습니다.**
   메뉴 옆에 그대로 보이므로 먼저 말해 두는 편이 낫습니다.

## 데모 전 준비

- [ ] 모델 `qwen3-32b-awq`가 Ready인지 확인
- [ ] `rhsummit`에 시크릿 `evalhub-model-service-ca`가 있는지 확인 (없으면 `scripts/19-evalhub-model-ca.sh`)
- [ ] Endpoint URL과 파라미터 JSON을 메모장에 준비 (화면에서 타이핑하면 길고 틀리기 쉽습니다)
- [ ] 콘솔에서 한 번 미리 실행해 Complete까지 가는지 확인
- [ ] 목록에 Failed로 남은 실행이 눈에 띄면 지울지 결정
  - `qwen3-32b-awq-safety-fairness-sample`(10월 5일)은 안전성 평가 시험 실행이며 6종 중 1종이 실패해 Failed로 표시됩니다.

## 안전성 평가 묶음은 데모에 쓰지 않습니다

2026-10-05에 Safety & Fairness 6종을 항목당 50문항으로 돌려 봤습니다. 실행은 되지만 보여줄 수준이 아닙니다.

- 통과 판정이 계산되지 않습니다. 평가 묶음에 적힌 지표 이름과 도구가 내는 이름이 달라 비교가 안 됩니다.
- 6종 중 BBQ는 결과 저장 단계에서 실패하고, Winogender는 지표가 기록되지 않습니다.
- 나온 수치는 표본이 작고 평가 방식이 모델과 맞지 않아 모델의 안전성 점수로 읽을 수 없습니다.

서비스 검증은 장표대로 "기준은 서비스가 정하고 도구는 플랫폼이 제공한다"는 설명으로 진행합니다.

## 관련 문서

- [08-mlflow-화면-설명.md](08-mlflow-화면-설명.md) — Experiments 화면 읽는 법
- [10-데모-순서.md](10-데모-순서.md) — 전체 데모 순서
- [12-검증-범위.md](12-검증-범위.md) — 플랫폼 검증과 서비스 검증의 범위
