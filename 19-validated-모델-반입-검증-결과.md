# 19. Red Hat validated 모델 반입 검증 결과 (2026-10-05)

1차 데모 리뷰에서 "Red Hat validated 모델이 기준 미달로 나온다"는 점이 부정적으로 받아들여졌습니다. 그래서 시나리오를 바꾸고, 그에 맞는 모델을 실제로 배포해 측정했습니다.

## 바뀐 시나리오

| 단계 | 내용 | 결과 |
|---|---|---|
| 커뮤니티 모델 | 필요 메모리와 성능 정보 없이 올려 본다 | **배포 실패** (GPU 메모리 부족) |
| Red Hat validated 모델 | 카탈로그의 최소 VRAM과 성능표를 보고 우리 GPU에 맞는 모델을 고른다 | **배포 성공** |
| 반입 후 검증 | 우리 GPU에서 성능과 정확도를 측정해 기준과 비교한다 | **통과 → 레지스트리 등록** |

## 모델을 고른 근거

우리 GPU는 NVIDIA L40S 48GB 1장입니다. 카탈로그 성능 데이터(모델 122종)를 전부 확인했습니다.

- **L40S로 측정한 모델은 카탈로그에 하나도 없습니다.** 측정 가속기는 H100, H200, A100-80, A100-40, B200, L4 여섯 종입니다.
- 그래서 **최소 VRAM이 48GB보다 충분히 작고, 단일 GPU(A100-40 또는 L4) 측정값이 있는 모델**을 골랐습니다. 우리 GPU가 목록에 없으므로 카탈로그 값은 참고값이고, 반입 후 직접 측정이 필요합니다. 이것이 2차 검증의 이유입니다.

고른 모델은 고객이 많이 쓰는 계열에서 두 개입니다.

| 모델 | 최소 VRAM | 카탈로그 측정 가속기 | 받는 곳 |
|---|---|---|---|
| `RedHatAI/gpt-oss-20b` | 15.9 GB | A100-40, L4, H100, H200, B200 | Red Hat 레지스트리의 모델 이미지 |
| `RedHatAI/Qwen3-8B-FP8-dynamic` | 10.9 GB | A100-40, L4, H100, H200 | Red Hat 레지스트리의 모델 이미지 |

둘 다 카탈로그의 모델 이미지(modelcar) 주소로 바로 배포했습니다. 내려받아 저장소에 올리는 작업이 필요 없었습니다.

## 측정 결과

같은 GPU, 같은 평가 묶음, 같은 기준으로 측정했습니다. 기준은 초당 30토큰 이상, 첫 토큰 200 ms 이하입니다.

| 모델 | 구분 | 배포 | 생성 속도 | 첫 토큰 | 정확도 묶음 | 판정 |
|---|---|---|---|---|---|---|
| Qwen3.6-35B-A3B 원본 | 커뮤니티, 양자화 없음 (67 GiB) | **실패** | - | - | - | 배포 불가 |
| gpt-oss-20b | Red Hat validated | 성공 (첫 배포 13분, 재시작 3분) | **154.0 tok/s** | 19~32 ms | 0.55 (기준 0.5) | **통과** |
| Qwen3-8B-FP8-dynamic | Red Hat validated | 성공 (5분) | **66.9 tok/s** | 26 ms | 0.65 (기준 0.5) | **통과** |

참고로 이전 운영 모델인 커뮤니티 양자화 Qwen3-32B-AWQ는 36.9 tok/s, 첫 토큰 105 ms였습니다.

측정 조건: 입력 256 / 출력 128 토큰, GuideLLM(EvalHub 경유), 한 번에 한 요청씩 보내는 구간의 값입니다.

### 커뮤니티 원본의 실패

`Qwen/Qwen3.6-35B-A3B` 원본을 2026-10-05에 다시 배포해 같은 오류를 재현했습니다. 가중치를 내려받고 GPU에 올리는 데 약 10분이 걸린 뒤 실패합니다.

```
torch.OutOfMemoryError: CUDA out of memory. Tried to allocate 512.00 MiB.
GPU 0 has a total capacity of 44.39 GiB of which 119.31 MiB is free.
```

모델 카드에는 필요 메모리가 적혀 있지 않습니다. 10분을 기다린 뒤에야 안 된다는 것을 압니다. validated 모델은 카탈로그에 최소 VRAM이 있어 배포 전에 판단할 수 있습니다.

지금은 정지 상태로 두었습니다. 배포 목록에 "배포 실패: GPU 메모리 부족"으로 남아 있습니다.

### 카탈로그 값과 비교

