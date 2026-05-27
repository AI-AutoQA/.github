# 왜 EBS Provisioned Rate for Volume Initialization + gp3 Throughput을 사용해야 하는가?

## 문제

AMI로 새 EC2를 시작하면 root EBS는 **비어 있는 상태로 먼저 생성**됩니다.
실제 데이터(모델 파일 등)는 **처음 읽을 때 그때그때 S3 snapshot에서 가져오는 lazy 방식**이기 때문에,
vLLM이 16GB 모델을 로드하는 동안 모든 블록을 S3에서 끌어와야 합니다.

→ **cold-start 46분 소요**

## 해결

| 설정 | 역할 |
|---|---|
| `volume_initialization_rate = 300` MiB/s | EC2 시작과 동시에 snapshot 전체를 EBS로 미리 초기화. 이후 모델 read는 로컬 디스크 속도로 처리 |
| `throughput = 300` MiB/s | gp3 기본값(125 MiB/s)보다 빠른 read 속도 보장 |

→ **cold-start 2분 30초 (18배 단축), 비용 EC2 1대당 약 $0.49**

## Fast Snapshot Restore 대신 쓰는 이유

AWS에는 같은 문제를 해결하는 **Fast Snapshot Restore(FSR)** 기능도 있습니다.

| 항목 | Fast Snapshot Restore | EBS Provisioned Rate for Volume Initialization |
|---|---|---|
| 과금 방식 | **상시 과금** (snapshot이 존재하는 동안 계속) | **사용 시점만 과금** (볼륨 초기화 중에만) |
| 비용 (서울 리전) | 약 $0.75/snapshot/시간 × 24h = **$18/일** | 120GB 기준 **EC2 1대당 약 $0.49** |
| 즉시성 | 볼륨 생성 직후부터 즉시 full speed | 초기화 완료까지 약 6~7분 소요 |
| 적합한 상황 | 하루 수십 회 이상 cold-start | 하루 몇 회 수준의 cold-start |

AutoQA GPU 서버는 **하루 몇 회 수준**으로만 시작되기 때문에 FSR을 상시 켜두는 것은 비용 낭비입니다.
Provisioned Rate for Volume Initialization이 훨씬 경제적입니다.

## Terraform 설정값

```hcl
# infra/terraform/main/terraform.tfvars
ai_gpu_root_volume_size                = 120  # GB
ai_gpu_root_volume_throughput          = 300  # MiB/s
ai_gpu_root_volume_initialization_rate = 300  # MiB/s
```

비용을 끄고 싶을 때:

```hcl
ai_gpu_root_volume_initialization_rate = null   # 초기화 비활성화
ai_prewarm_model_on_boot               = true   # EC2 시작 후 모델 미리 읽기 (fallback)
```

## 핵심 요약

- **원인**: AMI snapshot-backed EBS의 lazy initialization → vLLM이 모든 블록을 S3에서 가져와야 해서 46분 소요
- **해결**: `volume_initialization_rate=300`으로 볼륨 생성 직후 snapshot 전체 초기화 + `throughput=300`으로 read 속도 향상
- **결과**: 46분 → 2분 30초, EC2 1대당 $0.49의 추가 비용으로 18배 성능 개선
