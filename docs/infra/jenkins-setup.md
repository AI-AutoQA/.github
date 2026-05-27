# Jenkins CI/CD 설정 가이드

> **전제조건**: [infra/SETUP-GUIDE.md](../infra/SETUP-GUIDE.md)의 단계 ①~⑧이 완료되어 Jenkins 컨테이너가 실행 중이어야 합니다.

---

## 개요

AutoQA Jenkins는 EC2에서 Docker 컨테이너로 실행되며, 아래 3개 파이프라인을 관리합니다.

| 파이프라인 | 소스 경로 | 빌드 방식 | 배포 대상 |
|---|---|---|---|
| `autoqa-backend` | `backend/` | EC2 로컬 Docker 빌드 | EC2 docker compose |
| `autoqa-frontend` | `frontend/` | EC2 로컬 Docker 빌드 | EC2 docker compose |
| `autoqa-playwright` | `playwright/` | ECR 푸시 | ECS Fargate |
> **트리거 전략**: `develop` 브랜치에서 `release-1.0`, `release-1.1` 등 `release-*` 브랜치를
> 생성하면 Webhook이 발사되어 파이프라인이 실행됩니다.
Jenkins 접속 주소: `https://k14a101.p.ssafy.io:8443`

---

## 필요한 인증 정보

| 인증 정보 | 쓰는 곳 | 설정 위치 | 용도 |
|---|---|---|---|
| `ec2_jenkins_access_key_id` / `ec2_jenkins_secret_access_key` | AWS CLI profile `autoqa-jenkins` | 외부 EC2에서 `aws configure --profile autoqa-jenkins` | Playwright ECR push, ECS task definition/service 업데이트 |
| GitLab Project Access Token | Jenkins credential `gitlab-autoqa-creds` | Jenkins UI → Credentials | Jenkins가 GitLab repository clone |
| GitLab Webhook Secret Token | GitLab Webhook 설정 | Jenkins job의 Build Triggers에서 생성 후 GitLab에 입력 | GitLab push 이벤트를 Jenkins job으로 전달 |
| `JENKINS_ADMIN_ID` / `JENKINS_ADMIN_PASSWORD` | Jenkins 로그인 | 외부 EC2 `~/ec2-setup/.env` | Jenkins 관리자 접속 |

AWS 배포 권한은 Jenkins UI credential에 넣지 않습니다. Jenkins 컨테이너는 외부 EC2의 `~/.aws`를 읽기 전용으로 마운트하고, Jenkinsfile에서 `--profile autoqa-jenkins`를 사용합니다.

---

## 단계 ①: Jenkins 최초 접속 및 플러그인 설치

### 1-1. 접속 및 로그인

`https://k14a101.p.ssafy.io:8443` 에 접속합니다.

- 아이디: `.env`의 `JENKINS_ADMIN_ID` 값
- 비밀번호: `.env`의 `JENKINS_ADMIN_PASSWORD` 값

> Jenkins는 CasC(Configuration as Code)로 자동 구성됩니다. 초기 설정 마법사는 나타나지 않습니다.

### 1-2. 필수 플러그인 설치

**Jenkins 관리 → Plugin Manager → Available plugins** 에서 아래 플러그인을 검색하여 설치합니다.

| 플러그인 | 용도 |
|---|---|
| `GitLab` | GitLab Webhook 수신 및 파이프라인 트리거 |
| `GitLab API` | GitLab API 연동 (MR 상태 업데이트) |
| `Docker Pipeline` | Jenkinsfile에서 Docker 명령 실행 |
| `Pipeline` | Pipeline / Multibranch Pipeline 지원 (미설치 시 설치) |
| `Configuration as Code` | CasC YAML 재적용 기능 (casc.yaml 사용 시) |

**설치 후 Jenkins 재시작:**

```
Jenkins 관리 → 플러그인 관리 → 설치 완료 후 "Jenkins 재시작" 체크박스 선택
```

또는 EC2에서:
```bash
sudo docker compose -f /opt/autoqa/docker-compose.yml restart jenkins
```

---

## 단계 ②: GitLab Credential 등록

Jenkins가 GitLab 저장소를 clone하려면 GitLab Project Access Token이 필요합니다.

### 2-1. GitLab Project Access Token 발급

GitLab 저장소 → **Settings → Access Tokens → Add new token**

- Token name: `jenkins-autoqa`
- Expiration date: 프로젝트 종료일 이후로 설정
- Role: `Reporter` 이상
- Scopes: `read_repository` 체크

**발급 완료 후 Token 해시 문자열을 복사해 둡니다 (딱 한 번만 보임).**

