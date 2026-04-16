# Hermes Dashboard Gateway 분리 운영 설계서

> **For Hermes:** 이 문서는 `hermes update` 이후 기존 `hermes-workspace`가 upstream `NousResearch/hermes-agent` 기준으로 다시 붙어 capability mismatch가 나는 문제를 막기 위한 운영 설계서입니다. 구현 시에는 core Hermes와 dashboard gateway를 분리하고, dashboard 쪽은 절대경로/별도 launchd service로 고정합니다.

**Goal:** `hermes update`가 실행되어도 기존 `hermes-workspace` 대시보드가 계속 enhanced API(`sessions/skills/memory/config`)를 유지하도록, dashboard가 generic active install을 따라가지 않는 구조로 바꿉니다.

**Architecture:** 일반 CLI/업데이트 흐름은 현재 `~/.hermes/hermes-agent` upstream install에 그대로 두고, `hermes-workspace`가 바라보는 gateway만 `/Users/teseuteu/hermes-agent-outsourc-e` checkout 기반의 별도 launchd service로 분리합니다. workspace는 계속 `3000`, dashboard gateway는 계속 `8642`를 사용하되, `8642`의 소유권을 generic `ai.hermes.gateway`가 아니라 dashboard 전용 service가 가지도록 고정합니다.

**Tech Stack:** macOS `launchd`, Hermes gateway, `outsourc-e/hermes-agent`, `outsourc-e/hermes-workspace`, shell scripts, localhost health checks

---

## 1. 현재 문제 요약

현재 확인된 실제 상태:
- 일반 Hermes source: `/Users/teseuteu/.hermes/hermes-agent`
- 일반 Hermes CLI symlink: `/Users/teseuteu/.local/bin/hermes -> /Users/teseuteu/.hermes/hermes-agent/venv/bin/hermes`
- 일반 gateway launchd: `~/Library/LaunchAgents/ai.hermes.gateway.plist`
- workspace launchd: `~/Library/LaunchAgents/ai.hermes.workspace.plist`
- watchdog script: `~/.hermes/scripts/hermes-watchdog.sh`
- workspace launch script: `~/.hermes/scripts/hermes-workspace-launch.sh`
- workspace UI port: `3000`
- gateway port: `8642`
- enhanced fork checkout already exists: `/Users/teseuteu/hermes-agent-outsourc-e`

문제의 본질:
1. `hermes update`는 `~/.hermes/hermes-agent`를 갱신합니다.
2. `~/.local/bin/hermes`와 `ai.hermes.gateway`는 이 active install을 따라갑니다.
3. 그 결과 launchd gateway가 upstream `NousResearch/hermes-agent` 기준으로 뜹니다.
4. `hermes-workspace`는 여전히 enhanced APIs를 기대합니다.
5. `/api/sessions`, `/api/skills`, `/api/memory`, `/api/config`가 사라져 dashboard가 degraded 됩니다.

즉, 재발 원인은 `workspace` 자체가 아니라 **dashboard가 generic active Hermes install을 따라가는 구조**입니다.

---

## 2. 목표 상태

분리 후 최종 상태는 아래와 같습니다.

### Core Hermes
- 목적: 일반 CLI, 터미널 작업, `hermes update`, 일반 운영
- repo: `/Users/teseuteu/.hermes/hermes-agent`
- remote: `https://github.com/NousResearch/hermes-agent.git`
- 서비스: 필요 시 기존 `ai.hermes.gateway`는 유지하되, dashboard 용도로는 사용하지 않음

### Dashboard Hermes Gateway
- 목적: `hermes-workspace`가 붙는 enhanced API 제공
- repo: `/Users/teseuteu/hermes-agent-outsourc-e`
- remote: `https://github.com/outsourc-e/hermes-agent.git`
- 서비스: 새 별도 launchd label 사용
- 권장 label: `ai.hermes.dashboard-gateway`
- 포트: `8642`

### Hermes Workspace
- repo: `/Users/teseuteu/hermes-workspace`
- 서비스: 기존 `ai.hermes.workspace` 유지
- 포트: `3000`
- backend target: `http://127.0.0.1:8642` (dashboard 전용 gateway)

### Watchdog
- gateway health check 대상: `http://127.0.0.1:8642/health`
- workspace health check 대상: `http://127.0.0.1:3000/`
- kickstart 대상 gateway label: `ai.hermes.dashboard-gateway`
- kickstart 대상 workspace label: `ai.hermes.workspace`

---

