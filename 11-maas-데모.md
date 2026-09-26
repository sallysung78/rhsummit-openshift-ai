# MaaS (Models as a Service) — 별도 데모 정리

카탈로그·레지스트리가 "어떤 모델을 들여와 쓸 것인가"라면, MaaS는 **"그 모델을 조직 안
여러 팀에 어떻게 서비스로 내줄 것인가"** 입니다. 지금 데모에는 없던 축이라 따로 정리합니다.

## 현재 상태 — 켰습니다 (2026-09-26)

`MaasTenantConfig default-tenant` **Ready=True**, `AITenant models-as-a-service` **True**,
`maas-api` Running. 아래는 켜기 전 상태이고, 켜는 데 필요했던 것은 그 다음 절입니다.

### 켜기 전 상태 (기록)

| | |
|---|---|
| DSC 컴포넌트 | `aigateway` = **Removed** |
| 하위 스위치 | `aigateway.modelsAsAService.managementState`, `aigateway.batchGateway.managementState` |
| UI 모듈 | `maas` (대시보드에 등록됨, `maas-ui` 파드 Running) — 백엔드가 없어 화면만 있음 |
| 관련 CRD | `llminferenceservices.serving.kserve.io`, `llminferenceserviceconfigs` (이미 설치됨) |
| 게이트웨이 | `data-science-gateway` (Gateway API, Istio) — 이미 동작 중 |

## 켜는 데 실제로 필요했던 것 — DSC 스위치 하나로 끝나지 않습니다

DSC에서 `aigateway: Managed` + `modelsAsAService: Managed`로 바꾸면 `ai-gateway-operator`와
`maas-controller`가 뜹니다. 그다음 **전제조건 세 개가 차례로 드러났고, 셋 다 오퍼레이터가
만들어 주지 않습니다.** 각각 상태 메시지가 정확히 무엇이 없는지 알려 줬습니다.

| 순서 | 멈춘 곳 | 메시지 | 해결 |
|---|---|---|---|
| ① | AITenant | `gateway openshift-ingress/maas-default-gateway not found: the Gateway must be created by a network or cluster administrator` | 기존 `data-science-gateway`를 본떠 **Gateway 생성** (같은 클래스·TLS, allowedRoutes에 MaaS 네임스페이스 추가) |
| ② | MaasTenantConfig | `dependency missing: AuthConfig CRD (authorino.kuadrant.io/v1beta3) not available` | **Red Hat Connectivity Link**(`rhcl-operator`, redhat-operators) 설치 + `Kuadrant` CR. 75초, authorino·limitador·dns 오퍼레이터가 함께 옴 |
| ③ | MaasTenantConfig | `database Secret 'maas-db-config' not found in namespace 'redhat-ai-gateway-infra'. Create the Secret with key 'DB_CONNECTION_URL'` | **PostgreSQL 16 배포** + 연결 시크릿 |

> **순서대로만 보입니다.** ①을 고치기 전엔 ②가, ②를 고치기 전엔 ③이 보이지 않습니다.
> 한 번에 다 알려 주지 않으니, 켜기 전에 이 표를 체크리스트로 쓰세요.

> ②에서 **community의 `kuadrant-operator`가 아니라 redhat-operators의 `rhcl-operator`**를
> 쓰세요. AllNamespaces 전용이라 **OperatorGroup을 같이 만들어야** 합니다 — 없으면 Subscription이
> InstallPlan도 없이 조용히 멈춥니다.

이 결과로 클러스터의 DB가 하나 늘었습니다 → [07-부가-리소스.md](07-부가-리소스.md)

> 컴포넌트 이름이 `aigateway`이고 그 아래 `modelsAsAService`입니다(철자 "AsA" 의도적).
> 예전 `kserve.modelsAsService` 필드는 deprecated입니다 — 문서 검색 시 두 이름이 섞여
> 나옵니다.

## MaaS가 답하는 질문

카탈로그·레지스트리와 나란히 놓으면 역할이 분명해집니다.

| | 묻는 것 | 답하는 사람 |
|---|---|---|
| 모델 카탈로그 | 어떤 모델이 있고 어디서 검증됐나 | 플랫폼 (Red Hat) |
| 모델 레지스트리 | 우리가 무엇을 들여와 어느 버전을 쓰나 | 플랫폼 팀 |
| **MaaS** | **누가 어떤 모델을 얼마나 쓸 수 있나** | **플랫폼 팀 → 사용 팀** |

