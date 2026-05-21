안녕, 노바. 이전 Cowork 세션에서 이어받는 핸드오프야. 아래 내용을 읽고, 마지막 세션 시작 체크리스트 순서대로 바로 실행해줘. 확인 묻지 말고 진행해.

---

## 1. 이전 세션 완료 작업

### 1-1. dev-plan v2 → v3 업데이트 및 커밋

**왜**: Claude Code가 코드 리뷰에서 8개 지적사항을 발견. v2에서 수정, Claude Code가 추가로 v3 작성.

**v2 주요 수정 내용**:
- `webhook으로 Claude 세션 깨움` 아키텍처 오류 수정 — 웹훅은 Discord에 단방향 POST, Claude 세션 깨울 수 없음
- 멘션 감지 `includes()` → 단어 경계 정규식으로 교체 (오탐 방지)
- `pg_dump` 비밀번호 노출 방지: `~/.pgpass` 사용 필수 명시
- 1순위에 MCP_API_KEY 로테이션(1-D), /health 모니터링(1-E) 추가
- 인스턴스 명칭 규칙 `{이름}-{Claude이름}[-{용도}]` 통일

**v3 추가 내용 (Claude Code 작성)**:
- `1-F ⭐ 표준 시작 프롬프트 + CLAUDE.md 폴링 강제 룰` — "허브 등록" 한 마디로 register + wait_for_reply 진입. **응답 지연의 진짜 해결책**
- MCP_API_KEY 로테이션: 서버 `.env` 2곳 + 클라이언트 PC별 + Cowork URL 갱신 절차 상세화
- Bridge `/health` → 직접 엔드포인트 추가 대신 `/api/instances`의 `discord-bridge` `last_seen` 간접 체크로 변경 (코드 변경 없이 동작)
- 명칭 규칙 표 정정: `mark-nova-hq`, `mark-nova-cowork`, `casey-{claudename}-{용도}` 예시 추가

**커밋**:
- `kbc-devops/cross-claude-mcp` main: `46ff307` "docs: handoff-2026-04-09"
- `kbc-devops/cross-claude-mcp` reload-window: `021e77f` "docs: CLAUDE.md cross-claude rules + dev-plan v2"
- `kbc-devops/cross-claude-discord-bridge` main: `5d7ad59` "docs: cross-claude-dev-plan v2"

### 1-2. kbc-devops/cross-claude-mcp 레포 신규 생성

**왜**: 로컬 `C:\Users\heung\kbc-workspace\cross-claude-mcp`의 remote가 `rblank9/cross-claude-mcp`(외부 오픈소스)를 가리키고 있어 push 불가(403). 커스텀 작업을 kbc-devops 조직 레포로 관리해야 함.

**조치**:
1. GitHub API로 `kbc-devops/cross-claude-mcp` 레포 생성
2. 로컬 remote 변경: `git remote set-url origin https://github.com/kbc-devops/cross-claude-mcp.git`
3. `git push origin --all` → main + reload-window 브랜치 모두 push
4. CLAUDE.md + cross-claude-dev-plan.md + handoff 파일 커밋/push

**주의**: `.git/index.lock` 잔재 파일 가끔 생김 (VS Code가 잠금) → 필요 시 `del C:\Users\heung\kbc-workspace\cross-claude-mcp\.git\index.lock`

### 1-3. cross-claude 시스템 업그레이드 파악 및 Discord 공지

**Claude Code가 구현 완료한 것들** (toy01 `rblank9/cross-claude-mcp` 기반 서버 반영):

| 커밋 | 내용 |
|------|------|
| `7e0c5de` | fix: `/webhook/deploy`를 `mcpAuthRouter` 앞으로 이동 (express.json 충돌 해결) |
| `4b18878` | feat(3-A/3-B/3-C): 메시지 아카이브 + stale 인스턴스 30일 자동 삭제 + shared_data TTL |
| `e6cbe76` | fix(db): unread_mentions SCHEMA_SQL 누락 + SQLite webhook_url binding 오류 |
| `034bcab` | feat(2-A/2-C): unread_mentions 큐 + 단어 경계 멘션 감지 (escapeRegex 포함) |
| `415914c` | feat: register 도구에 webhook_url 파라미터 직접 등록 지원 |

bridge 레포 (`kbc-devops/cross-claude-discord-bridge`):

| 커밋 | 내용 |
|------|------|
| `3f7913b` | feat(2-D): webhook config 리로드 주기 10분 → 30초 |

공지: `#cross-ai-cowork` 채널 메시지 `#185`에 전송 완료

### 1-4. nova-msla-server 미응답 원인 분석

**증상**: #199 (상세 배포 보고) 이후 #202~#204 세 번 호출에 무응답  
**원인**: 긴 응답 생성 후 `wait_for_reply` 폴링 루프 재진입 실패 (컨텍스트 과부하 또는 루프 누락)  
**근본 해결**: 1-F (CLAUDE.md 폴링 강제 룰) 미구현 상태 — 아직 남은 작업

---

## 2. 현재 서버/레포 상태

### toy01 서버
| 프로세스 | PM2 ID | 포트 | 버전 | 상태 |
|----------|--------|------|------|------|
| cross-claude | 8 | 3500 | 2.0.0 | ✅ online |
| cross-claude-bridge | 7 | — | 1.0.0 | ✅ online |

- MCP 엔드포인트: `https://cross.marksun.me/mcp`
- 배포 웹훅: `POST https://cross.marksun.me/webhook/deploy` (HMAC-SHA256)
- DB: `postgresql://crossclaude:***@127.0.0.1:5432/crossclaude`
- 레포: `/home/mark/repos/cross-claude-mcp`, `/home/mark/repos/cross-claude-discord-bridge`

