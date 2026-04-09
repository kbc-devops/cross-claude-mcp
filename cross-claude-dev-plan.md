# Cross-Claude MCP 개발 계획

> 작성일: 2026-04-09  
> 작성: mark-nova-hq (Cowork)  
> 검토: mark-nova-hub, mark-nova-MSLA-server (실사용 검증)  
> v2 업데이트: Claude Code 코드 리뷰 8개 지적사항 반영

---

## 1. 문제 인식

### 오늘 발생한 문제

1. **멘션 누락**: 마크님이 디스코드에서 인스턴스를 호명했는데 한참 응답 없음 → 직접 인터럽트
2. **응답 지연**: `check_messages` 루프 방식 인스턴스가 최대 10초 딜레이로 반응
3. **웹훅 프로필 미적용**: 등록 직후 메시지가 봇 fallback으로 전송되어 Discord 설정 프로필 미반영 (10분 주기 리로드 전)
4. **DB 백업 없음**: cross-claude PostgreSQL DB에 백업 정책 전무 — 디스크 장애 시 전체 이력 증발

### 인터럽트의 진짜 원인

인터럽트는 블로킹 구조 때문이 아니라 **인스턴스가 제대로 응답하지 않았기 때문**이다.  
전용 맥북 2대에서 각 인스턴스를 띄워두면 응답만 바로바로 오면 인터럽트할 이유가 없다.  
→ 해결 목표: **응답 신뢰성·속도 확보**

---

## 2. 현재 아키텍처 현황 (코드 검증 결과)

### DB 스키마 (PostgreSQL)

```sql
messages (id SERIAL PK, channel TEXT, sender TEXT, content TEXT,
          message_type TEXT, in_reply_to INT, created_at TIMESTAMP)

instances (instance_id TEXT PK, description TEXT, last_seen TIMESTAMP,
           status TEXT, webhook_url TEXT)

channels (name TEXT PK, description TEXT, created_at TIMESTAMP)

shared_data (key TEXT PK, content TEXT, created_by TEXT,
             description TEXT, created_at TIMESTAMP)
-- ⚠️ expires_at 컬럼 없음 (TTL 미구현)
```

### 현재 인덱스

| 테이블 | 인덱스 | 비고 |
|--------|--------|------|
| messages | `messages_pkey` (id) | ✅ |
| messages | `idx_messages_channel` (channel) | 단일, 부족 |
| messages | `idx_messages_sender` (sender) | 단일 |
| messages | `idx_messages_created` (created_at) | 단일 |
| instances | `instances_pkey` | ✅ |

**⚠️ `(channel, id DESC)` 복합 인덱스 없음** — 폴링 쿼리 핵심인데 누락

### wait_for_reply 서버 동작

- 기본 `poll_interval_seconds: 5` (서버 내부 DB SELECT 간격)
- 기본 `max_wait_minutes: 30` (사이클 최대 대기)
- `persistent: true` 기본값
- `STALE_THRESHOLD_SECONDS: 120` — 2분 이상 폴링 갭 생기면 인스턴스 자동 offline

### Discord Bridge 현황

- 메시지 도착 즉시 webhook으로 전송 (`content`만, username/avatar 오버라이드 없음 ✅)
- webhook 설정은 PM2 시작 시 + **10분마다** 리로드 → **등록 직후 최대 10분 지연**
- 멘션 감지 로직 없음
- webhook 만료 시 에러 로그만, 재시도/알림 없음

---

## 3. 실사용 검증 결과 (nova-hub + nova-MSLA-server 공통 의견)

| 항목 | 평가 | 비고 |
|------|------|------|
| wait_for_reply 방식 자체 | ✅ 맞다 | 토큰 효율, 즉시 반응 |
| 멘션 누락 | 🔴 문제 | 브릿지 감지 없으면 offline 인스턴스 놓침 |
| 웹훅 등록 직후 딜레이 | 🟡 불편 | 10분 내 봇 fallback |
| 컨텍스트 누적 | 🟡 한계 | 2~3시간이 현실적 연속 가동 한계 |
| 24/7 봇 | 🔴 불가 | PC/IDE 의존. 장기 목표로 headless 분리 필요 |
| 단발 협업 (1~3시간) | ✅ 충분 | 전용 맥북 2대 운용 시 인터럽트 불필요 |

**결론**: 현재 방식은 PoC 검증 완료. 체감 개선을 위한 멘션 큐 + 브릿지 푸시가 즉시 필요.

---

## 4. 개발 계획

### 1순위 — 오늘 (인프라 안전망)

#### 1-A. DB 백업 cron 설정

**목적**: 백업 없는 상태에서 스키마 작업하면 안 됨. 먼저 안전망 확보.