MaaS는 배포된 모델 앞에 **게이트웨이**를 두고 다음을 제공합니다.

- **하나의 진입점** — 팀은 모델별 주소를 몰라도 됩니다. 게이트웨이 주소 + 모델 이름.
- **API 키 발급** — 사용자가 셀프서비스로 키를 만들고 자기 할당량을 봅니다.
- **티어·할당량** — 팀별 요청 수·토큰 한도. 한 팀이 GPU를 독점하지 못하게.
- **사용량 기록** — 누가 어떤 모델을 얼마나 썼는지.

## 데모 시나리오 (MaaS를 켰다고 가정, 약 4분)

이 데모의 앞 두 축과 이어지게 짰습니다 — "검증을 통과한 모델을 팀에 내준다".

### 장면 1 — 모델을 서비스로 등록 (플랫폼 팀 시점)

검증 파이프라인을 **통과한** `qwen3-32b-awq`를 MaaS에 노출합니다.

*멘트:* 3번에서 기각된 35B는 여기 오지 못합니다. 검증 게이트와 서비스 노출이 이어져
있어야 "검증"이 의미가 있습니다.

### 장면 2 — API 키 발급 (사용 팀 시점)

MaaS 화면에서 사용자가 키를 만들고 할당량을 확인합니다.

*멘트:* 사용 팀은 GPU가 어디 있는지, 모델이 어느 네임스페이스에 떠 있는지 몰라도
됩니다. 키 하나와 주소 하나.

### 장면 3 — 호출과 한도

```bash
curl https://<gateway>/v1/chat/completions \
  -H "Authorization: Bearer <api-key>" \
  -d '{"model":"qwen3-32b-awq","messages":[...],"chat_template_kwargs":{"enable_thinking":false}}'
```

한도를 넘기면 `429`. *멘트:* 한 팀의 부하가 다른 팀을 밀어내지 않습니다.

### 장면 4 — 사용량 (플랫폼 팀 시점)

팀별 사용량. *멘트:* GPU 한 장의 비용을 누가 쓰는지 보입니다 — 7단계 GPU 관측과 이어집니다.

## 이 데모와 이어지는 한 문장

> **카탈로그에서 고르고, 검증으로 걸러서, 레지스트리에 기록하고, MaaS로 내준다.**

## 켜기 전 확인할 것

- [ ] `aigateway` 를 Managed로 바꾸면 어떤 오퍼레이터가 추가되는지 (kuadrant 계열 예상 — 확인 필요)
- [ ] 기존 `data-science-gateway` 라우팅에 영향이 없는지 (KServe RawDeployment 경로)
- [ ] `LLMInferenceService`로 재배포가 필요한지, 기존 `InferenceService`를 그대로 노출할 수 있는지
- [ ] 티어·할당량을 어디서 정의하는지 (CR인지 UI인지)
- [ ] GPU 1장이라 동시 사용 팀 시연은 부하 제한 시연으로 대체

> **솔직하게**: MaaS 백엔드는 켜서 Ready를 확인했습니다. 다만 **위 4장면의 화면 흐름
> (모델 노출 → 키 발급 → 호출 → 사용량)은 아직 돌려 보지 않았습니다.** 다음 단계는
> `qwen3-32b-awq`를 `MaaSModelRef`로 노출하고 키 발급·호출·429까지 한 번 통과시키는 것입니다.

### 새로 생긴 MaaS CRD (확인됨)

| CRD | 역할 |
|---|---|
| `maasmodelrefs` | 어떤 모델을 MaaS로 노출할지 (`modelRef` 필수) |
| `maassubscriptions` | 누가 어떤 모델을 쓰나 (`modelRefs`, `owner` 필수) |
| `maasauthpolicies` | 인증·과금 메타데이터 (`modelRefs`, `subjects` 필수) |
| `externalmodels` | 외부 모델(다른 공급자 API)을 같은 게이트웨이로 |
| `maastenantconfigs` | API 키 만료, maas-api 규모, 텔레메트리 |
| `aitenants` | 테넌트 부트스트랩 — 게이트웨이·OIDC |
