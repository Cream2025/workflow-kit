# Workflow-Kit

Cream2025 조직용 GitHub Actions 재사용 워크플로우 및 액션 라이브러리. 멀티플랫폼 빌드, 클라우드 배포, 알림을 위한 CI/CD 템플릿 제공.

## Critical Rules

- 모든 워크플로우는 `workflow_call` 트리거만 사용 (이 저장소에서 직접 실행하지 않음)
- 시크릿/자격 증명을 YAML에 하드코딩하지 않음 — 반드시 GitHub Secrets로 전달
- 커밋 메시지 형식: `[예나] {설명}` (한국어)
- AWS 기본 리전: `ap-northeast-2` (서울)

## Architecture

```
.github/
├── actions/common/
│   ├── build/                    # 플랫폼별 빌드 액션
│   │   ├── maui/{aos,ios,win}/   # .NET MAUI 빌드
│   │   ├── unity/{addressable,server}/  # Unity 빌드
│   │   └── web/default/          # Docker 웹 빌드
│   ├── upload/
│   │   ├── aws-ecr/              # Docker 이미지 → ECR
│   │   └── aws-s3/               # 파일 → S3 (sync/delete 옵션)
│   ├── deploy/
│   │   └── unity-multiplay-hosting/aws-s3/  # UGS Multiplay 배포
│   └── notify-result/
│       ├── discord/              # Discord 웹훅 알림
│       └── slack/                # Slack 웹훅 알림
├── actions/utils/
│   └── increment-var-counter/    # GitHub 변수 카운터 증가
└── workflows/                    # 12개 재사용 워크플로우
```

## Workflows

| Workflow | Purpose | Runner |
|----------|---------|--------|
| `common-build-and-upload-unity-addressable-aws-s3` | Unity Addressable 번들 빌드 | self-hosted (Windows) |
| `common-build-and-upload-unity-client-aos-aws-s3` | Unity Android APK 빌드 | ubuntu-latest |
| `common-build-and-upload-unity-server-aws-s3` | Unity Linux 서버 빌드 | self-hosted (Windows) |
| `common-build-and-upload-web-default-aws-ecr` | 웹앱 Docker 이미지 빌드 | ubuntu-latest |
| `common-build-and-upload-maui-{aos,ios,win}-aws-s3` | MAUI 크로스플랫폼 빌드 | OS별 |
| `common-get-version-{default,label}` | 버전 해석 (태그/라벨 기반) | ubuntu-latest |
| `common-deploy-eks-deployment` | EKS 배포 (kubectl set-image) | ubuntu-latest |
| `common-deploy-unity-multiplay-hosting-aws-s3` | UGS Multiplay 배포 | ubuntu-latest |
| `common-create-git-tags` | Git 태그 생성/업데이트 | ubuntu-latest |

## Key Patterns

**Composite Actions**: 모든 액션은 `using: "composite"` — inputs/outputs로 체이닝.

**버전 관리**: `v{major}.{minor}.{patch}` 형식. live 브랜치는 patch=0, 그 외는 타임스탬프(`YYMMDDHHMMSS`) 또는 0.

**네이밍 규칙**: 워크플로우 `common-{동작}-{플랫폼}-{대상}.yml`, 액션은 디렉토리 구조로 계층 표현.

**조건부 커밋**: 변경 사항이 없으면 커밋 스킵 (Unity Addressable 빌드).

**S3 업로드**: `--delete` 옵션으로 sync 지원, 경로 커스터마이징 가능.

## Tech Stack

GitHub Actions, Bash, PowerShell, Unity, .NET MAUI, Docker, AWS (S3/ECR/EKS), UGS CLI, kubectl

## Modular Docs

See `.claude/skills/` for:
- `github-actions-templates/` - GitHub Actions 워크플로우 패턴 가이드
