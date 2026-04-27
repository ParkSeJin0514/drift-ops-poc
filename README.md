# Drift Ops PoC

Skyline Airways 인프라의 Terraform Drift를 자동으로 감지하고, 위험도에 따라 Revert / Review / Accept 액션을 수행하는 GitOps 기반 운영 자동화 파이프라인입니다.

## 개요

클라우드 인프라를 Terraform(IaC)으로 관리하더라도, 콘솔 직접 변경이나 외부 요인으로 인해 코드와 실제 인프라 사이에 차이(Drift)가 발생할 수 있습니다. 이 프로젝트는 Drift 감지부터 대응까지의 전 과정을 GitHub Actions로 자동화합니다.

## 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│  GitHub Actions (Scheduled / Manual Trigger)                    │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌───────────────────┐  │
│  │  terraform   │───▶│   Triage     │───▶│  Action 분기      │  │
│  │  plan        │    │   Engine     │    │                   │  │
│  │  (drift 감지) │    │  (위험도 판정)│    │  Critical/IAM     │  │
│  └──────────────┘    └──────────────┘    │   → Revert        │  │
│                                          │  SG/RDS/Compute   │  │
│                                          │   → Review(Ticket)│  │
│                                          │  Tags only        │  │
│                                          │   → Accept(PR)    │  │
│                                          └───────────────────┘  │
│                             │                                   │
│              ┌──────────────┼──────────────┐                    │
│              ▼              ▼              ▼                    │
│        GitHub Issue    Slack 알림    terraform apply             │
│        (자동 생성)     (실시간)      (자동 Revert)               │
└─────────────────────────────────────────────────────────────────┘
```

## 주요 기능

### 1. Drift 감지 및 자동 분류

`terraform plan -detailed-exitcode`를 실행하여 IaC 코드와 실제 인프라 간의 차이를 감지합니다. 감지된 Drift는 Triage Engine을 통해 위험도와 권장 액션이 자동으로 분류됩니다.

| 위험도 | 조건 | 권장 액션 |
|--------|------|-----------|
| **Critical** | Security Group에 `0.0.0.0/0` 추가, IAM 변경 | Revert (자동 복원) |
| **High** | Security Group 규칙 변경, RDS/DB 변경 | Review (티켓 생성) |
| **Medium** | EC2, Launch Template 등 컴퓨팅 리소스 변경 | Review (티켓 생성) |
| **Low** | 태그(Tags) 변경만 감지 | Accept (PR 자동 생성) |

### 2. 액션별 대응 프로세스

**Revert** — Critical 위험도의 Drift가 감지되면 `production` 환경 승인 후 `terraform apply`로 IaC 상태로 즉시 복원합니다.

**Review** — GitHub Issue에 Ticket 라벨을 부여하고, 담당자가 원인을 분석한 뒤 Revert 또는 Accept를 수동으로 결정합니다.

**Accept** — 태그 등 운영 영향이 없는 변경은 자동으로 PR을 생성하여 Terraform 코드에 현재 인프라 상태를 반영합니다.

### 3. GitHub Issue 자동 생성

Drift가 감지될 때마다 아래 정보를 포함한 Issue가 자동으로 생성됩니다.

- 변경된 리소스, 위험도, 권장 액션, 판정 사유
- Terraform Plan 요약
- Workflow 실행 링크
- 후속 조치 체크리스트

라벨은 `Drift`, 위험도(`Critical`/`High`/`Medium`/`Low`), 액션(`Revert`/`Review`/`Accept`)이 자동 부여됩니다.

### 4. Slack 알림

모든 Drift 이벤트와 처리 결과가 Slack으로 실시간 통보됩니다.

| 상황 | 알림 내용 |
|------|-----------|
| Drift 감지 | 리소스, 위험도, 사유, 변경 상세, 대응 매뉴얼 링크 |
| No Drift | 인프라 정상 상태 확인 |
| Revert 완료/실패 | 복원 결과 및 Build 링크 |
| Accept PR 생성 | PR 링크 |
| Monthly Report | 월간 통계 요약 |

### 5. 월간 리포트

매월 자동으로 해당 월의 Drift 통계를 집계하여 리포트를 생성합니다.

- 위험도별 건수 및 비율
- 처리 현황 (Revert / Review / Accept)
- 미결/완료 건수
- 주요 Drift 리소스 TOP 5
- 전체 Drift 목록

리포트는 `reports/` 디렉토리에 마크다운으로 저장되고, S3에도 업로드되며, Slack으로 요약이 발행됩니다.

## 워크플로우 구성

```
.github/workflows/
├── drift-revert.yaml           # Drift 감지 → Triage → Revert/Review/Accept
└── drift-monthly-report.yaml   # 월간 리포트 생성 및 발행
```

### drift-revert.yaml

| Job | 실행 조건 | 수행 내용 |
|-----|-----------|-----------|
| `drift-detect` | 항상 실행 | Plan 실행, Triage 판정, Issue 생성, Slack 알림 |
| `revert` | Action = Revert | `production` 환경 승인 후 `terraform apply` |
| `accept` | Action = Accept | 브랜치 생성 → Accept 기록 작성 → PR 자동 생성 |

### drift-monthly-report.yaml

| 단계 | 수행 내용 |
|------|-----------|
| 리포트 생성 | 해당 월의 Drift Issue를 집계하여 마크다운 리포트 작성 |
| GitHub 저장 | `reports/` 디렉토리에 커밋 및 푸시 |
| S3 업로드 | `s3://skyline-terraform/drift-reports/`에 백업 |
| Slack 발행 | 요약 통계와 리포트 링크를 Slack에 전송 |

