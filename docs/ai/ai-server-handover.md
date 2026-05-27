# AI 서버 개발팀 전달사항

> 기준일: 2026-05-04  
> 대상: AI 서버 개발팀 / Python, ML, Docker, vLLM 담당  
> 목적: Naver MVP 검증 중 확인된 AI GPU Worker 차단 요소와 다음 작업 기준 정리

---

## 1. MVP 검증 결과 요약

검증 대상 URL은 `https://www.naver.com/`이고, 직접 SQS 메시지를 넣어 단계별로 확인했습니다.

| 항목 | 결과 | 비고 |
|---|---|---|
| Web 분석 | 성공 | Playwright Worker가 Naver를 분석하고 S3 산출물을 업로드함 |
| AI 시나리오 생성 | 미완료 | GPU/ECS는 구동했지만 현재 AI 이미지의 vLLM/transformers 조합에서 모델 로드 실패 |
| QA 실행 | 성공 | AI 생성 대신 fallback suite를 S3에 올려 QA 실행 경로를 검증함 |
| 정리 상태 | 완료 | 테스트 SQS 메시지 purge, ECS 서비스 desired 0, GPU ASG desired 0 |

MVP job id:

```text
mvp-naver-20260504-140319
```

확인된 S3 산출물:

```text
s3://autoqa-artifacts-prod/reports/mvp-naver-20260504-140319/analysis/analysis-summary.json
s3://autoqa-artifacts-prod/reports/mvp-naver-20260504-140319/analysis/final-report.json
s3://autoqa-artifacts-prod/reports/mvp-naver-20260504-140319/analysis/site-summary.json
s3://autoqa-artifacts-prod/reports/mvp-naver-20260504-140319/analysis/pages/p001_www_naver_com_/baseline-annotated.png
s3://autoqa-artifacts-prod/jobs/mvp-naver-20260504-140319/result/scenarios.json
s3://autoqa-artifacts-prod/reports/mvp-naver-20260504-140319/execute/suite-report.json
```

`suite-report.json` 기준 QA 실행은 `passed`, 총 1개 시나리오 / 2개 step 통과입니다. 단, `jobs/.../result/scenarios.json`은 AI가 생성한 결과가 아니라 다운스트림 검증을 위한 수동 fallback suite입니다.

---

## 2. 현재 운영 인프라 계약

AI Worker는 ECS on EC2 구조로 실행됩니다. 모델 파일은 ECR 이미지에 넣지 않고, GPU EC2의 로컬 모델 캐시를 컨테이너 `/models`로 읽기 전용 mount합니다.

| 항목 | 값 |
|---|---|
| AWS profile | `autoqa-main` |
| Account | `592931548131` |
| Region | `ap-northeast-2` |
| ECS cluster | `autoqa-gpu-cluster` |
| ECS service | `autoqa-gemma-gpu-worker` |
| Capacity provider | `autoqa-g6e-capacity-provider` |
| ASG | `autoqa-ai-gpu-asg` |
| Instance type | `g6e.xlarge` |
| Baked AMI | `ami-06629b46a47d70fd6` |
| ECR image | `592931548131.dkr.ecr.ap-northeast-2.amazonaws.com/autoqa/ai:latest` |
| Worker command | `python -m app.worker` |
| Host model cache | `/mnt/nvme/models` |
| Container model cache | `/models` |

현재 AMI 모델 경로:

```text
/models/gemma-base
/models/lora
/models/no-lora
/models/qwen3-8b-fp8
/models/qwen3-8b-bf16
```

Qwen3-8B-BF16을 기본 모델로 사용할 때 Terraform 변수는 아래 기준입니다.

```hcl
ai_instance_type   = "g6e.xlarge"
ai_base_model_id   = "Qwen/Qwen3-8B"
ai_base_model_path = "/models/qwen3-8b-bf16"
ai_lora_path       = "/models/no-lora"
ai_use_vllm_lora   = false
ai_gpu_ami_id      = "ami-06629b46a47d70fd6"
ai_gpu_root_volume_initialization_rate = 300
ai_gpu_root_volume_throughput          = 300
```

Qwen BF16 cold-start에서는 AMI에 모델 파일이 있어도 새 EC2 root EBS가 snapshot-backed volume으로 생성되며 첫 read가 느릴 수 있습니다. 상시 과금되는 Fast Snapshot Restore 대신 기본 운영값은 EBS Provisioned Rate for Volume Initialization 300 MiB/s입니다. 서울 리전 기준 현재 120GB snapshot 전체를 보수적으로 잡아도 새 EC2 1대당 약 `$0.49` 수준이고, 비용이 싫으면 `ai_gpu_root_volume_initialization_rate = null`로 끈 뒤 `ai_prewarm_model_on_boot = true`를 fallback으로 사용할 수 있습니다.

---

## 3. SQS/Worker 메시지 계약

AI Worker는 `SQS_WORK_QUEUE_URL`에서 long polling으로 메시지를 받고, 성공 시 `SQS_RESULT_QUEUE_URL`에 결과 메시지를 보낸 뒤 work message를 삭제합니다.