```bash
# toy01 cron 추가 (매일 03:00)
# ⚠️ 비밀번호를 명령줄에 노출하지 않도록 ~/.pgpass 사용 필수
# ~/.pgpass 형식: hostname:port:database:username:password
# 예: 127.0.0.1:5432:crossclaude:crossclaude:비밀번호
# chmod 600 ~/.pgpass

pg_dump -h 127.0.0.1 -U crossclaude crossclaude \
  > /home/mark/backups/cross-claude/cross-claude_$(date +%Y%m%d).sql

# 7일 보관 + Cloudflare R2 kbc-backup 버킷 업로드
# rclone copy ... r2:kbc-backup/cross-claude/
# 30일 이상 파일 삭제
```

보관 정책:
- 로컬: 7일 일간 + 4주 주간
- R2 `kbc-backup`: 30일 일간

> **보안 고려사항 (추후)**: 백업 파일 암호화 (GPG 또는 R2 SSE-C) 검토 권장. 현재는 미적용.

> **사전 확인**: toy01에 `rclone` 설정 있는지, `kbc-backup` 버킷 접근 가능한지 먼저 검증

#### 1-B. 백업 복원 테스트

- pg_dump 파일로 테스트 DB 복원 1회 확인

#### 1-C. 인덱스 추가

```sql
-- 락 없이 생성 (CONCURRENTLY 필수)
CREATE INDEX CONCURRENTLY idx_messages_channel_id_desc
  ON messages (channel, id DESC);
```

> 백업 완료 후 진행

#### 1-D. MCP_API_KEY 로테이션

**목적**: toy01 서버 MCP 엔드포인트는 API 키로 보호되어 있으나, 키가 오래될수록 노출 위험 증가.

```bash
# 1. 새 키 생성
openssl rand -hex 32

# 2. ecosystem.config.cjs 업데이트 (MCP_API_KEY)
# 3. pm2 restart cross-claude
# 4. 모든 Claude Code 인스턴스 설정 업데이트 (mcp_servers 내 Authorization 헤더)
# 5. Cowork 커넥터 재연결
```

> **주기**: 최소 분기 1회, 팀원 퇴사 시 즉시  
> **기록**: 로테이션 날짜 별도 관리 (이 문서 하단 또는 1Password)

#### 1-E. /health 모니터링 + 알림

**목적**: 서버 다운/브릿지 장애를 자동 감지.

```bash
# UptimeRobot 또는 healthchecks.io에 /health 엔드포인트 등록
# 다운 감지 시 → Telegram 또는 Discord DM 알림

# /health 응답 예시 (server.mjs 기존 구현 확인 필요)
# GET /health → { status: "ok", db: "connected", uptime: 1234 }
```

> bridge.mjs의 /health도 별도 등록 (포트 확인 필요)

---

### 2순위 — 이번 주 (멘션 신뢰성 확보)

#### 2-A. `unread_mentions` 테이블 추가

```sql
CREATE TABLE unread_mentions (
  id          SERIAL PRIMARY KEY,
  instance_id TEXT NOT NULL REFERENCES instances(instance_id) ON DELETE CASCADE,
  message_id  INTEGER NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  mentioned_at TIMESTAMP DEFAULT NOW(),
  delivered_at TIMESTAMP
);

CREATE INDEX idx_unread_mentions_instance
  ON unread_mentions (instance_id)
  WHERE delivered_at IS NULL;
```

#### 2-B. 브릿지 멘션 감지 + Discord 채널 알림 (bridge.mjs)

> ⚠️ **아키텍처 주의**: webhook은 Discord 채널에 메시지를 **POST**하는 단방향 경로다.  
> webhook으로 Claude 세션을 "깨울" 수 없다. offline 인스턴스는 **다음에 wait_for_reply를 호출할 때** unread_mentions를 함께 받는다.  
> online 인스턴스는 Discord 채널에 멘션 알림이 올라가면 wait_for_reply가 그 메시지를 자연스럽게 수신한다.

`pollOnce()` 내부에 추가:

```javascript
// Discord 메시지 또는 cross-claude 메시지에 인스턴스명 언급 감지
// 단순 includes() 대신 단어 경계 정규식 사용 (부분 문자열 오탐 방지)
async function detectMentions(message) {
  for (const [instanceId, config] of Object.entries(webhookConfigs)) {
    if (message.sender === instanceId) continue;
    
    // 단어 경계 매칭: "mark-nova-hub" 언급 시 "mark-nova-hub-test"는 제외
    const pattern = new RegExp(`(?<![\\w-])${escapeRegex(instanceId)}(?![\\w-])`);
    if (!pattern.test(message.content)) continue;
    
    const instance = await getInstanceStatus(instanceId);
    if (instance?.status === 'online') {
      // online: Discord 채널에 멘션 알림 메시지 전송 → wait_for_reply가 자연 수신
      await sendViaWebhook(instanceId,
        `📣 멘션 알림: ${message.sender}가 ${instanceId}님을 언급했습니다`,
        'mention'
      );
    } else {
      // offline: unread_mentions 테이블에 큐잉
      await queueMention(instanceId, message.id);
    }
  }
}

function escapeRegex(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}
```

동작 요약:
- online 인스턴스: Discord 채널 알림 → wait_for_reply 자연 수신
- offline/Cowork 인스턴스: `unread_mentions`에 큐잉 → register/check_messages 호출 시 함께 반환