### GitHub 레포
| 레포 | 브랜치 | 최신 커밋 |
|------|--------|----------|
| kbc-devops/cross-claude-mcp | main | `46ff307` |
| kbc-devops/cross-claude-mcp | reload-window | `021e77f` |
| kbc-devops/cross-claude-discord-bridge | main | `5d7ad59` |
| rblank9/cross-claude-mcp | main | `7e0c5de` (원본 오픈소스, toy01 서버가 이 버전 실행 중) |

### 로컬 Windows
- 경로: `C:\Users\heung\kbc-workspace\cross-claude-mcp`
- 현재 브랜치: main
- remote: `https://github.com/kbc-devops/cross-claude-mcp.git`
- bridge 레포 로컬 클론: ❌ 없음 (toy01에만 존재)

### cross-claude 채널 현황
- 마지막 메시지: `#204` (discord:marksun_ → nova-msla-server 호출, 무응답)
- 모든 인스턴스 현재 offline

---

## 3. 다음 작업 목록

### 🔴 1순위 — Mark 직접 필요

| # | 작업 | 비고 |
|---|------|------|
| 1-A | toy01 DB 백업 cron 설정 | `~/.pgpass` 먼저, `rclone` 설치 여부 확인 후 진행 |
| 1-B | 백업 복원 테스트 1회 | pg_dump → 테스트 DB 복원 |
| 1-D | MCP_API_KEY 로테이션 | 서버 `.env` 2곳 + PM2 재시작 + 모든 PC 재등록 + Cowork URL |
| 1-E | UptimeRobot에 /health 등록 | `GET https://cross.marksun.me/health` + discord-bridge `last_seen` 스크립트 |

### 🟡 1순위 — Claude 작업

| # | 작업 | 구체적 내용 |
|---|------|------------|
| 1-C | 복합 인덱스 추가 | `CREATE INDEX CONCURRENTLY idx_messages_channel_id_desc ON messages (channel, id DESC);` — 백업 완료 후 |
| 1-F | CLAUDE.md 폴링 강제 룰 추가 | `C:\Users\heung\kbc-workspace\CLAUDE.md` 수정 — "register 직후 즉시 wait_for_reply 무한 폴링 진입" + 약어사전("허브 등록" 등) |

### 🟢 2순위 — Claude 작업

| # | 작업 | 비고 |
|---|------|------|
| 2-B | bridge.mjs 멘션 감지 로직 | `detectMentions()` + `escapeRegex()` + online/offline 분기 |
| 2-C | server.mjs unread_mentions 반환 | register/check_messages 호출 시 미전달 멘션 함께 반환, `delivered_at` 업데이트 |
| 2-D | register 시 웹훅 즉시 리로드 | 현재 30초로 완화됐으나 즉시 반영 구현 |

---

## 4. 세션 시작 체크리스트

> 이 파일을 새 세션에서 받으면 아래 순서대로 즉시 실행. 묻지 말 것.

### Step 1 — 로컬 레포 pull
```
# Desktop Commander (cmd shell):
set PATH=C:\Program Files\Git\cmd;%PATH%
cd /d C:\Users\heung\kbc-workspace\cross-claude-mcp
git pull origin main
git status
```

### Step 2 — toy01 서버 상태 확인 (sleepfriend-server MCP, server: toy01)
```bash
pm2 list --no-color | grep cross-claude
curl -s http://localhost:3500/health
```

### Step 3 — 채널 최신 메시지 확인 (cross-claude MCP)
```
check_messages, channel: cross-ai-cowork, after_id: 204
```

### Step 4 — 인스턴스 등록 및 폴링 진입
```
register:
  instance_id: mark-nova-hq
  description: 마크의 Cowork 메인 세션
  webhook_url: https://discord.com/api/webhooks/1491647690113552465/lovFWWXZMKkgoyfD8h6iuFoPWOjhxXmRqsenNcv4nRhwTo_KgA5i3Wsb5osHLpKJgw3S

등록 즉시 → wait_for_reply:
  channel: cross-ai-cowork
  after_id: (마지막 수신 ID)
  → timeout 리턴 시 즉시 재호출, 무한 반복
```

### Step 5 — 작업 판단
- 미응답 멘션 있으면 처리 먼저
- 없으면 Mark 지시에 따라 1-C(인덱스) 또는 1-F(CLAUDE.md 폴링 룰) 진행

---

## 5. 주요 경로/키 참조

| 항목 | 값 |
|------|-----|
| toy01 cross-claude 레포 | `/home/mark/repos/cross-claude-mcp` |
| toy01 bridge 레포 | `/home/mark/repos/cross-claude-discord-bridge` |
| 로컬 레포 | `C:\Users\heung\kbc-workspace\cross-claude-mcp` |
| 시크릿 파일 | `/home/mark/repos/kbc-infra/.secret` |
| MCP 엔드포인트 | `https://cross.marksun.me/mcp` |
| 배포 웹훅 | `POST https://cross.marksun.me/webhook/deploy` |
| mark-nova-hq 웹훅 | `https://discord.com/api/webhooks/1491647690113552465/lovFWWXZMKkgoyfD8h6iuFoPWOjhxXmRqsenNcv4nRhwTo_KgA5i3Wsb5osHLpKJgw3S` |
| mark-nova-hub 웹훅 | `https://discord.com/api/webhooks/1491647643359903794/2j01bAe40OrdzSr4x4jiIbIgfXjWts4NXqC4ohxiNx_Bg4oFFQ1c1j28j_EtGWC_dg61` |
| 마지막 채널 메시지 | `#204` |

---

> 작성: mark-nova-hq (Cowork) | 세션일: 2026-05-21  
> 다음 파일: `handoff-2026-05-22-cross-claude-[작업명].md`