## 3. 설계 원칙

### 원칙 1: dashboard는 `~/.local/bin/hermes`를 사용하지 않는다
금지:
- `~/.local/bin/hermes gateway run`
- `hermes gateway start`
- current active install을 추적하는 generic wrapper

이유:
- update 후 upstream baseline drift를 그대로 따라가기 때문입니다.

### 원칙 2: dashboard gateway는 절대경로로 직접 실행한다
반드시 아래 2개를 고정합니다.
- Python: `/Users/teseuteu/hermes-agent-outsourc-e/venv/bin/python`
- WorkingDirectory: `/Users/teseuteu/hermes-agent-outsourc-e`

### 원칙 3: workspace는 8642만 보되, 그 8642의 소유권을 dashboard 전용 서비스가 가진다
즉 `~/.hermes/scripts/hermes-workspace-launch.sh`는 계속:

```bash
export HERMES_API_URL="http://127.0.0.1:8642"
```

를 유지하되, 8642가 generic `ai.hermes.gateway`가 아니라 새 `ai.hermes.dashboard-gateway`의 포트여야 합니다.

### 원칙 4: core update와 dashboard update를 분리한다
- core update: `hermes update`
- dashboard update: `git -C /Users/teseuteu/hermes-agent-outsourc-e pull --ff-only`

---

## 4. 필요한 파일 구조

### 새로 만들 파일
1. `~/Library/LaunchAgents/ai.hermes.dashboard-gateway.plist`
2. `~/.hermes/scripts/hermes-dashboard-gateway-launch.sh`
3. 필요 시 `~/.hermes/scripts/hermes-dashboard-gateway-update.sh`

### 수정 파일
1. `~/.hermes/scripts/hermes-watchdog.sh`
   - generic gateway label → dashboard gateway label로 변경
2. 필요 시 `~/Library/LaunchAgents/ai.hermes.watchdog.plist`
   - label은 유지 가능, script target만 바꾸면 됨
3. `~/.hermes/scripts/hermes-workspace-launch.sh`
   - 보통 `HERMES_API_URL=http://127.0.0.1:8642` 유지
   - 특별한 변경이 필요 없을 가능성이 높음

### 유지 파일
1. `~/Library/LaunchAgents/ai.hermes.workspace.plist`
2. `/Users/teseuteu/hermes-workspace/.env`
3. `/Users/teseuteu/hermes-workspace` 전체 repo

---

## 5. 권장 구현 순서

### Task 1: enhanced fork 실행 준비 상태 확인

**Objective:** `/Users/teseuteu/hermes-agent-outsourc-e`가 dashboard gateway로 실행 가능한지 검증합니다.

**Files:**
- Inspect: `/Users/teseuteu/hermes-agent-outsourc-e`
- Inspect: `/Users/teseuteu/hermes-agent-outsourc-e/venv`

**Steps**
1. remote/branch 확인
2. venv 존재 여부 확인
3. `python -m hermes_cli.main gateway run --replace`가 가능한지 확인
4. `/health` 뿐 아니라 `/api/sessions` 같은 enhanced endpoint 존재 여부 확인

**Commands**
```bash
git -C /Users/teseuteu/hermes-agent-outsourc-e remote -v
git -C /Users/teseuteu/hermes-agent-outsourc-e branch --show-current
/Users/teseuteu/hermes-agent-outsourc-e/venv/bin/python -m hermes_cli.main --version
```

**Verification**
다음 조건이 만족되어야 합니다.
- checkout remote가 `outsourc-e/hermes-agent`
- venv python 존재
- `/api/sessions` 계열 구현 코드가 실제로 존재하거나, 수동 실행 시 endpoint 응답이 나옴

---

### Task 2: dashboard gateway launch script 작성

**Objective:** dashboard gateway를 enhanced fork 절대경로로 고정하는 launch script를 만듭니다.

**Files:**
- Create: `~/.hermes/scripts/hermes-dashboard-gateway-launch.sh`

**Script shape**
```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="/Users/teseuteu/hermes-agent-outsourc-e"
cd "$ROOT"

export HERMES_HOME="/Users/teseuteu/.hermes"
export PATH="$ROOT/venv/bin:/Users/teseuteu/.local/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"
export VIRTUAL_ENV="$ROOT/venv"

exec "$ROOT/venv/bin/python" -m hermes_cli.main gateway run --replace
```

