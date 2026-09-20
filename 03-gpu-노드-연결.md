# GPU 노드 — 이 계정에서 실제로 돌릴 수 있는 것

AWS 계정 `<AWS_ACCOUNT_ID>` / `us-east-2` 기준, 2026-09-14 실측.

> 이 문서의 `scripts/`, `manifests/` 경로는 모두 **저장소 루트 기준**입니다.
> 명령은 `ocp-ai/` 에서 실행하세요.

## 결론

H200과 H100을 둘 다 요청했지만 이 계정에서는 온디맨드로 확보할 수 없었습니다.
그래서 클러스터는 **`g6e.2xlarge` 1대(NVIDIA L40S 48 GB × 1)**, 시간당 $2.24로 돌아갑니다.

**상태: 2026-09-14 설치 및 검증 완료.**

| | |
|---|---|
| 노드 | `<GPU_NODE>` (롤 `gpu,worker`, us-east-2a) |
| MachineSet | `<INFRA_ID>-gpu-us-east-2a` |
| GPU | NVIDIA L40S, Ada Lovelace, 46068 MiB, compute capability 8.9 |
| 드라이버 / CUDA | 580.126.20 / 런타임 13.0 |
| 오퍼레이터 | NFD 4.20.0, NVIDIA GPU Operator 26.3.3 (ClusterPolicy `ready`) |
| 스케줄링 | `nvidia.com/gpu` capacity 1, allocatable 1 |
| 테인트 | `nvidia.com/gpu=true:NoSchedule` |
| RHOAI 프로파일 | `redhat-ods-applications`의 HardwareProfile `l40s-gpu` |

### 수행한 검증

- `nvidia.com/gpu: 1`을 요청한 파드에서 `nvidia-smi` — L40S, 46068 MiB 인식
- GPU Operator 자체 검사: `cuda workload validation is successful`
- `dcgmi diag -r 1` — software Pass, GPU0 Pass, DCGM 4.5.2, 디바이스 ID `26b9`
- GPU Feature Discovery 라벨: `nvidia.com/gpu.product=NVIDIA-L40S`,
  `nvidia.com/gpu.family=ada-lovelace`, `nvidia.com/gpu.count=1`
- `nvidia-gpu-operator` 파드 10개 Running/Completed

`nvidia-dcgm-exporter`가 롤아웃 중 두 번 재시작(exit 1)한 뒤 안정됐습니다. 프로브가 45초
지연 후 :9400의 `/health`를 치는데 초기 시도가 드라이버 로딩과 경합한 것입니다. 설정 오류가
아니며 `1/1 Running`이고 ClusterPolicy는 `ready`입니다. 재시작 횟수가 계속 오를 때만
들여다보면 됩니다.

## 근거

| 경로 | 결과 |
|---|---|
| `p5en.48xlarge` (H200 ×8) 온디맨드 | us-east-2a·2b·2c 전부 `InsufficientInstanceCapacity` |
| `p5.48xlarge` (H100 ×8) 온디맨드 | us-east-2a·2b·2c 전부 `InsufficientInstanceCapacity` |
| `p5e.48xlarge` (H200 ×8) 온디맨드 | 온디맨드 SKU 자체가 없음 — Capacity Block 전용 |
| 스팟 (P5 계열 전부) | **AWS Organizations SCP로 명시적 거부** |
| `g6e.2xlarge` (L40S ×1) 온디맨드 | us-east-2a에서 첫 시도 성공 |

쿼터는 한 번도 제약이 아니었습니다. P 계열 한도는 384 vCPU이고
`p5en.48xlarge`/`p5.48xlarge`는 192 vCPU인데 P 인스턴스는 0대 실행 중이었습니다.

### 이 계정에서 스팟은 쓸 수 없습니다

```
ec2:RunInstances ... with an explicit deny in a service control policy:
arn:aws:organizations::<ORG_ID>:policy/<ORG>/service_control_policy/<POLICY_ID>
```

이 거부는 Red Hat Demo Platform의 AWS Organization, 즉 이 계정보다 상위에 있어서
계정 내 IAM 변경으로는 풀 수 없습니다. 스팟은 온디맨드와 별도 용량 풀을 쓰므로
원래는 H200을 확보할 가장 유력한 경로였다는 점에서 알아둘 가치가 있습니다
(시간당 약 $27 대 온디맨드 $63.30).

