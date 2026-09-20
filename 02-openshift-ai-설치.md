# 2단계 — OpenShift AI 3.5 설치

소요 시간 **약 25분**. RWX 스토리지 준비가 먼저이고, 그다음 오퍼레이터입니다.

## 먼저: RWX 스토리지

RHOAI 워크벤치와 데이터 사이언스 파이프라인은 **ReadWriteMany** 볼륨을 요구합니다.
IPI가 깔아주는 `gp3-csi`는 ReadWriteOnce만 지원하므로 그대로는 안 됩니다.

선택지는 둘입니다:

| | 필요한 것 | 판단 |
|---|---|---|
| **AWS EFS CSI** | 관리형 NFS 엔드포인트, 추가 노드 없음 | 채택 |
| OpenShift Data Foundation | 전용 스토리지 노드 3대 | 샌드박스에는 과함 |

```bash
source scripts/00-env.sh
./scripts/03-efs-setup.sh
```

클러스터 VPC 안에 EFS 파일시스템을 만들고, 프라이빗 서브넷마다 마운트 타깃을 붙이고,
2049 포트를 VPC CIDR에만 여는 보안 그룹을 만든 뒤, `efs-sc` StorageClass 매니페스트를
생성합니다.

### EFS 오퍼레이터의 함정

AWS EFS CSI Driver Operator는 **AllNamespaces 전용**인데, 권장 네임스페이스인
`openshift-cluster-csi-drivers`에는 **OperatorGroup이 없습니다.**

OperatorGroup 없이 Subscription을 만들면 OLM이 InstallPlan을 아예 만들지 않습니다.
`status`는 `CatalogSourcesUnhealthy=False` 하나만 남고 비어 있어서, 카탈로그는 멀쩡한데
이미지를 받는 중인 것처럼 보입니다. 실제로는 설정 오류이고 영원히 기다려도 안 됩니다.

`manifests/20-efs-operator.yaml`의 **첫 객체가 OperatorGroup인 이유**가 이것입니다.

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-cluster-csi-drivers
  namespace: openshift-cluster-csi-drivers
spec: {}   # 빈 spec == AllNamespaces
```

Subscription이 1분이 지나도 `state`/`installedCSV`가 비어 있으면
`oc get og -n <네임스페이스>` 부터 확인하세요.

## 오퍼레이터 채널을 먼저 대조하세요

**적용하기 전에 반드시 실행하세요:**

```bash
./scripts/04-verify-catalog.sh
```

이 단계가 실제로 잡아낸 것: RHOAI의 `stable` 채널은 여전히 **2.x 라인**(2.25.11)을
가리킵니다. 카탈로그의 *기본* 채널은 `stable-3.x` → **3.5.0** 입니다.

매니페스트는 `stable-3.5`로 고정합니다 — 3.5의 z-stream 패치는 받되 3.6으로 조용히
넘어가지 않습니다.

## RHOAI 3.x는 새 빌드가 아니라 다른 API입니다

2.x 기준으로 쓴 매니페스트는 적용되지 않습니다. 확인된 차이:

**저장 버전이 v2입니다** — `DataScienceCluster`, `DSCInitialization` 모두.

**컴포넌트 이름이 바뀌었습니다:**

| 2.x | 3.x |
|---|---|
| `datasciencepipelines` | `aipipelines` |
| `modelmeshserving` | 제거됨 (KServe로 대체) |
| `codeflare` | 제거됨 |
| — | `aigateway`, `llamastackoperator`, `trainer`, `sparkoperator`, `feastoperator`, `mlflowoperator`, `ogx`, `mcplifecycleoperator` 신규 |

**Service Mesh 선택이 사라졌습니다.** 2.x에서는 KServe를 `Serverless`(Service Mesh +
Serverless + Authorino 세 오퍼레이터 필요)와 `RawDeployment` 중에 골라야 했습니다.
3.x에서는 `kserve`에 `defaultDeploymentMode`가 없고 DSCI에 `serviceMesh` 필드 자체가
없습니다. raw deployment가 유일한 방식이고, 남은 설정은
`rawDeploymentServiceConfig: Headless|Headed` 뿐입니다.

워커 3대로 버틸 수 있는 이유가 이것입니다.

> 스키마를 추측하지 마세요. 실제 CRD를 읽으면 됩니다:
> ```bash
> oc get crd datascienceclusters.datasciencecluster.opendatahub.io -o json \
>   | jq '.spec.versions[] | select(.storage==true) | .schema.openAPIV3Schema.properties.spec'
> ```

## 설치

```bash
./scripts/05-install-rhoai.sh
```

순서: EFS CSI 오퍼레이터 → `efs-sc` StorageClass → RHOAI 오퍼레이터 → DSCI → DSC.

### DSCInitialization은 오퍼레이터가 만들어 줍니다

3.5 오퍼레이터는 설치 직후 `default-dsci`를 **스스로 생성합니다.**
`manifests/31-dsci.yaml` 적용은 `applicationsNamespace`나 모니터링을 바꿀 때만 필요합니다.

`monitoring.metrics: {}`(빈 객체)는 Cluster Observability와 Tempo 오퍼레이터 없이도
모니터링 스택을 초기화 상태로 유지합니다. 그 결과 나타나는
`MonitoringStackAvailable=False`, `TempoAvailable=False`, `AlertingAvailable=False`는
**실패가 아니라 정보성 조건**입니다.

### "모든 Ready 조건이 True"를 기다리면 안 됩니다

`Removed`로 설정한 컴포넌트는 `<Name>Ready=False`에 `reason=Removed`를 영원히 보고합니다.
모든 조건이 True가 되는 일은 없습니다.

설치 스크립트가 `status.phase`와 집계 조건 `ComponentsReady`/`ModulesReady`를 기준으로
판정하는 이유입니다.

## 완료 확인

```bash
oc get datasciencecluster default-dsc -o jsonpath='{.status.phase}{"\n"}'   # Ready
oc get datasciencecluster default-dsc -o json | jq -r '
  .status.conditions[] | select(.type|endswith("Ready")) |
  "\(if .status=="True" then "OK  " else "--  " end) \(.type)  \(.reason // "")"'
```

`--` 로 표시되는 것은 전부 `reason=Removed`여야 합니다. 그 외의 False는 진짜 문제입니다.

파드 확인:

```bash
oc -n redhat-ods-applications get pods    # 21개 정도
oc -n rhoai-model-registries get pods     # 2개
```

## 접속

3.x의 정문은 `openshift-ingress`의 `data-science-gateway` 라우트입니다:

```bash
oc -n openshift-ingress get route data-science-gateway -o jsonpath='https://{.spec.host}{"\n"}'
```

→ `https://rh-ai.apps.<클러스터>.<도메인>`

기존 `rhods-dashboard-redhat-ods-applications.apps.<도메인>` 라우트도 남아있지만
게이트웨이로 리다이렉트만 합니다.

## 이 단계에서 켜고 끈 것

GPU가 없는 상태이므로 가속기 스케줄링 컴포넌트는 꺼 두었습니다 —
`ray`, `kueue`, `trainer`, `trainingoperator`, `sparkoperator`.
3단계에서 GPU를 붙인 뒤 필요하면 `manifests/32-dsc.yaml`에서 `Managed`로 바꾸면 됩니다.

켜 둔 것: `dashboard`, `workbenches`, `aipipelines`, `kserve`, `modelregistry`.

---

이전: [1단계 — OpenShift 설치](01-openshift-설치.md) ·
다음: [3단계 — GPU 노드 연결](03-gpu-노드-연결.md)
