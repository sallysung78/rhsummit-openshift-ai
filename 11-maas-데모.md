# MaaS (Models as a Service) — 별도 데모 정리

카탈로그·레지스트리가 "어떤 모델을 들여와 쓸 것인가"라면, MaaS는 **"그 모델을 조직 안
여러 팀에 어떻게 서비스로 내줄 것인가"** 입니다. 지금 데모에는 없던 축이라 따로 정리합니다.

## 현재 상태 — 이 클러스터에는 꺼져 있습니다

| | |
|---|---|
| DSC 컴포넌트 | `aigateway` = **Removed** |
| 하위 스위치 | `aigateway.modelsAsAService.managementState`, `aigateway.batchGateway.managementState` |
| UI 모듈 | `maas` (대시보드에 등록됨, `maas-ui` 파드 Running) — 백엔드가 없어 화면만 있음 |
| 관련 CRD | `llminferenceservices.serving.kserve.io`, `llminferenceserviceconfigs` (이미 설치됨) |
| 게이트웨이 | `data-science-gateway` (Gateway API, Istio) — 이미 동작 중 |

**켜려면** DSC에서 `aigateway.managementState: Managed` + `modelsAsAService.managementState:
Managed`. 이번 세션에서는 켜지 않았습니다 — 오퍼레이터가 추가로 들어오고 게이트웨이
정책이 바뀌므로 **켜기 전에 리허설 시간을 확보**하세요.

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

> **솔직하게**: 위 시나리오의 화면 흐름은 MaaS를 실제로 켜서 확인한 것이 아닙니다.
> 컴포넌트·CRD·UI 모듈의 존재는 클러스터에서 확인했지만, **화면과 동작은 켠 뒤
> 리허설로 검증**해야 합니다. 켜서 확인하기를 원하시면 별도 작업으로 진행합니다.