온디맨드 시도는 인가를 *통과해서* 용량 검사에서 실패했으므로, SCP는 인스턴스 타입이
아니라 스팟 시장을 겨냥한 것입니다.

### AWS의 "다른 AZ를 써보라"는 안내는 신뢰할 수 없습니다

두 실패가 서로 모순되는 AZ를 추천했습니다:

- us-east-2a에서: "us-east-2b, us-east-2c를 선택하면 용량을 얻을 수 있다"
- us-east-2b에서: "us-east-2a, us-east-2c를 선택하면 ..."

해당 타입이 *제공되는* 다른 AZ를 나열할 뿐, 여유 용량이 있는 AZ가 아닙니다. 용량을
확인하는 유일한 방법은 실제로 기동을 시도하는 것이고, `scripts/06-gpu-find-capacity.sh`가
그 일을 합니다.

## Capacity Block — 여기서 H100/H200으로 가는 유일한 길

온디맨드가 없을 때도 예약 용량에는 슬롯이 있었습니다 (1대, 24시간):

| 타입 | GPU | AZ | 최단 시작 | 선불 |
|---|---|---|---|---|
| `p5.48xlarge` | H100 80 GB × 8 | us-east-2b | 2026-09-14 11:30 UTC | **$996.67** |
| `p5en.48xlarge` | H200 141 GB × 8 | us-east-2a | 2026-09-18 11:30 UTC | **$1,318.08** |
| `p5e.48xlarge` | H200 141 GB × 8 | us-east-2c | 2026-10-04 11:30 UTC | $1,146.24 |

AWS 콘솔에서 선불 구매가 필요합니다. 블록이 활성화되면 `providerSpec`의
`capacityReservationId`를 설정하고(워커 spec 복제본에 빈 값으로 이미 존재합니다)
해당 타입·AZ로 `scripts/02-gpu-machineset.sh`를 다시 실행하면 연결됩니다.

## us-east-2 온디맨드 GPU 가격

| 타입 | GPU | 시간당 | 일 |
|---|---|---|---|
| `p5en.48xlarge` | H200 141 GB × 8 | 63.30 | 1,519 |
| `p5.48xlarge` | H100 80 GB × 8 | 55.04 | 1,321 |
| `p4d.24xlarge` | A100 40 GB × 8 | 21.96 | 527 |
| `g6e.12xlarge` | L40S 48 GB × 4 | 10.49 | 252 |
| `g6e.4xlarge` | L40S 48 GB × 1 | 3.00 | 72 |
| **`g6e.2xlarge`** | **L40S 48 GB × 1** | **2.24** | **54** |
| `g5.2xlarge` | A10G 24 GB × 1 | 1.21 | 29 |

`p4de.24xlarge`(A100 80 GB)는 `p5e`와 마찬가지로 온디맨드 SKU가 없습니다.

## 나중에 H200 재시도

```bash
source scripts/00-env.sh
./scripts/06-gpu-find-capacity.sh p5en.48xlarge:us-east-2a \
  p5en.48xlarge:us-east-2b p5en.48xlarge:us-east-2c
```

조합을 차례로 시도해 처음 기동되는 것에서 멈추고, 실패한 것은 정리합니다. 실패 시
아무것도 남지 않으므로 재시도 비용은 없습니다. 둘 다 원하지 않으면 L40S MachineSet을
먼저 0으로 줄이세요.

P 계열 온디맨드 용량은 장기 약정 고객에게 우선 배정되는 경향이 있어 며칠간 안 잡힐 수도
있습니다. 예측 가능한 선택지는 Capacity Block입니다.

## 기억해둘 버그 하나

첫 `g6e` 실행에서 3개 AZ 전부 "용량 없음"으로 보고됐는데 **틀린 판정**이었습니다.
탐색 스크립트가 `FailedCreate` 이벤트를 MachineSet 이름 접두사로 매칭했는데, 앞선
p5en/p5 시도가 같은 MachineSet 이름을 재사용했습니다. 쿠버네티스는 이벤트를 약 한 시간
보존하므로 g6e 시도가 이전 인스턴스 타입의 판정을 물려받았고, 스크립트는 정상
프로비저닝 중이던 MachineSet을 지웠습니다.