| 모델 | 카탈로그 (A100-40 1장, 입력 512 / 출력 256) | 우리 측정 (L40S 1장, 입력 256 / 출력 128) |
|---|---|---|
| gpt-oss-20b | 토큰 간 10.7 ms(약 94 tok/s), 첫 토큰 59 ms | 154.0 tok/s, 첫 토큰 19~32 ms |
| Qwen3-8B-FP8-dynamic | 토큰 간 11.7 ms(약 86 tok/s), 첫 토큰 75 ms | 66.9 tok/s, 첫 토큰 26 ms |

가속기와 입력 길이가 달라 직접 비교는 아닙니다. 같은 자릿수로 나온다는 것까지만 말할 수 있습니다.

### 수치를 말할 때 조심할 것

- **gpt-oss-20b는 추론형 모델이라 측정이 일부 맞지 않습니다.** 답하기 전의 생각 과정을 별도 필드로 내보내는데, GuideLLM이 이를 본문 토큰으로 세지 않습니다. 그래서 토큰 간 지연이 0으로 기록되고, 첫 토큰 시간도 실행마다 다르게(19 ms, 32 ms, 콘솔 목록 선택 실행에서는 0) 나옵니다. **초당 토큰 수(154)는 세 번 모두 같은 값**이 나왔습니다.
- **정확도 묶음은 데모용 표본입니다.** 영어 공개 벤치마크 3종을 100문항씩 돌린 것이고, 기준값은 서비스팀이 정할 값의 자리표시자입니다. gpt-oss-20b는 세 항목 모두 기준을 0.01~0.03 차이로 넘겼습니다. 모델의 품질 순위로 읽으면 안 됩니다.
- **두 모델 모두 통과했습니다.** "validated 모델 중 하나만 통과"하는 그림은 아닙니다. 하나만 통과로 보이게 하려면 기준을 서비스 요구에 맞게 올려야 하는데(예: 100 tok/s), 그 기준에는 근거가 있어야 합니다.

## 레지스트리 등록

통과한 두 모델을 모델 레지스트리(`rhsummit-registry`)에 등록했습니다.

| 등록 이름 | 버전 | 기록한 내용 |
|---|---|---|
| `gpt-oss-20b` | v1 | 출처, 모델 이미지 주소, 카탈로그 최소 VRAM, 측정 가속기, 실측 속도와 첫 토큰, 정확도 점수, 판정 기준, 상태 |
| `qwen3-8b-fp8` | v1 | 같음 |

기존 `qwen-demo`(커뮤니티 원본 실패, FP8 미달, 32B-AWQ 채택 기록)는 그대로 두었습니다.

## 평가 전용 파이프라인

기존 검증 파이프라인은 모델을 내리고 올리는 과정이 있어 30~50분이 걸립니다. 데모에서 바로 돌릴 수 있도록, **이미 떠 있는 모델을 평가만 하는** 파이프라인을 추가했습니다.

```
check-model → performance-suite → accuracy-suite → verdict
```

| 단계 | 하는 일 |
|---|---|
| check-model | 모델에 질문 1개를 보내 응답 확인 |
| performance-suite | 성능 평가 묶음 실행 (GuideLLM, 30 tok/s 기준) |
| accuracy-suite | 정확도 평가 묶음 실행 (3종, 각 100문항) |
| verdict | 속도, 첫 토큰 시간, 정확도를 합쳐 통과 여부 판정 |

- 콘솔 위치: **Develop & train → Pipelines → Pipeline definitions → `model-evaluation`**
- 걸리는 시간: 약 5분
- 2026-10-05에 두 모델로 끝까지 실행해 성공했습니다.
  - `PASSED gpt-oss-20b: 154.02 tok/s, TTFT 19.3 ms, accuracy suite score 0.550`
  - `PASSED qwen3-8b-fp8: 66.86 tok/s, TTFT 26.2 ms, accuracy suite score 0.650`
- 성능과 정확도를 차례로 돌립니다. GPU가 1장이라 동시에 돌리면 성능 수치가 흔들리기 때문입니다.

두 묶음의 기준은 EvalHub에 들어 있고, 파이프라인은 통과 여부를 읽어 옵니다. 콘솔의 Benchmark suite 화면에서 같은 묶음을 직접 실행할 수도 있습니다([17번 문서](17-콘솔에서-EvalHub-평가-실행.md)).

## 지금 클러스터 상태

| 모델 | 상태 | 비고 |
|---|---|---|
| `gpt-oss-20b` | **서빙 중** | GPU 사용 중. 클러스터 내부 주소만 있음 |
| `qwen3-8b-fp8` | 정지 | 정지 표시를 지우면 다시 뜸 |
| `qwen36-35b-a3b-community` | 정지 | 실패 기록용 |
| `qwen3-32b-awq` (이전 운영 모델) | 복제본 0으로 내림 | 외부 게이트웨이 주소는 남아 있으나 응답할 모델이 없음 |

