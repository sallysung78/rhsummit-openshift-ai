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
`maas-controller`가 뜹니다. 그다음 **전제조건이 차례로 드러났고(①~④는 켜는 단계, ⑤~⑦은 모델을 내주는 단계), 셋 다 오퍼레이터가
만들어 주지 않습니다.** 각각 상태 메시지가 정확히 무엇이 없는지 알려 줬습니다.

| 순서 | 멈춘 곳 | 메시지 | 해결 |
|---|---|---|---|
| ① | AITenant | `gateway openshift-ingress/maas-default-gateway not found: the Gateway must be created by a network or cluster administrator` | 기존 `data-science-gateway`를 본떠 **Gateway 생성** (같은 클래스·TLS, allowedRoutes에 MaaS 네임스페이스 추가) |
| ② | MaasTenantConfig | `dependency missing: AuthConfig CRD (authorino.kuadrant.io/v1beta3) not available` | **Red Hat Connectivity Link**(`rhcl-operator`, redhat-operators) 설치 + `Kuadrant` CR. 75초, authorino·limitador·dns 오퍼레이터가 함께 옴 |
| ③ | MaasTenantConfig | `database Secret 'maas-db-config' not found in namespace 'redhat-ai-gateway-infra'. Create the Secret with key 'DB_CONNECTION_URL'` | **PostgreSQL 16 배포** + 연결 시크릿 |

| ④ | Gen AI Studio 에서 **`maas-api is not available`** | maas-api 로그: `Gateway has no external hostname configured` · HTTPRoute: `namespace "redhat-ai-gateway-infra" is not allowed by the parent` | Gateway listener 에 **`hostname: maas.apps.<도메인>`** + **passthrough Route** + `allowedRoutes` 에 `redhat-ai-gateway-infra` 추가 |

> ④는 백엔드가 Ready 인데 **UI에서만** 드러납니다. `MaasTenantConfig Ready=True` 여도
> 게이트웨이에 외부 호스트명이 없으면 maas-api 의 `/v1/tenants` 가 500 을 내고, 대시보드는
> "not available"을 띄웁니다. 확인: `curl https://maas.apps.<도메인>/maas-api/health` → 200,
> 키 없이 `/v1/models` → 401 이면 게이트웨이·인증이 모두 정상입니다.
>
> **Route 는 reencrypt 가 아니라 passthrough** 여야 합니다. listener 에 hostname 을 주면
> Istio 가 SNI 로 걸러내는데, reencrypt 에서는 라우터가 그 이름으로 SNI 를 보내지 않습니다.
> 인증서는 같은 네임스페이스의 `router-certs-default`(`*.apps` 와일드카드)를 씁니다.
>
> 대시보드 게이트웨이의 인프라 ConfigMap(`data-science-gateway-config`)을 **같이 쓰지 마세요.**
> 같은 이름의 serving-cert 를 두 서비스가 요청해 충돌합니다(`serving-cert-generation-error`).

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

## 모델을 MaaS로 내주기 — 실제로 필요했던 것 (2026-09-27)

Gen AI Studio에서 키를 만들려 하면 **"No subscriptions available"** 이 나옵니다. 키는 구독에
속해야 만들어지고, 구독에는 **모델이 최소 1개** 있어야 합니다(`modelRefs minItems=1`).

### ⑤ MaaS에 등록할 수 있는 모델은 `LLMInferenceService`뿐입니다

`MaaSModelRef`가 받는 종류는 `LLMInferenceService` 또는 `ExternalModel`입니다. **일반
`InferenceService`는 등록할 수 없습니다.** `ExternalModel`은 endpoint가 포트 없는 FQDN만
받아서 클러스터 안 모델(8080 포트)을 가리킬 수 없습니다.

그래서 운영 32B를 **같은 S3 가중치·같은 vLLM 인자로 `LLMInferenceService`로 옮겼습니다**
(다운타임 5분). 달라진 점:

| | InferenceService (이전) | LLMInferenceService (지금) |
|---|---|---|
| 가중치 | `storage.key` | `model.uri: s3://…` + ServiceAccount 시크릿 |
| vLLM | 헤드리스 서비스, **8080 평문** | `…-kserve-workload-svc`, **8000 TLS**(서비스 CA) |
| 라우팅 | 없음 | 게이트웨이 → **InferencePool** → 엔드포인트 피커 → vLLM |
| 외부 주소 | 없음 | `https://maas.apps.<도메인>/rhsummit/qwen3-32b-awq/v1/…` |
| `--trust-remote-code` | 런타임 기본값으로 켜짐 | **꺼짐** |

그다음 **모델 등록 → 구독(소유자 `kube:admin`, 시간당 10만 토큰) → 인증 정책**을 만듭니다.

### ⑥ KServe의 게이트웨이 인증 정책이 MaaS 정책을 덮어씁니다