**Verification**
```bash
bash -n ~/.hermes/scripts/hermes-dashboard-gateway-launch.sh
chmod 755 ~/.hermes/scripts/hermes-dashboard-gateway-launch.sh
```
Expected: syntax OK

---

### Task 3: dashboard-gateway launchd plist 작성

**Objective:** generic `ai.hermes.gateway`와 별도로 dashboard 전용 gateway service를 추가합니다.

**Files:**
- Create: `~/Library/LaunchAgents/ai.hermes.dashboard-gateway.plist`

**Plist shape**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>ai.hermes.dashboard-gateway</string>

  <key>ProgramArguments</key>
  <array>
    <string>/bin/bash</string>
    <string>/Users/teseuteu/.hermes/scripts/hermes-dashboard-gateway-launch.sh</string>
  </array>

  <key>WorkingDirectory</key>
  <string>/Users/teseuteu/hermes-agent-outsourc-e</string>

  <key>EnvironmentVariables</key>
  <dict>
    <key>HERMES_HOME</key>
    <string>/Users/teseuteu/.hermes</string>
    <key>PATH</key>
    <string>/Users/teseuteu/hermes-agent-outsourc-e/venv/bin:/Users/teseuteu/.local/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    <key>VIRTUAL_ENV</key>
    <string>/Users/teseuteu/hermes-agent-outsourc-e/venv</string>
  </dict>

  <key>RunAtLoad</key>
  <true/>

  <key>KeepAlive</key>
  <dict>
    <key>SuccessfulExit</key>
    <false/>
  </dict>

  <key>StandardOutPath</key>
  <string>/Users/teseuteu/.hermes/logs/dashboard-gateway.log</string>

  <key>StandardErrorPath</key>
  <string>/Users/teseuteu/.hermes/logs/dashboard-gateway.error.log</string>
</dict>
</plist>
```

**Verification**
```bash
plutil -lint ~/Library/LaunchAgents/ai.hermes.dashboard-gateway.plist
```
Expected: OK

---

### Task 4: generic gateway와 dashboard gateway의 포트 소유권 정리

**Objective:** 8642를 dashboard gateway가 소유하도록 전환합니다.

**Files:**
- Modify operational state only

**Steps**
1. 현재 `ai.hermes.gateway`가 8642를 잡고 있는지 확인
2. dashboard-gateway를 내리지 않고 바로 올리면 포트 충돌이 나므로, cutover 창을 짧게 잡아 순서대로 전환
3. `ai.hermes.gateway`는 중지하거나, dashboard 용도가 아니라면 다른 포트/비활성 정책을 정함

**Commands**
```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.gateway.plist || true
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.dashboard-gateway.plist
launchctl kickstart -k gui/$(id -u)/ai.hermes.dashboard-gateway
sleep 3
curl --max-time 5 -i http://127.0.0.1:8642/health
```

**Verification**
아래 두 개를 모두 확인해야 합니다.
```bash
curl --max-time 5 -i http://127.0.0.1:8642/health
for p in /api/sessions /api/skills /api/memory /api/config; do curl --max-time 5 -i http://127.0.0.1:8642$p; done
```
Expected:
- `/health` = 200
- enhanced endpoints = 404가 아니어야 함

---

### Task 5: watchdog을 dashboard-gateway 기준으로 수정

**Objective:** watchdog이 더 이상 generic gateway를 재기동하지 않고, dashboard 전용 gateway를 감시하도록 바꿉니다.

**Files:**
- Modify: `~/.hermes/scripts/hermes-watchdog.sh`

**Current relevant lines**
```bash
GATEWAY_LABEL="ai.hermes.gateway"
```

**Replace with**
```bash
GATEWAY_LABEL="ai.hermes.dashboard-gateway"
```

Workspace check는 그대로 유지:
```bash
if check_http "http://127.0.0.1:3000/"; then
  log 'workspace healthy'