## 필수 설정

### GitHub Secrets

| Secret | 설명 |
|--------|------|
| `AWS_ACCESS_KEY_ID` | AWS 인증 Access Key |
| `AWS_SECRET_ACCESS_KEY` | AWS 인증 Secret Key |
| `TERRAFORM_TFVARS` | Terraform 변수 파일 내용 (JSON 문자열) |
| `SLACK_WEBHOOK_URL` | Slack Incoming Webhook URL |
| `PAT_TOKEN` | Accept PR 생성용 GitHub Personal Access Token |

### GitHub Environment

`revert` Job은 `production` 환경에 바인딩되어 있어, 승인자(Reviewer)가 승인해야만 `terraform apply`가 실행됩니다.

### 대상 레포지토리

Drift 감지 대상 Terraform 코드는 별도 레포지토리에서 관리됩니다.

- **인프라 코드:** [skyline-infra-terraform](https://github.com/ParkSeJin0514/skyline-infra-terraform)
- **리전:** `ap-northeast-2` (서울)
- **Terraform 버전:** `1.14.7`

## 디렉토리 구조

```
drift-ops-poc/
├── .github/workflows/
│   ├── drift-revert.yaml
│   └── drift-monthly-report.yaml
├── reports/
│   └── drift-report-YYYY-MM.md
└── README.md
```

## 운영 문서

- [대응 매뉴얼](https://docs.google.com/document/d/14fw3PLb-42_FlzbPCEpnzBRDSIpfootR/edit) — Drift 유형별 대응 절차
- [운영 Runbook](https://docs.google.com/document/d/1qYRbQa0Chpt7JUnkkjJr6IE3tst14U3y/edit) — 파이프라인 운영 가이드

## 월간 리포트 예시 (2026년 3월)

| 항목 | 건수 |
|------|------|
| 총 Drift | 43건 |
| Critical | 7건 (16%) |
| High | 31건 (72%) |
| Medium | 3건 (7%) |
| Low | 2건 (5%) |
| Reverted | 7건 |
| Reviewed | 34건 |
| Accepted | 2건 |

전체 리포트는 [`reports/drift-report-2026-03.md`](reports/drift-report-2026-03.md)에서 확인할 수 있습니다.