LLMInferenceService가 MaaS 게이트웨이를 참조하면 odh-model-controller가 그 게이트웨이에
`<게이트웨이>-authn` 정책(쿠버네티스 토큰 검사)을 자동으로 붙입니다. 이것이 MaaS의 API 키
정책을 **덮어써서**(`Enforced=False, Overridden`) MaaSAuthPolicy가 `Pending`에 멈춥니다.

> 이 동작은 **Kuadrant(Connectivity Link)를 설치한 뒤에야 켜집니다.** 컨트롤러 시작 로그에
> `authPolicyCRD: false`가 찍혀 있다가, ②에서 설치하면서 켜진 것입니다.

**해결**: 컨트롤러는 같은 이름의 정책이라도 **자기 관리 라벨이 없으면 건너뜁니다**
(`Skipping reconciliation - AuthPolicy is not managed by odh-model-controller`). 이름은 그대로
두고 라벨과 대상을 바꿔, 존재하지 않는 라우트를 가리키는 **자리표시자**로 교체했습니다.
삭제하면 수 초 안에 다시 생기므로 반드시 **교체**해야 합니다.

> ⚠️ **`security.opendatahub.io/enable-auth: "false"` 를 쓰면 안 됩니다.** 처음에 이렇게
> 풀려고 했는데, KServe가 라우트 정책을 **"Anonymous access"** 로 바꿔서 **키 없이 누구나
> 모델을 호출할 수 있게 됐습니다.** 7분 만에 발견해 되돌렸고, 그 사이 추론 요청은 제 테스트
> 1건뿐이었습니다. 인증 설정을 바꿀 때는 **바꾼 직후 키 없는 호출이 401인지 반드시**
> 확인하세요.

### ⑦ Authorino가 maas-api 인증서를 신뢰하지 않습니다

발급받은 키로 불러도 **403**이 납니다. Authorino가 키를 검증하려고 maas-api
(`…/internal/v1/api-keys/validate`)를 HTTPS로 부르는데, maas-api 인증서는 **서비스 CA**가
발급한 것이라 Authorino가 신뢰하지 않습니다. maas-api 로그에 `tls: bad certificate`가
찍힙니다.

**해결**: `kuadrant-system`에 이미 주입돼 있는 `openshift-service-ca.crt` ConfigMap을 Authorino
CR의 `spec.volumes`로 `/etc/pki/tls/certs`에 마운트합니다. 공인 CA 번들은 다른 경로에서 계속
읽히므로 외부 인증서 검증은 영향이 없습니다.

### 결과

| 호출 | 결과 |
|---|---|
| 발급한 키 | **200** — 답과 `usage`(토큰 수)가 함께 옴 |
| 잘못된 키 | 403 |
| 폐기한 키 | 403 |
| 키 없음 | 401 |

## 데모 — 키 발급과 사용량

1. **Gen AI Studio**(AI hub 아래 MaaS 화면)에서 구독 `rhsummit-demo`가 보이는지 확인
2. **API 키 발급** — 키는 `sk-oai-…` 형식, 발급 순간에만 전체가 보입니다. 복사해 두세요
3. 터미널에서 호출:

```bash
curl -sk https://maas.apps.<도메인>/rhsummit/qwen3-32b-awq/v1/chat/completions \
  -H "Authorization: Bearer sk-oai-…" -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-32b-awq",
       "messages":[{"role":"user","content":"OpenShift AI를 한 문장으로 설명해줘."}],
       "max_tokens":80, "chat_template_kwargs":{"enable_thinking":false}}'
```

4. 몇 번 호출한 뒤 **사용량 화면**을 새로고침. 토큰 한도는 시간당 10만입니다.

> 사용량 집계는 텔레메트리 기본값(켜짐)과 Limitador 수집 주기(30초)를 따릅니다. 호출 직후
> 비어 있으면 30초~1분 뒤 다시 보세요. **화면에 실제로 숫자가 뜨는 것까지는 제가 확인하지
> 못했습니다** — 리허설 때 한 번 돌려 보세요.

> **확인한 범위 (2026-09-28)**: 모델 노출 → 키 발급 → 호출(200/403/401)까지는 API로 통과했습니다
> (위 "결과" 표). **Gen AI Studio 화면에서 키를 만들고 사용량 숫자가 뜨는 것, 한도 초과 시 429는
> 아직 확인하지 않았습니다.** 리허설 때 화면으로 한 번 돌려 보세요.

### 새로 생긴 MaaS CRD (확인됨)

| CRD | 역할 |
|---|---|
| `maasmodelrefs` | 어떤 모델을 MaaS로 노출할지 (`modelRef` 필수) |
| `maassubscriptions` | 누가 어떤 모델을 쓰나 (`modelRefs`, `owner` 필수) |
| `maasauthpolicies` | 인증·과금 메타데이터 (`modelRefs`, `subjects` 필수) |
| `externalmodels` | 외부 모델(다른 공급자 API)을 같은 게이트웨이로 |
| `maastenantconfigs` | API 키 만료, maas-api 규모, 텔레메트리 |
| `aitenants` | 테넌트 부트스트랩 — 게이트웨이·OIDC |
