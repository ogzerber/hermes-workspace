# Hermes Dashboard Ops Runbook

> 작성일: 2026-04-16
> 목적: `hermes update` 이후에도 dashboard enhanced APIs가 유지되도록 core/dashboard를 분리 운영하는 절차서

---

## 구조 요약

| 역할                     | 경로                                      | 포트 | launchd label                 |
| ------------------------ | ----------------------------------------- | ---- | ----------------------------- |
| Core Hermes (CLI/update) | `~/.hermes/hermes-agent`                  | —    | `ai.hermes.gateway` (비활성)  |
| Dashboard Gateway        | `/Users/teseuteu/hermes-agent-outsourc-e` | 8642 | `ai.hermes.dashboard-gateway` |
| Workspace UI             | `/Users/teseuteu/hermes-workspace`        | 3000 | `ai.hermes.workspace`         |
| Watchdog                 | `~/.hermes/scripts/hermes-watchdog.sh`    | —    | `ai.hermes.watchdog`          |

---

## Core Hermes 업데이트

```bash
hermes update
~/.hermes/bin/hermes-post-update-check
```

**영향 범위:** `~/.hermes/hermes-agent` (일반 CLI만).

**주의:**
- `hermes gateway start` 단독 실행은 dashboard 복구용으로 쓰지 않는다. generic active install 을 따라가기 때문.
- 업데이트 후에는 반드시 `~/.hermes/bin/hermes-post-update-check` 를 실행한다.
- 현재 `hermes-post-update-check` 는 generic plist만 최신 상태로 갱신한 뒤, `ai.hermes.gateway` 를 다시 disable/bootout 하고 `ai.hermes.dashboard-gateway` 를 올려서 split architecture를 복원한다.

---

## Dashboard Gateway 업데이트

```bash
git -C /Users/teseuteu/hermes-agent-outsourc-e pull --ff-only
launchctl kickstart -k gui/$(id -u)/ai.hermes.dashboard-gateway
sleep 5
curl --max-time 5 http://127.0.0.1:8642/health
```

---

## 상태 점검 명령

```bash
# 서비스 상태
launchctl list | grep 'ai\.hermes'

# 포트 점유
lsof -nP -iTCP -sTCP:LISTEN | grep -E ':(3000|8642)\b'

# health + enhanced endpoints
curl --max-time 5 http://127.0.0.1:8642/health
for p in /api/sessions /api/skills /api/memory /api/config; do
  code=$(curl --max-time 5 -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8642${p})
  echo "$p -> $code"
done

# workspace capability probe
curl --max-time 5 http://127.0.0.1:3000/api/gateway-status

# launchd disabled/enabled 상태 확인
launchctl print-disabled gui/$(id -u) | egrep 'ai\.hermes\.(gateway|dashboard-gateway)'
```

---

## Dashboard Gateway 재시작

```bash
launchctl kickstart -k gui/$(id -u)/ai.hermes.dashboard-gateway
```

필요 시 재확인:

```bash
lsof -nP -iTCP:8642 -sTCP:LISTEN
curl --max-time 5 http://127.0.0.1:8642/health
curl --max-time 5 http://127.0.0.1:3000/api/gateway-status
```

정상 기준:
- `8642` 소유자 = `ai.hermes.dashboard-gateway`
- `ai.hermes.gateway` = disabled
- workspace capability 에서 `sessions/skills/memory/config/jobs = true`

---

## Rollback (dashboard-gateway 장애 시)

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.dashboard-gateway.plist || true
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.hermes.gateway.plist
launchctl kickstart -k gui/$(id -u)/ai.hermes.gateway
# watchdog 원복
sed -i '' 's/ai.hermes.dashboard-gateway/ai.hermes.gateway/' ~/.hermes/scripts/hermes-watchdog.sh
launchctl kickstart -k gui/$(id -u)/ai.hermes.watchdog
```

**주의:** rollback 후에는 generic gateway가 다시 8642를 소유하므로 enhanced API가 다시 404가 됩니다. 서비스 중단 회피용 임시 조치입니다.

---

## 관련 파일

- Launch script: `~/.hermes/scripts/hermes-dashboard-gateway-launch.sh`
- Plist: `~/Library/LaunchAgents/ai.hermes.dashboard-gateway.plist`
- Watchdog: `~/.hermes/scripts/hermes-watchdog.sh`
- Post-update recovery: `~/.hermes/bin/hermes-post-update-check`
- Generic gateway plist: `~/Library/LaunchAgents/ai.hermes.gateway.plist` (파일은 남아 있어도 launchd disabled 유지)
- 백업: `~/.hermes/logs/ai.hermes.gateway.plist.bak.*`

---

## 재발 방지 규칙

1. `8642` 는 항상 `ai.hermes.dashboard-gateway` 가 소유해야 한다.
2. `ai.hermes.gateway` 는 파일이 남아 있어도 launchd 에서 disabled 상태를 유지한다.
3. 업데이트 후에는 반드시 아래 순서로 처리한다.

```bash
hermes update
~/.hermes/bin/hermes-post-update-check
```

4. watchdog 는 `ai.hermes.dashboard-gateway` 기준으로 health 를 복구하며, generic `ai.hermes.gateway` 가 다시 enabled/loaded 되면 자동으로 disable/bootout 한다.
5. `hermes gateway status` 는 generic gateway 기준 출력이라 split architecture 상태 확인의 단일 source of truth로 쓰지 않는다. 실제 확인은 아래 3가지를 함께 본다.

```bash
lsof -nP -iTCP:8642 -sTCP:LISTEN
launchctl print-disabled gui/$(id -u) | egrep 'ai\.hermes\.(gateway|dashboard-gateway)'
curl --max-time 5 http://127.0.0.1:3000/api/gateway-status
```