이제 정확한 머신 이름(고유 접미사 포함)으로 매칭하고 시도 시작 시각보다 오래된 이벤트는
무시합니다. H200/H100 판정은 영향받지 않았습니다 — 그 이벤트들은 올바른 인스턴스 타입을
올바른 시각에 지목하고 있었습니다. 수정 후 `g6e.2xlarge`는 us-east-2a에서 첫 시도에
기동됐습니다.

## OpenShift 콘솔에서 GPU가 보이는 곳

**콘솔 내비게이션에 GPU 항목은 없고, 있어야 할 이유도 없습니다.** NVIDIA GPU Operator는
콘솔 플러그인을 제공하지 않습니다 — 이 클러스터의 ConsolePlugin은
`networking-console-plugin`과 `monitoring-plugin` 둘뿐입니다. 누락도 설정 오류도 아니며,
GPU 상태는 기존 페이지들에 흩어져 있습니다:

| 보고 싶은 것 | 위치 |
|---|---|
| 드라이버/오퍼레이터 상태, ClusterPolicy | Operators → Installed Operators → NVIDIA GPU Operator |
| 노드, GPU capacity, 테인트 | Compute → Nodes → `<GPU_NODE>…` |
| 노드 라벨 (`nvidia.com/gpu.product` 등) | 해당 노드 → Details 또는 YAML |
| **GPU 메트릭 대시보드** | Observe → Dashboards → **NVIDIA GPU / DCGM** |
| 임의 메트릭 질의 | Observe → Metrics, `DCGM_FI_*` |
| AI 워크로드용 GPU | RHOAI 대시보드 → 하드웨어 프로파일 "NVIDIA L40S (1 x 48GB)" |

### DCGM ServiceMonitor는 이미 동작하고 있었습니다

GPU Operator가 제공하는 모니터링 자산은 전부 **플랫폼** Prometheus에서 살아있습니다.
오퍼레이터가 자기 네임스페이스에 `openshift.io/cluster-monitoring=true`를 붙이기 때문입니다:

```
nvidia-gpu-operator  nvidia-dcgm-exporter          up
nvidia-gpu-operator  gpu-operator                  up
nvidia-gpu-operator  nvidia-node-status-exporter   up
```

- `DCGM_*` 메트릭 23종
- 레코딩 룰(`dcgm.accelerator.metrics`)이 더 친숙한 이름을 제공:
  `accelerator_gpu_utilization`, `accelerator_memory_used_bytes`,
  `accelerator_memory_total_bytes`, `accelerator_power_usage_watts`,
  `accelerator_temperature_celsius`, `accelerator_sm_clock_hertz`,
  `accelerator_memory_clock_hertz`
- 드라이버 롤아웃·NFD·reconciliation 실패에 대한 알림 룰 11종

이 중 어느 것에도 user workload monitoring은 **필요 없습니다.**

### 커스텀 GPU 알림은 사용자 네임스페이스에 두면 안 됩니다

`manifests/61-gpu-alerts.yaml`은 오퍼레이터가 제공하지 않는 알림 6종을 추가합니다:
`GPUNodeUnallocated`, `GPUAllocatedButIdle`, `GPUHighTemperature`,
`GPUMemoryNearlyFull`, `GPUUncorrectableRemappedRows`, `GPUMetricsMissing`.

이들은 의도적으로 별도 네임스페이스가 아니라 `nvidia-gpu-operator`에 있습니다.
처음에는 user workload monitoring을 켠 채 `gpu-monitoring` 네임스페이스에 넣었는데
**전부 조용히 죽어 있었습니다.** UWM은 사용자 정의 규칙의 모든 메트릭 셀렉터에
`namespace="<규칙의 네임스페이스>"`를 주입합니다. Thanos Ruler에서 실제 평가식을
꺼내 보면 이렇습니다:

```
DCGM_FI_DEV_GPU_TEMP{namespace="gpu-monitoring"} > 85
kube_node_status_allocatable{namespace="gpu-monitoring",resource="nvidia_com_gpu"}
```

DCGM 시리즈는 `namespace="nvidia-gpu-operator"`를 달고 있고 `kube_node_*` 시리즈에는
namespace 라벨이 아예 없어서 아무것도 매칭되지 않습니다 — 그런데도 모든 규칙이
`health=ok`에 오류 없음을 보고합니다. **실패 신호가 전혀 없습니다.**