### 2-2. Jenkins에 Credential 등록

**Jenkins 관리 → Manage Credentials → (global) → Add Credentials**

| 항목 | 값 |
|---|---|
| Kind | `Username with password` |
| Username | `oauth2` (고정값) |
| Password | 발급한 Token 해시값 |
| ID | `gitlab-autoqa-creds` |
| Description | `GitLab Project Access Token` |

**Save** 클릭.

---

## 단계 ③: 파이프라인 생성 (3개)

### 3-1. Backend 파이프라인

**Jenkins 대시보드 → 새로운 Item**

| 항목 | 값 |
|---|---|
| Item 이름 | `autoqa-backend` |
| 타입 | `Pipeline` |

**Pipeline 탭 설정:**

| 항목 | 값 |
|---|---|
| Definition | `Pipeline script from SCM` |
| SCM | `Git` |
| Repository URL | `https://lab.ssafy.com/<그룹>/<프로젝트>.git` |
| Credentials | `gitlab-autoqa-creds` |
| Branch Specifier | `*/release-*` |
| Script Path | `backend/Jenkinsfile` |

**저장.**

---

### 3-2. Frontend 파이프라인

**Jenkins 대시보드 → 새로운 Item**

| 항목 | 값 |
|---|---|
| Item 이름 | `autoqa-frontend` |
| 타입 | `Pipeline` |

**Pipeline 탭 설정:**

| 항목 | 값 |
|---|---|
| Definition | `Pipeline script from SCM` |
| SCM | `Git` |
| Repository URL | `https://lab.ssafy.com/<그룹>/<프로젝트>.git` |
| Credentials | `gitlab-autoqa-creds` |
| Branch Specifier | `*/release-*` |
| Script Path | `frontend/Jenkinsfile` |

**저장.**

---

### 3-3. Playwright Worker 파이프라인

**Jenkins 대시보드 → 새로운 Item**

| 항목 | 값 |
|---|---|
| Item 이름 | `autoqa-playwright` |
| 타입 | `Pipeline` |

**Pipeline 탭 설정:**

| 항목 | 값 |
|---|---|
| Definition | `Pipeline script from SCM` |
| SCM | `Git` |
| Repository URL | `https://lab.ssafy.com/<그룹>/<프로젝트>.git` |
| Credentials | `gitlab-autoqa-creds` |
| Branch Specifier | `*/release-*` |
| Script Path | `playwright/Jenkinsfile` |

**저장.**

---

## 단계 ④: GitLab Webhook 연결

각 파이프라인이 GitLab push 이벤트를 받아 자동 실행되도록 Webhook을 등록합니다.

### 4-1. Jenkins에서 Webhook URL 확인

각 파이프라인 설정화면 열기 → **Build Triggers 탭**

**"Build when a change is pushed to GitLab"** 체크박스를 활성화하면 아래에 URL이 나옵니다:

```
https://k14a101.p.ssafy.io:8443/project/autoqa-backend
```

> Webhook은 HTTPS (8443 포트)로 수신합니다.

### 4-2. GitLab에서 Webhook 등록

GitLab 저장소 → **Settings → Webhooks → Add new webhook**

| 항목 | 값 |
|---|---|
| URL | `https://k14a101.p.ssafy.io:8443/project/autoqa-backend` |
| Secret token | *(Jenkins Build Triggers에서 Generate 후 복사)* |
| Trigger | `Push events` 체크 → **Branch filter**: `release-*` 입력 |
| SSL verification | 활성화 |

**Add webhook → Test → Push events** 로 동작 확인.

> 파이프라인 3개(backend, frontend, playwright) 모두 동일하게 등록합니다.
> URL 마지막 부분만 `/project/autoqa-backend` → `/project/autoqa-frontend`, `/project/autoqa-playwright` 로 변경.

---

## 단계 ⑤: 첫 번째 수동 빌드 실행

Webhook 등록 전에 수동으로 빌드를 실행하여 설정이 정상인지 확인합니다.

### 5-1. Backend / Frontend 빌드

```
Jenkins 대시보드 → autoqa-backend → 지금 빌드 (Build Now)
```

**정상 흐름:**
```
[Checkout]       → 소스 clone
[변경사항 감지]   → 첫 빌드이므로 backend/ 변경으로 처리
[Docker 빌드]    → autoqa/backend:latest 이미지 생성
[배포]           → docker compose up -d --no-deps --force-recreate backend
[이미지 정리]    → 이전 이미지 삭제
✅ Backend 배포 완료
```

**빌드 완료 후 컨테이너 상태 확인 (EC2에서):**
```bash
sudo docker compose -f /opt/autoqa/docker-compose.yml ps
```