주요 큐:

```text
AI work queue          https://sqs.ap-northeast-2.amazonaws.com/592931548131/autoqa-ai-queue
Scenario result queue  https://sqs.ap-northeast-2.amazonaws.com/592931548131/autoqa-scenario-result-queue
```

work message body는 snake_case입니다.

```json
{
  "job_id": "mvp-naver-20260504-140319",
  "summary_s3_url": "s3://autoqa-artifacts-prod/reports/mvp-naver-20260504-140319/analysis/site-summary.json",
  "bref_s3_url": "s3://autoqa-artifacts-prod/reports/mvp-naver-20260504-140319/analysis/final-report.json",
  "callback": {
    "result_s3_prefix": "s3://autoqa-artifacts-prod/jobs/mvp-naver-20260504-140319/result/"
  },
  "options": {}
}
```

성공 시 Worker가 업로드해야 하는 파일:

```text
{callback.result_s3_prefix}/scenarios.json
```

S3 업로드 형식은 wrapper를 포함합니다.

```json
{
  "jobId": "mvp-naver-20260504-140319",
  "status": "completed",
  "ts": 1770000000,
  "result": {
    "suiteId": "...",
    "scenarios": []
  }
}
```

Worker는 Redis에도 `job_id` 기준 QA 시나리오 캐시를 저장합니다. S3 업로드가 성공하고 Redis 저장만 실패한 경우에는 S3 결과를 우선 정상 결과로 보고 계속 진행합니다.

---

## 4. 이번 검증에서 수정된 인프라 소스

아래 수정은 Java/backend 변경 없이 `infra` 쪽에만 반영했습니다.

| 파일 | 변경 내용 | 이유 |
|---|---|---|
| `infra/terraform/main/ai_ecs_userdata.sh.tpl` | `systemctl start --no-block ecs` 사용 | cloud-init `cloud-final`과 `ecs.service`의 동기 start deadlock 방지 |
| `infra/terraform/main/ai_ecs_gpu.tf` | container command를 `python -m app.worker`로 override | AI Dockerfile 기본 CMD가 FastAPI 서버라 SQS Worker가 실행되지 않던 문제 수정 |
| `infra/terraform/main/ai_ecs_gpu.tf` | `IDLE_TIMEOUT_SEC=300` | SQS 처리 후 5분 idle이면 Worker 종료 |
| `infra/terraform/main/ai_ecs_gpu.tf` | ECS service lifecycle에서 `task_definition` ignore 제거 | 새 task definition이 service에 반영되도록 함 |

주의: 소스 수정 후 실제 AWS 반영은 Terraform apply가 필요합니다. cleanup 과정에서는 비용 절감을 위해 ECS service와 ASG를 0으로 내렸습니다.

---

## 5. AI 이미지/모델 차단 요소

현재 ECR AI 이미지에서 관측한 주요 패키지 조합:

```text
vllm 0.7.3
transformers 5.7.0
```

Gemma4 로드 실패:

```text
ValueError: rope_scaling should have a 'rope_type' key
```

의미:

- 현재 vLLM이 Gemma4 config의 RoPE 설정 형식을 제대로 해석하지 못합니다.
- Gemma4 기준으로 계속 갈 경우 vLLM/transformers를 Gemma4 config와 호환되는 버전으로 명확히 pin해야 합니다.
- 임시로 config를 변환하는 방식보다, 먼저 vLLM 공식 지원 버전으로 올리는 쪽을 권장합니다.

Qwen3 계열 테스트에서 확인한 실패:

```text
Qwen2Tokenizer has no attribute all_special_tokens_extended
```

```text
ValueError: The checkpoint you are trying to load has model type `qwen3` but Transformers does not recognize this architecture.
```

의미:

- `vllm 0.7.3`과 현재 transformers 조합이 Qwen3 계열과 맞지 않습니다.
- transformers를 낮추면 tokenizer 문제는 바뀌지만 `qwen3` architecture를 모르는 문제가 발생했습니다.
- tokenizer monkey patch 후에는 GPU 메모리 할당까지 진행됐지만 safetensors shard 로딩에서 장시간 정체됐습니다.

AI팀 권장 작업:

1. 우선 MVP 목표 모델을 하나로 고정합니다. 현재 인프라 적용 기준은 Qwen3-8B-BF16입니다.
2. Qwen3-8B-BF16을 지원하는 vLLM/transformers/CUDA 조합을 새 Docker image에서 명시적으로 pin합니다.
3. 컨테이너 내부에서 `/models/qwen3-8b-bf16`, `/models/no-lora`를 mount한 상태로 `model_service.load_model()` 단독 smoke test를 먼저 통과시킵니다.
4. 그 다음 `python -m app.worker`로 SQS 없이 로컬 fixture를 처리하는 테스트를 추가합니다.
5. 마지막으로 실제 `autoqa-ai-queue`에 direct message를 넣어 `scenarios.json` 생성, result queue 발행, work message 삭제 순서를 검증합니다.

---

## 6. Cold-start/Autoscaling 차단 요소

