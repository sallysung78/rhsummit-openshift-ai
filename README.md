# 빈 AWS 계정에서 OpenShift AI 모델 배포까지

AWS 계정 하나만 있는 상태에서 시작해, OpenShift를 올리고, OpenShift AI를 설치하고,
GPU 노드를 붙이고, 모델을 레지스트리에 등록해 배포하기까지의 전 과정입니다.

모든 수치와 오류 메시지는 2026년 9월 `us-east-2`에서 **실제로 측정하고 겪은 것**이며,
문서를 쓰기 위해 재구성한 것이 아닙니다.

## 순서

| | 문서 | 내용 | 소요 |
|---|---|---|---|
| 1 | [OpenShift 설치](01-openshift-설치.md) | 사전 요건, 쿼터, 리전 선택, IPI 설치 | 약 45분 |
| 2 | [OpenShift AI 설치](02-openshift-ai-설치.md) | EFS RWX 스토리지, RHOAI 3.5 오퍼레이터, DSC | 약 25분 |
| 3 | [GPU 노드 연결](03-gpu-노드-연결.md) | GPU 용량 확보, NFD, NVIDIA GPU Operator, 관측 | 약 30분 |
| 4 | [모델 배포 시나리오](04-모델-배포-시나리오.md) | S3, 모델 선택, 레지스트리 등록, KServe 배포 | 약 40분 |
| 5 | [촬영 대본](05-촬영-대본.md) | 콘솔 조작 순서와 멘트 | — |

처음부터 끝까지 **약 2시간 20분**이고, 대부분은 기다리는 시간입니다.

명령은 모두 저장소 루트(`ocp-ai/`)에서 실행합니다:

```bash
source scripts/00-env.sh
```

## 결과물

| | |
|---|---|
| OpenShift | 4.20.36 (EUS) — 마스터 3 × m6i.xlarge, 워커 3 × m6i.2xlarge |
| OpenShift AI | 3.5.0 Self-Managed — dashboard, workbenches, aipipelines, kserve, modelregistry |
| GPU | 1 × g6e.2xlarge, NVIDIA L40S 48 GB, 드라이버 580.126.20 |
| RWX 스토리지 | AWS EFS → `efs-sc` |
| 모델 스토리지 | S3, 전용 IAM 사용자로 버킷 하나만 접근 |
| 모델 | Qwen3-8B-AWQ (5.7 GiB), Qwen3-32B-AWQ (18.0 GiB) |
| 비용 | 실행 중 일 약 $103, 정지 시 일 $6.93 (실측) |

## 이 문서가 다른 설치 가이드와 다른 점

각 단계에 **실제로 막혔던 지점과 그 진단 과정**이 들어 있습니다. 대부분은 오류
메시지가 원인을 가리키지 않는 종류였습니다:

**RHOAI `stable` 채널은 2.x입니다.** 기본 채널은 `stable-3.x`이고, 3.x는 새 빌드가 아니라
v2 API에 컴포넌트 이름까지 바뀐 다른 물건입니다. 2.x 매니페스트는 적용되지 않습니다.
→ [2단계](02-openshift-ai-설치.md)

**OperatorGroup이 없으면 Subscription은 조용히 멈춥니다.** InstallPlan이 만들어지지
않고 status가 비어서, 설정 오류가 이미지 다운로드 지연처럼 보입니다.
→ [2단계](02-openshift-ai-설치.md)

**AWS의 "다른 AZ를 써보라"는 안내는 신뢰할 수 없습니다.** 여유 용량이 있는 AZ가 아니라
해당 타입이 제공되는 AZ를 나열할 뿐이고, AZ마다 서로 모순되는 답을 줍니다.
→ [3단계](03-gpu-노드-연결.md)

**사용자 네임스페이스의 알림은 플랫폼 메트릭을 볼 수 없습니다.** user workload monitoring이
모든 셀렉터에 `namespace="..."`를 주입해서, 규칙은 `health=ok`를 보고하면서 영원히
발동하지 않습니다. 실패 신호가 전혀 없습니다.
→ [3단계](03-gpu-노드-연결.md)

**GPU는 좋은데 확보가 안 됩니다.** H200/H100은 us-east-2 전 AZ에서 온디맨드 용량이 없고
스팟은 조직 SCP로 차단돼 있었습니다. 쿼터는 한 번도 제약이 아니었습니다.
→ [3단계](03-gpu-노드-연결.md)

## 참고 노트

이 환경을 만들면서 함께 읽은 Red Hat 랩 자료의 한국어 요약입니다. 개념과 실습 의도를
정리한 것이며 단계별 절차는 각 노트의 원문 링크를 보세요.

| 노트 | 내용 |
|---|---|
| [모델 서빙 기초](notes/모델-서빙-기초.md) | 서빙이란 무엇인가, 워크플로, ServingRuntime·InferenceService, 런타임 선택 |
| [AgentOps 모듈 5 — 에이전트·LLM 평가](notes/agentops-05-평가.md) | 관측과 평가의 차이, 스코어러 2계층, Prompt Registry |
| [AgentOps 모듈 6 — 개발에서 운영으로](notes/agentops-06-개발에서-운영으로.md) | 평가 파이프라인, 프롬프트 회귀 탐지, 품질 게이트 |

## 이 저장소에 대하여

문서만 담았습니다. 본문에 나오는 `scripts/`와 `manifests/`는 실제 클러스터를 만들고
운영하는 데 쓴 것이고 이 저장소에는 포함되어 있지 않습니다. 각 단계의 명령이 무엇을
하는지는 본문에 풀어 두었으므로, 스크립트 없이도 손으로 따라갈 수 있습니다.

환경 식별자는 전부 플레이스홀더입니다:

| 표기 | 의미 |
|---|---|
| `<AWS_ACCOUNT_ID>` | AWS 계정 번호 |
| `<BASE_DOMAIN>` | Route 53 퍼블릭 호스티드 존 도메인 |
| `<CLUSTER>` | 클러스터 이름 |
| `<INFRA_ID>` | `openshift-install`이 만드는 `<클러스터>-<난수>` 형태의 인프라 ID |
| `<VPC_ID>` `<EFS_ID>` `<HOSTED_ZONE_ID>` | 해당 AWS 리소스 ID |
| `<GPU_NODE>` | GPU 노드의 이름 |

비밀번호와 키는 어느 문서에도 없습니다.

## 주의

이 환경은 수명과 외부에서 집행되는 예산 한도를 갖는 Red Hat Demo Platform 샌드박스입니다.
둘 다 AWS API로는 보이지 않습니다. 샌드박스가 회수되면 정지해 둔 것도 함께 사라지며,
그 경우 pull secret만 새로 받아 1단계부터 다시 실행하면 같은 환경이 재현됩니다.