플랫폼 모니터링 네임스페이스로 옮기자 표현식이 원문 그대로 적재되고
`GPUNodeUnallocated`가 즉시 `state=pending`에 활성 알림 1건이 되어, 아무도 요청하지 않는
유휴 L40S를 정확히 잡아냈습니다.

**일반화하면**: 플랫폼 메트릭을 질의하는 알림은 플랫폼 모니터링 네임스페이스에 있어야
합니다. 그리고 새 규칙은 적용이 깨끗하게 됐는지가 아니라 **rules API에서 `query` 필드를
다시 읽어** 검증해야 합니다.

### NFD 네임스페이스도 같은 라벨이 필요했습니다

`openshift-nfd`에는 `nvidia-gpu-operator`와 달리 `openshift.io/cluster-monitoring` 라벨이
없었습니다. NFD의 ServiceMonitor는 플랫폼 Prometheus 안에만 존재하는 경로
`tlsConfig.caFile: /etc/prometheus/configmaps/serving-certs-ca-bundle/service-ca.crt`를
고정하고 있고, `NFDDegraded` 규칙은 Thanos Ruler로 라우팅되어 같은 namespace 재작성을
당했습니다 — 즉 NFD의 유일한 헬스 알림도 죽어 있었습니다.
`manifests/62-nfd-platform-monitoring.yaml`이 라벨을 추가하며, 이제 `NFDDegraded`가
`nfd_degraded_info == 1` 로 플랫폼 Prometheus에 적재됩니다.

NFD *규칙*은 고쳐졌습니다. 작성 시점에 NFD 메트릭 *타깃*은 아직 올라오지 않아서
NFD 스크레이프 메트릭은 더 봐야 할 수 있습니다.

### user workload monitoring: 켰지만 GPU 때문은 아닙니다

`manifests/60-user-workload-monitoring.yaml`이 활성화합니다. 이것이 무엇을 주고
무엇을 주지 않는지 분명히 하면:

- **GPU 메트릭과는 무관합니다** — 그것들은 처음부터 플랫폼 범위였습니다.
- **본인이 직접 만든** ServiceMonitor와 PrometheusRule이 동작하게 만듭니다. 없으면
  오류 없이 무시됩니다.
- 앞으로 RHOAI에 중요합니다: KServe InferenceService와 vLLM은 사용자 네임스페이스에서
  돌고, 이미 설치된 RHOAI 모델 서빙 대시보드는 UWM만 수집할 수 있는 `vllm:*` 메트릭을
  질의합니다.

파드 5개(Prometheus 2, Thanos Ruler 2, 오퍼레이터 1) 비용입니다. 워커는 활성화 전
CPU 요청 33~37%, 메모리 17~30%였으므로 여유가 있습니다. 되돌리려면
`openshift-monitoring`의 `cluster-monitoring-config`를 삭제하세요.

UWM이 드러낸 기존 문제 하나: RHOAI의
`data-science-pipelines-operator-service-monitor`가 이제 **down** 타깃을 만듭니다 —
`:8080/metrics`에서 `context deadline exceeded`. 엔드포인트는 파드 내부에서 HTTP 200이지만
파드 IP로는 응답하지 않고, `manager` 컨테이너는 포트 선언도 metrics 인자도 없어서
localhost에 바인딩하고 있습니다. RHOAI 자신의 오퍼레이터 관리 매니페스트라 패치해도
되돌려지며, UWM 이전에는 아예 스크레이프되지 않았습니다.

### 대시보드

`scripts/08-gpu-dashboard.sh`가 `manifests/50-gpu-console-dashboard.yaml`을 생성합니다.
`openshift-config-managed`에 `console.openshift.io/dashboard=true` 라벨이 붙은 ConfigMap이며,
6개 행 22개 패널로 구성됩니다: 요약 게이지, 연산(점유율 대 텐서 파이프 활성도), 프레임버퍼,
온도/전력/클럭, PCIe와 에러 카운터, 클러스터 전체 GPU 할당.

바꾸려면 스크립트를 다시 실행하면 매니페스트를 새로 쓰고 재적용합니다:

```bash
source scripts/00-env.sh
./scripts/08-gpu-dashboard.sh
```