MVP 중 SQS queue attribute에는 메시지가 보였지만 CloudWatch SQS metric/alarm 반영이 늦거나 0으로 조회되어 scale-out이 즉시 발생하지 않았습니다. 이후 purge 후에는 stale scale-out action으로 desired가 다시 1로 올라가는 현상도 있었습니다.

다음 작업이 필요합니다.

| 항목 | 권장 수정 |
|---|---|
| Playwright scale-out metric math | `m1 + m2` 대신 `FILL(m1, 0) + FILL(m2, 0)` 검토 |
| SQS scale-out statistic | 짧은 burst에는 `Sum`보다 `Maximum`이 안전한지 검증 |
| AI scale-out 조건 | visible message만 볼지, visible + not visible을 함께 볼지 결정 |
| Scale-in | AI는 5분 idle 정책과 CloudWatch scale-in이 같은 방향인지 검증 |
| Manual cleanup | ASG instance가 scale-in protection 상태면 `set-instance-protection --no-protected-from-scale-in` 필요 |

정리 시 최종 확인한 상태:

```text
autoqa-analyze-queue: 0 visible / 0 in-flight
autoqa-ai-queue: 0 visible / 0 in-flight
autoqa-execute-queue: 0 visible / 0 in-flight
autoqa-page-analysis-result-queue: 0 visible / 0 in-flight
autoqa-scenario-result-queue: 0 visible / 0 in-flight
autoqa-execute-result-queue: 0 visible / 0 in-flight
autoqa-result-queue: 0 visible / 0 in-flight

autoqa-worker ECS service: desired 0 / running 0 / pending 0
autoqa-gemma-gpu-worker ECS service: desired 0 / running 0 / pending 0
autoqa-ai-gpu-asg: desired 0, GPU instances terminating
scale-out alarms: OK, actions enabled
```

---

## 7. AI팀 재검증 절차

### 7-1. 이미지 빌드/push

```bash
AWS_ACCOUNT=592931548131
AWS_REGION=ap-northeast-2
ECR_REGISTRY=$AWS_ACCOUNT.dkr.ecr.$AWS_REGION.amazonaws.com
ECR_REPO=$ECR_REGISTRY/autoqa/ai

aws ecr get-login-password --profile autoqa-main --region $AWS_REGION \
  | docker login --username AWS --password-stdin $ECR_REGISTRY

docker build -t $ECR_REPO:latest -t $ECR_REPO:$(date +%Y%m%d-%H%M) ./ai
docker push $ECR_REPO:latest
docker push $ECR_REPO:$(date +%Y%m%d-%H%M)
```

### 7-2. Terraform 반영

```bash
cd infra/terraform/main
terraform fmt
terraform validate
terraform apply
```

### 7-3. Direct SQS 재검증

```bash
aws sqs send-message \
  --profile autoqa-main \
  --region ap-northeast-2 \
  --queue-url https://sqs.ap-northeast-2.amazonaws.com/592931548131/autoqa-ai-queue \
  --message-body file://ai-message.json
```

관측 포인트:

```bash
aws ecs describe-services \
  --profile autoqa-main \
  --region ap-northeast-2 \
  --cluster autoqa-gpu-cluster \
  --services autoqa-gemma-gpu-worker

aws logs tail /autoqa/ecs/ai-worker \
  --profile autoqa-main \
  --region ap-northeast-2 \
  --since 30m

aws s3 cp \
  s3://autoqa-artifacts-prod/jobs/{job_id}/result/scenarios.json - \
  --profile autoqa-main \
  --region ap-northeast-2
```

성공 기준:

| 확인 항목 | 기준 |
|---|---|
| Cold-start | SQS 메시지 입력 후 AI ECS desired/running이 1로 올라감 |
| GPU instance | `g6e.xlarge`가 ASG로 1대 기동 후 ECS에 등록됨 |
| Worker command | CloudWatch log에 `AutoQA AI Worker 시작` 출력 |
| Model load | `model_service.is_model_loaded() == True` |
| S3 result | `jobs/{job_id}/result/scenarios.json` 생성 |
| Result queue | scenario result queue에 success payload 발행 |
| Work queue | 성공 후 원본 AI work message 삭제 |
| Scale-down | 큐가 비고 idle 5분 후 ECS desired 0, ASG desired 0 |

---

## 8. 완료 정의

AI팀 작업은 아래가 모두 만족될 때 완료로 봅니다.

| 항목 | 기준 |
|---|---|
| 이미지 | `autoqa/ai:latest`에서 Gemma4 또는 선택 모델이 GPU로 로드됨 |
| 버전 | vLLM/transformers/CUDA 버전이 requirements 또는 Dockerfile에 pin됨 |
| Worker | `python -m app.worker`가 ECS에서 실행됨 |
| 추론 | 실제 AI 생성 suite가 QA contract를 통과함 |
| S3 | `scenarios.json`이 wrapper 형식으로 저장됨 |
| Redis | `job_id` 기준 QA 시나리오 캐시 저장 |
| SQS | 성공 시 result queue 발행 후 work message 삭제 |
| 비용 | 메시지가 없으면 ECS/ASG가 0으로 내려감 |