GPU가 1장이라 한 번에 하나만 뜹니다. 바꾸는 방법은 이렇습니다.

```bash
# 지금 모델 정지
oc -n rhsummit annotate inferenceservice gpt-oss-20b serving.kserve.io/stop=true --overwrite
# 다른 모델 시작
oc -n rhsummit annotate inferenceservice qwen3-8b-fp8 serving.kserve.io/stop-
```

이전 운영 모델로 되돌리려면 validated 모델을 정지한 뒤 복제본을 1로 올립니다.

```bash
oc -n rhsummit patch llminferenceservice qwen3-32b-awq --type=merge -p '{"spec":{"replicas":1}}'
```

**주의: 외부 엔드포인트.** 새로 배포한 validated 모델은 클러스터 내부 주소만 있습니다. 게이트웨이를 통한 외부 엔드포인트(데모 3)는 지금 `qwen3-32b-awq`에만 연결돼 있고, 그 모델은 내려가 있습니다. 데모 3을 validated 모델로 하려면 LLMInferenceService로 다시 배포하는 작업이 남아 있습니다. 게이트웨이 인증은 그대로이며, 키 없이 호출하면 401로 거부되는 것을 확인했습니다.

## 고객이 많이 쓰는 계열 중 우리 GPU에 올릴 수 있는 것

DeepSeek, Qwen, gpt-oss, GLM 계열을 카탈로그 성능 데이터에서 뽑았습니다. 기준은 최소 VRAM 44GB 이하(L40S의 실제 가용 메모리)입니다.

| 모델 | 최소 VRAM | 측정 가속기 (1장) | 비고 |
|---|---|---|---|
| Qwen2.5-7B-Instruct-quantized.w4a16 | 6.4 GB | A100-40, L4 | |
| Qwen2.5-7B-Instruct-quantized.w8a8 | 10.1 GB | A100-40, L4 | |
| Qwen2.5-7B-Instruct-FP8-dynamic | 10.1 GB | A100-40, L4 | |
| **Qwen3-8B-FP8-dynamic** | 10.9 GB | A100-40, L4 | 이번에 배포·측정 |
| **gpt-oss-20b** | 15.9 GB | A100-40, L4 | 이번에 배포·측정 |
| gpt-oss-20b-essential | 15.9 GB | A100-40, L4 | |
| Qwen2.5-7B-Instruct (양자화 없음) | 17.6 GB | A100-40, L4 | |
| Qwen3.6-35B-A3B-NVFP4 | 28.9 GB | A100-80, H100, H200 | FP4 형식. L40S에서 제 성능이 나는지 확인 안 됨 |
| Qwen3.6-35B-A3B-FP8 | 43.1 GB | A100-80, H100, H200 | 지난번에 시험. 메모리가 빠듯해 9.87 tok/s (`--enforce-eager` 등 절약 설정) |
| Qwen3.5-35B-A3B-FP8-dynamic | 43.4 GB | A100-80, H100, H200 | 위와 같이 빠듯함 |

올릴 수 없는 것:

- **DeepSeek**: 카탈로그에는 DeepSeek-R1-0528(양자화 w4a16도 427 GB)과 DeepSeek-V4-Pro뿐입니다. GPU 여러 장이 필요합니다.
- **GLM**: GLM-5.2-FP8 하나이고 869 GB입니다. H200 8장 기준입니다.
- **gpt-oss-120b**: 75.1 GB.
- **Qwen 대형**: Qwen3-Next-80B(50.5 GB부터), Qwen3-Coder-Next 등은 48GB를 넘습니다.

## 재현 방법

비공개 저장소의 파일입니다.

| 파일 | 내용 |
|---|---|
| `manifests/84-isvc-gpt-oss-20b.yaml`, `85-isvc-qwen3-8b-fp8.yaml` | validated 모델 배포 |
| `scripts/21-evalhub-collections.sh` | 평가 묶음 두 개 생성 |
| `scripts/20-eval-pipeline.sh` | 평가 파이프라인 업로드와 실행 |
| `pipelines/evaluation_pipeline.py` | 평가 파이프라인 정의 |
| `scripts/19-evalhub-model-ca.sh` | 모델 인증서 시크릿 (콘솔 연결 확인용 필드 추가) |

## 관련 문서

- [17-콘솔에서-EvalHub-평가-실행.md](17-콘솔에서-EvalHub-평가-실행.md)
- [18-검증-파이프라인-단계-설명.md](18-검증-파이프라인-단계-설명.md)
- [15-모델-카탈로그-세-분류.md](15-모델-카탈로그-세-분류.md)