29개 패널 쿼리를 전부 Thanos에 대해 확인했고 데이터를 반환합니다. 그중 둘
(`kube_pod_container_resource_requests{resource="nvidia_com_gpu"}`)은 GPU를 요청하는
파드가 없으면 비는데, 실제로 파드를 띄워 시리즈가 값 1로 나타나는 것을 확인했습니다.
클러스터 전체 합계에는 `or vector(0)`을 넣어 빈 화면 대신 0으로 표시됩니다.

검증에 대한 단서 하나: 쿼리·ConfigMap 등록·대시보드 스키마는 모두 직접 확인했고
스키마는 이 콘솔이 이미 렌더링하는 RHOAI 대시보드와 동일합니다. 다만 **페이지 자체를
브라우저로 열어보지는 못했습니다** — 대화형 로그인이 필요하기 때문입니다.

이 대시보드에서 함께 읽어야 할 두 지표: **GPU utilisation**은 커널이 올라가 있던 시간을
세고, **tensor pipe activity**는 실제 행렬 연산량을 셉니다. 사용률이 100%에 가까운데
텐서 활성도가 0에 가까우면 다른 곳이 병목이라는 신호이고 — 대개 데이터 로더이며,
같은 화면의 PCIe 수신량이 지속적으로 높게 나타납니다.

## RHOAI에서 GPU 사용하기

`manifests/44-rhoai-gpu-hardwareprofile.yaml`이 `l40s-gpu`라는 HardwareProfile을 추가합니다.
MachineSet이 노드에 `nvidia.com/gpu=true:NoSchedule` 테인트를 걸어 일반 워크로드를 막기
때문에 RHOAI 파드에는 대응하는 톨러레이션이 필요합니다 — 없으면 GPU 워크벤치가
untolerated-taint 오류로 Pending에 머뭅니다. RHOAI 3.x는 그 톨러레이션을
`spec.scheduling.node`에 담습니다:

```yaml
spec:
  identifiers:
  - identifier: nvidia.com/gpu
    resourceType: Accelerator
    defaultCount: 1
    minCount: 1
    maxCount: 1
  scheduling:
    type: Node
    node:
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
      - key: nvidia.com/gpu
        operator: Exists
        effect: NoSchedule
```

워크벤치를 만들거나 모델을 배포할 때 하드웨어 프로파일로
"NVIDIA L40S (1 x 48GB)"를 고르세요. CPU/메모리 상한은 노드의 8 vCPU / 64 GiB보다 낮게
잡아 드라이버·device-plugin·DCGM 데몬셋이 들어갈 자리를 남겨 둡니다.

일반 파드도 같은 톨러레이션으로 GPU를 직접 쓸 수 있습니다:

```yaml
tolerations:
- key: nvidia.com/gpu
  operator: Exists
  effect: NoSchedule
resources:
  limits:
    nvidia.com/gpu: 1
```

## 설정을 잃지 않고 GPU만 끄기

비용이 드는 것은 노드뿐이고 오퍼레이터는 무료입니다. 쓰지 않을 때 0으로 줄였다가
필요할 때 되돌리면 드라이버는 자동으로 다시 설치됩니다:

```bash
source scripts/00-env.sh
oc -n openshift-machine-api scale machineset <INFRA_ID>-gpu-us-east-2a --replicas=0
oc -n openshift-machine-api scale machineset <INFRA_ID>-gpu-us-east-2a --replicas=1
```

## 나중에 GPU를 추가할 때

`scripts/02-gpu-machineset.sh`가 워커 MachineSet을 GPU용으로 복제합니다(인스턴스 타입,
AZ 고정, 큰 AI 이미지를 위한 300 GiB 루트 볼륨, `node-role.kubernetes.io/gpu` 라벨,
`nvidia.com/gpu=true:NoSchedule` 테인트):

```bash
./scripts/02-gpu-machineset.sh g6e.2xlarge us-east-2a 1
oc apply -f manifests/05-machineset-gpu.yaml
```

GPU를 켜면 Node Feature Discovery와 NVIDIA GPU Operator 설치도 필요하고,
`manifests/32-dsc.yaml`에서 `ray`/`kueue`/`trainer`/`trainingoperator`를 다시
`Managed`로 바꾸는 것도 고려하세요.

---

이전: [2단계 — OpenShift AI 설치](02-openshift-ai-설치.md) · 다음: [4단계 — 모델 배포 시나리오](04-모델-배포-시나리오.md)
