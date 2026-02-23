# Slack Notification Action Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a Slack notification composite action that mirrors the existing Discord action's interface, using Slack Attachment formatting.

**Architecture:** A single composite action at `.github/actions/common/notify-result/slack/action.yml` with two steps — payload creation and webhook delivery. Same inputs as the Discord action (except webhook URL name).

**Tech Stack:** GitHub Actions composite action, Bash, curl, Slack Incoming Webhooks, Slack Attachment JSON format

---

## Reference

- **Discord action (mirror source):** `.github/actions/common/notify-result/discord/action.yml`
- **Slack Attachment docs:** Color bar + fields JSON posted to Incoming Webhook URL
- **Naming convention:** `common-{동작}-{플랫폼}-{대상}`, actions use directory hierarchy

---

### Task 1: Create the Slack notification composite action

**Files:**
- Create: `.github/actions/common/notify-result/slack/action.yml`

**Step 1: Create the action file**

Create `.github/actions/common/notify-result/slack/action.yml` with this exact content:

```yaml
name: Notify Result by Slack
description: workflow 수행 결과를 Slack 채널에 전송

inputs:
  failed_jobs:
    required: true
    description: 실패한 job 내역.(ex. job1,job2)
  title:
    required: true
    description: 완료/실패에 대한 제목
  slack_webhook_url:
    required: true
    description: Slack 웹훅 url
  version:
    required: false
    description: 배포한 버전
  upload_path:
    required: false
    description: 빌드 결과물 업로드 경로
  notify_success:
    required: false
    default: "true"
    description: 성공한 내역 알림 보낼지 여부

runs:
  using: "composite"
  steps:
    - name: Create Payload
      id: create-payload
      run: |
        github_url="https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}"
        payload=""

        if [[ "${{ inputs.failed_jobs }}" != "" ]]; then
          jobs="${{ inputs.failed_jobs }}"
          formatted_jobs=""
          IFS=',' read -ra JOB_ARRAY <<< "$jobs"
          for job in "${JOB_ARRAY[@]}"; do
            formatted_jobs+="  - ${job}\n"
          done

          text=":cry: *[배포 실패]*\n"
          text+="- 서비스: \`${{ inputs.title }}\`\n"
          text+="- 실패한 job:\n${formatted_jobs}"
          text+=":point_right: <${github_url}|workflow 확인>"

          payload=$(cat <<EOF
        {
          "attachments": [
            {
              "color": "#FF0000",
              "text": "${text}"
            }
          ]
        }
        EOF
          )
        elif [[ "${{ inputs.notify_success }}" == "true" ]]; then
          text=":hatching_chick: *[배포 성공]*\n"
          text+="- 서비스: \`${{ inputs.title }}\`\n"

          if [[ -n "${{ inputs.version }}" ]]; then
            text+="- 버전: \`${{ inputs.version }}\`\n"
          fi

          if [[ -n "${{ inputs.upload_path }}" ]]; then
            text+="- 업로드 경로: ${{ inputs.upload_path }}\n"
          fi

          payload=$(cat <<EOF
        {
          "attachments": [
            {
              "color": "#36a64f",
              "text": "${text}"
            }
          ]
        }
        EOF
          )
        fi

        echo "payload<<SLACKEOF" >> $GITHUB_OUTPUT
        echo -e "$payload" >> $GITHUB_OUTPUT
        echo "SLACKEOF" >> $GITHUB_OUTPUT
      shell: bash

    - name: Send Message
      if: ${{ steps.create-payload.outputs.payload != '' }}
      env:
        PAYLOAD: ${{ steps.create-payload.outputs.payload }}
        SLACK_WEBHOOK_URL: ${{ inputs.slack_webhook_url }}
      run: |
        curl -H "Content-Type: application/json" \
             -X POST \
             -d "$PAYLOAD" \
             "$SLACK_WEBHOOK_URL"
      shell: bash
```

**Key differences from Discord action:**
- Uses Slack emoji syntax (`:cry:`, `:hatching_chick:`, `:point_right:`) instead of Unicode emoji
- Uses Slack link syntax (`<url|text>`) instead of Markdown links
- Uses Slack mrkdwn bold (`*text*`) instead of Discord Markdown (`**text**`)
- Wraps message in `attachments` array with `color` field instead of `content` string
- Passes full JSON payload directly to curl instead of escaping a raw string

**Step 2: Validate YAML syntax**

Run: `python -c "import yaml; yaml.safe_load(open('.github/actions/common/notify-result/slack/action.yml'))"`
Expected: No output (valid YAML)

If python/yaml not available, run: `cat .github/actions/common/notify-result/slack/action.yml | head -5`
Expected: Shows `name: Notify Result by Slack`

**Step 3: Commit**

```bash
git add .github/actions/common/notify-result/slack/action.yml
git commit -m "[예나] Slack 알림 composite action 추가"
```

---

### Task 2: Update CLAUDE.md with Slack action reference

**Files:**
- Modify: `CLAUDE.md`

**Step 1: Add slack to the notify-result line in Architecture section**

In the Architecture tree, update the notify line from:
```
│   └── notify-result/discord/    # Discord 웹훅 알림
```
to:
```
│   └── notify-result/
│       ├── discord/              # Discord 웹훅 알림
│       └── slack/                # Slack 웹훅 알림
```

**Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "[예나] CLAUDE.md에 Slack 알림 액션 추가"
```