---

### 5-2. Playwright 빌드

```
Jenkins 대시보드 → autoqa-playwright → 지금 빌드 (Build Now)
```

**정상 흐름:**
```
[Checkout]            → 소스 clone
[변경사항 감지]        → playwright/ 변경으로 처리
[ECR 로그인]          → AWS ECR 인증
[Docker 빌드]         → ECR 이미지 빌드
[ECR 푸시]            → 592931548131.dkr.ecr.ap-northeast-2.amazonaws.com/autoqa/playwright
[ECS Task Def 갱신]   → 새 이미지 태그로 Task Definition 재등록
[이미지 정리]         → 로컬 이미지 삭제
✅ Playwright Worker 배포 완료
```

> Playwright 이미지 빌드 후 ECS Worker의 desired count가 0이면 즉시 컨테이너가 시작되지 않습니다.  
> SQS에 잡이 들어올 때 CloudWatch Alarm → AppAutoScaling으로 자동 Scale Out됩니다.

---

## 파이프라인 동작 방식 요약

```
[GitLab Push]
    │
    └─► Jenkins Webhook 수신
            │
            ├─ backend/ 변경 감지 ─────────────────────────────────────────────
            │       EC2 로컬 빌드: docker build -t autoqa/backend:latest ./backend
            │       docker compose up -d --no-deps --force-recreate backend
            │       (ECR 미사용 — EC2에 이미지가 직접 존재)
            │
            ├─ frontend/ 변경 감지 ────────────────────────────────────────────
            │       EC2 로컬 빌드: docker build -t autoqa/frontend:latest ./frontend
            │       docker compose up -d --no-deps --force-recreate frontend
            │       (ECR 미사용 — EC2에 이미지가 직접 존재)
            │
            └─ playwright/ 변경 감지 ──────────────────────────────────────────
                    ECR 로그인 → Docker 빌드 → ECR 푸시
                    ECS Task Definition 재등록
                    aws ecs update-service → 다음 Scale Out 시 새 이미지 사용
```

---

## 자주 발생하는 문제 해결

### ❌ `docker: command not found` (Backend/Frontend 빌드 실패)

Jenkins 컨테이너가 Docker 소켓을 마운트했는지 확인합니다.
```bash
sudo docker inspect autoqa-jenkins | grep docker.sock
# /var/run/docker.sock 이 보여야 정상
```
`docker-compose.yml`에 소켓 마운트가 있고, `user: root`로 설정되어 있어야 합니다.

---

### ❌ ECR 로그인 실패 (`autoqa-playwright`)

```
Error: An error occurred (UnauthorizedException)
```

**원인**: EC2의 `autoqa-jenkins` AWS 프로파일이 없거나 키가 만료됨  
**해결**: EC2에서 확인
```bash
aws sts get-caller-identity --profile autoqa-jenkins
# 오류 시: aws configure --profile autoqa-jenkins 으로 재설정
```

---

### ❌ `git clone` 실패

```
Error cloning remote repo 'origin'
```

**원인**: GitLab Credential ID가 틀리거나 PAT 만료  
**해결**: Jenkins 관리 → Credentials → `gitlab-autoqa-creds` 수정 후 재발급된 PAT 입력

---

### ❌ Backend 컨테이너가 빌드 후 재시작됨

빌드는 성공했지만 컨테이너가 `Restarting` 상태인 경우:
```bash
sudo docker compose -f /opt/autoqa/docker-compose.yml logs backend --tail=50
```

주요 원인:
- `/opt/autoqa/.env`에 값이 비어있음 (DB_HOST, DB_PASSWORD 등)
- RDS 엔드포인트가 잘못됨 → `terraform output rds_endpoint` 재확인

---

### ❌ Webhook 수신 안 됨

GitLab Webhook Test 시 `Connection refused`:
- EC2 방화벽에서 8443 포트가 열려있는지 확인: `sudo ufw status`
- Jenkins 컨테이너가 실행 중인지 확인: `sudo docker compose ps`

---

## 참고: AWS 자격증명 갱신 방법

IAM Access Key를 재발급했거나 분실한 경우:

**1. Terraform으로 새 키 재발급 (infra/terraform/main 에서):**
```bash
# 기존 키 삭제 후 재생성은 Terraform이 자동 처리
terraform apply -target=aws_iam_access_key.ec2_jenkins
terraform output ec2_jenkins_access_key_id
terraform output -raw ec2_jenkins_secret_access_key
```

**2. EC2에서 프로파일 재설정:**
```bash
aws configure --profile autoqa-jenkins
```