fi
```

**Reload commands**
```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.watchdog.plist || true
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.watchdog.plist
launchctl kickstart -k gui/$(id -u)/ai.hermes.watchdog
```

**Verification**
```bash
~/.hermes/scripts/hermes-watchdog.sh
tail -n 20 ~/.hermes/logs/watchdog.log
```
Expected:
- `dashboard-gateway` 기준으로 kickstart/logging
- workspace `3000` health 유지

---

### Task 6: workspace가 enhanced dashboard gateway에 계속 붙는지 검증

**Objective:** `hermes-workspace`는 계속 3000에서 떠 있으면서 backend만 새 dashboard-gateway를 바라보는지 확인합니다.

**Files:**
- Inspect: `~/.hermes/scripts/hermes-workspace-launch.sh`
- Inspect: `/Users/teseuteu/hermes-workspace/.env`

**Expected config**
```bash
export HERMES_API_URL="http://127.0.0.1:8642"
```

**Verification**
```bash
curl --max-time 5 -I http://127.0.0.1:3000/
curl --max-time 5 http://127.0.0.1:3000/api/gateway-status
```

Expected:
- UI 200
- gateway-status에서 `sessions`, `skills`, `memory`, `config`가 true 또는 적어도 enhanced availability를 반영

---

### Task 7: core update와 dashboard update runbook 분리

**Objective:** 앞으로 update할 때 dashboard가 다시 깨지지 않도록 운영 runbook을 분리 문서화합니다.

**Files:**
- Create or append: `/Users/teseuteu/hermes-workspace/docs/dashboard-ops-runbook.md`

**Runbook rules**
#### Core Hermes update
```bash
hermes update
```
영향 범위:
- 일반 CLI
- core install only

주의:
- dashboard recovery용으로 `hermes gateway start`를 습관적으로 쓰지 않음

#### Dashboard gateway update
```bash
git -C /Users/teseuteu/hermes-agent-outsourc-e pull --ff-only
launchctl kickstart -k gui/$(id -u)/ai.hermes.dashboard-gateway
curl --max-time 5 http://127.0.0.1:8642/health
```

**Verification**
문서에 아래 검사 순서를 포함합니다.
1. 8642 `/health`
2. enhanced endpoints
3. 3000 `/api/gateway-status`
4. Telegram 실제 송신 테스트(필요 시)

---

## 6. cutover 전 체크리스트

- [ ] `/Users/teseuteu/hermes-agent-outsourc-e` venv가 정상
- [ ] dashboard-gateway launch script 작성 완료
- [ ] dashboard-gateway plist 문법 검사 완료
- [ ] 8642 소유권 전환 절차 확인 완료
- [ ] `~/.hermes/scripts/hermes-watchdog.sh` 수정안 준비 완료
- [ ] `hermes-workspace`는 계속 `3000` 유지
- [ ] 기존 `ai.hermes.gateway`를 dashboard 용도에서 분리할 정책 결정 완료
- [ ] rollback용 현재 plist/script 백업 완료

---

## 7. rollback 계획

### Rollback 조건
- dashboard-gateway가 8642에서 healthy하지 않음
- enhanced endpoints가 여전히 404
- workspace가 3000에서 정상 표시되지 않음

### Rollback 절차
```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.dashboard-gateway.plist || true
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.gateway.plist
launchctl kickstart -k gui/$(id -u)/ai.hermes.gateway
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.watchdog.plist || true
# watchdog script 원복 후 다시 bootstrap
```

주의:
- rollback은 서비스 복구만 의미합니다.
- capability mismatch까지 해결되는 것은 아니므로, rollback 후에는 다시 기존 문제 상태로 돌아갈 수 있습니다.
- 따라서 rollback은 “서비스 중단 회피”용이고, 목표 상태는 dashboard-gateway 분리 운영입니다.

---

## 8. 핵심 결론

이 설계의 핵심은 단 하나입니다.

> `hermes-workspace`가 더 이상 `~/.local/bin/hermes`나 `~/.hermes/hermes-agent`의 active baseline을 따라가지 않게 만들고, `/Users/teseuteu/hermes-agent-outsourc-e`를 직접 실행하는 전용 gateway service에만 붙게 한다.

이 구조가 되면:
- `hermes update`를 해도 core만 바뀌고
- dashboard gateway는 그대로 유지되며
- 기존 대시보드가 매번 upstream baseline drift 때문에 초기화되는 문제를 크게 줄일 수 있습니다.

---

## 9. 구현 후 최종 검증 명령

```bash
launchctl list | egrep 'ai\.hermes\.(dashboard-gateway|workspace|watchdog)'
lsof -nP -iTCP -sTCP:LISTEN | egrep ':(3000|8642)\b'
curl --max-time 5 http://127.0.0.1:8642/health
for p in /api/sessions /api/skills /api/memory /api/config; do echo "== $p =="; curl --max-time 5 -i http://127.0.0.1:8642$p; done
curl --max-time 5 http://127.0.0.1:3000/api/gateway-status
```

Expected:
- `3000` = workspace
- `8642` = dashboard-gateway
- enhanced endpoints available
- workspace capability probe 정상