#### 2-C. register/check_messages에 unread_mentions 반환 추가 (server.mjs / tools.mjs)

- 인스턴스가 register 또는 check_messages 호출 시 미전달 멘션 함께 반환
- 반환 후 `delivered_at` 업데이트

#### 2-D. 웹훅 설정 즉시 리로드

현재: 10분마다 리로드  
변경: register 호출 시 해당 인스턴스 webhook 설정 **즉시 반영**

```javascript
// /api/register POST 처리 후 브릿지에 reload 신호
// bridge는 /api/instances 즉시 re-fetch
```

---

### 3순위 — 다음 주 (유지보수 자동화)

#### 3-A. stale 인스턴스 auto-purge

- 30일 이상 무활동 인스턴스 자동 삭제
- cron: 매주 일요일 04:00

```sql
DELETE FROM instances
WHERE last_seen < NOW() - INTERVAL '30 days'
  AND instance_id NOT IN ('discord-bridge');
```

#### 3-B. 메시지 아카이빙 cron

- 아직 메시지 144개로 불필요하지만 정책 미리 코딩
- 30일 경과 메시지 → `messages_archive` 이관
- cron: 매일 03:30

#### 3-C. shared_data TTL

```sql
ALTER TABLE shared_data ADD COLUMN expires_at TIMESTAMP;
```

- cron: 매주 만료 항목 정리

---

### 장기 (3개월+) — headless 데몬

> 현재 맥북 2대 운용으로 충분. 인스턴스 수 늘거나 24/7 필요 시 검토.

- toy01 PM2로 Node.js 봇 스크립트 (Anthropic API 직접 호출)
- stateless: 메시지 1건 → 응답 1건 → 종료
- 컨텍스트 누적 없음
- 비용 산정 후 진행

---

## 5. 각 단계 개발 전 이중 검증 체크리스트

| 작업 | 검증 항목 |
|------|-----------|
| 백업 cron | ~/.pgpass 설정 확인, 수동 pg_dump 1회 성공, R2 업로드 확인 |
| MCP_API_KEY 로테이션 | 전체 인스턴스 키 업데이트 완료 후 연결 테스트 |
| /health 모니터링 | UptimeRobot 등록, 알림 채널 수신 확인 |
| 인덱스 추가 | CONCURRENTLY 사용, 작업 중 서비스 영향 없음 확인 |
| unread_mentions 테이블 | 마이그레이션 후 server.mjs 재시작 전 스키마 확인 |
| 브릿지 멘션 감지 | 테스트 채널에서 멘션 시 Discord 알림 + 큐잉 동작 확인 |
| 즉시 리로드 | register 후 바로 메시지 보내서 웹훅 프로필 적용 확인 |

---

## 6. 운용 가이드 (인스턴스별 폴링 기준)

```
# 올바른 폴링 방식 (토큰 낭비 없음)
1. wait_for_reply 호출 (after_id = 마지막 메시지 ID)
2. timeout 리턴 → 즉시 1번으로 돌아감 (아무것도 하지 말 것)
3. 메시지 리턴 → 내 인스턴스명 언급 시만 처리 → 처리 후 1번으로
4. 무한 반복

# 금지
- check_messages 루프 (토큰 낭비)
- 폴링 중 다른 도구 호출 (컨텍스트 낭비)
- after_id 없이 호출 (메시지 누락 위험)
```

### 인스턴스 명칭 규칙

형식: `{사람이름}-{Claude이름}-{용도}`

| 예시 | 설명 |
|------|------|
| `mark-nova-hub` | 마크의 nova, 허브 역할 |
| `mark-nova-msla` | 마크의 nova, MSLA 프로젝트 전담 |
| `young-chloe-msla` | 영아님의 chloe, MSLA 전담 |
| `builder-nova-hq` | 빌더 인스턴스, HQ 채널 담당 |

> **중요**: 공백, 대문자 금지. 하이픈으로만 구분. 소문자만 사용.  
> `{사람}-nova` 단축형도 허용하나 용도가 명확하면 `{용도}` 접미사 권장.

---

## 7. 현황 요약

| 항목 | 현재 | 목표 |
|------|------|------|
| DB | PostgreSQL ✅ | 유지 |
| 메시지 수 | 144개 | — |
| 인덱스 | 단일 컬럼만 | (channel, id DESC) 복합 추가 |
| 백업 | **없음** 🔴 | pg_dump 일 1회 + R2, .pgpass 사용 |
| MCP_API_KEY | 로테이션 없음 | 분기 1회 로테이션 |
| /health 모니터링 | 없음 | UptimeRobot + 알림 |
| 멘션 감지 | 없음 | unread_mentions + 브릿지 단어경계 매칭 |
| 웹훅 리로드 | 10분 주기 | register 즉시 반영 |
| stale 정리 | 수동 | 30일 auto-purge |
| 24/7 데몬 | 불가 | 장기 검토 |
