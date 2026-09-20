# 1단계 — 빈 AWS 계정에서 OpenShift 4.20까지

소요 시간 **약 45분**. 이 중 42분은 `openshift-install`이 알아서 합니다.

## 시작 전에 확인할 것

빈 AWS 계정이라도 IPI 설치는 다음 네 가지를 전제합니다. 하나라도 없으면 설치가
중간에 실패하므로 먼저 확인하세요.

| 항목 | 확인 방법 | 이 환경의 값 |
|---|---|---|
| Route 53 **퍼블릭** 호스티드 존 | `aws route53 list-hosted-zones` | `<BASE_DOMAIN>` |
| Red Hat pull secret | [console.redhat.com](https://console.redhat.com/openshift/install/pull-secret) | 로컬 파일 |
| IAM 자격증명 | `aws sts get-caller-identity` | `<IAM_ADMIN_USER>` |
| 서비스 쿼터 | 아래 참조 | 여유 충분 |

베이스 도메인이 Route 53에 **퍼블릭 존으로** 존재해야 합니다. IPI는 여기에
`api.<클러스터>.<도메인>`과 `*.apps.<클러스터>.<도메인>` 레코드를 직접 만듭니다.
존이 없으면 설치가 시작조차 되지 않습니다.

### 쿼터

실제로 필요한 양은 생각보다 적습니다. 컨트롤 플레인 3 × 4 vCPU + 워커 3 × 8 vCPU = **36 vCPU**.

| 쿼터 | 코드 | 필요 | 측정값 |
|---|---|---|---|
| Running On-Demand Standard instances | `L-1216C47A` | 36 | 1152 |
| **EC2-VPC Elastic IPs** | `L-0263D0A3` | **3** | **5** |

Elastic IP가 조용한 병목입니다. IPI는 AZ마다 NAT 게이트웨이를 하나씩 만들고 각각
EIP를 소비합니다. 기본 한도가 5라 3-AZ 클러스터는 아슬아슬하게 들어갑니다.

## 리전 선택

나중에 GPU를 붙일 생각이라면 **리전 선택이 그때를 결정합니다.** 클러스터를 만든 뒤에는
리전을 바꿀 수 없고, GPU 노드는 클러스터와 같은 VPC에 있어야 합니다.

이 환경에서 `us-east-2`를 고른 이유는 검토 대상 인스턴스 타입 전부(H200 2종 포함)가
3개 AZ 모두에 제공되는 유일한 리전이었기 때문입니다. 확인 방법:

```bash
aws ec2 describe-instance-type-offerings --location-type availability-zone \
  --filters "Name=instance-type,Values=p5en.48xlarge,g6e.2xlarge,m6i.2xlarge" \
  --region us-east-2 --query 'InstanceTypeOfferings[].[InstanceType,Location]' --output table
```

## install-config.yaml

`install-config.template.yaml`이 출발점입니다. 핵심만 보면:

```yaml
baseDomain: <BASE_DOMAIN>
metadata:
  name: ocpai
controlPlane:
  replicas: 3
  platform:
    aws: { type: m6i.xlarge, zones: [us-east-2a, us-east-2b, us-east-2c] }
compute:
- name: worker
  replicas: 3
  platform:
    aws: { type: m6i.2xlarge, rootVolume: { size: 200, type: gp3 } }
platform:
  aws:
    region: us-east-2
```

**워커를 `m6i.2xlarge`(8 vCPU / 32 GiB)로 잡은 이유**는 OpenShift AI 때문입니다.
기본값인 `m6i.large`로는 RHOAI 컴포넌트가 다 들어가지 않습니다.

## 실행

```bash
source scripts/00-env.sh
./scripts/01-create-cluster.sh
```

스크립트는 설치를 시작하기 전에 프리플라이트를 돕니다 — AWS 신원, Route 53 존 존재,
쿼터 여유, pull secret 형식. 그다음 `install-config.yaml`을 렌더링하고
`openshift-install create cluster`를 실행합니다.

### 설치 전에 검증하고 싶다면

리소스를 하나도 만들지 않고 설정만 실제 AWS API에 대해 검증할 수 있습니다:

```bash
openshift-install create manifests --dir <디렉터리>
```

AZ 존재 여부, 인스턴스 타입 유효성, Route 53 존, 자격증명을 전부 확인합니다.
42분을 날리기 전에 30초로 잡을 수 있는 오류가 많습니다.

## 완료 확인

```bash
source scripts/00-env.sh
oc get clusterversion
oc get nodes
oc get co | grep -v 'True.*False.*False'   # 비어 있으면 전부 정상
```

기대 결과: 노드 6대 Ready(마스터 3 + 워커 3, AZ별 분산), 클러스터 오퍼레이터 34개 정상.

## 접속

설치가 끝나면 콘솔 주소와 `kubeadmin` 비밀번호가 출력됩니다. 비밀번호는
`cluster/auth/kubeadmin-password`에도 저장됩니다.

기본 ingress 인증서는 self-signed이므로 브라우저 경고는 정상입니다.

**`cluster/` 디렉터리를 지우지 마세요.** `metadata.json`과
`.openshift_install_state.json`이 없으면 나중에 `openshift-install destroy`를 실행할 수
없어 AWS를 손으로 정리해야 합니다.

## 겪은 함정

**`00-env.sh`가 zsh에서 깨졌습니다.** 원래 `BASH_SOURCE`를 썼는데 zsh에서는 비어 있어
`source scripts/00-env.sh`가 조용히 잘못된 경로를 계산하고 `KUBECONFIG`를 망가뜨렸습니다.
스크립트로 *실행*할 때는 shebang 덕에 멀쩡해서 더 찾기 어려웠습니다. 지금은 양쪽 셸에서
자기 경로를 해석합니다.

---

다음: [2단계 — OpenShift AI 설치](02-openshift-ai-설치.md)
