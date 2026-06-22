---
name: dev-prod-db-separation
description: >-
  Next.js + Supabase + Vercel 프로젝트에서 "지금까지 하나의 공유 DB로 테스트하다가, 본격 서비스로
  가면서 개발(dev) DB와 운영(prod) DB를 안전하게 분리"하는 작업의 검증된 플레이북. 사용자가
  "개발 DB랑 운영 DB 나누자", "dev/prod 분리", "테스트용 DB 따로 만들자", "스테이징 환경 구축",
  "운영 DB에 테스트 데이터 쌓이는 거 막고 싶어", "Preview는 dev, Production은 운영으로",
  "Supabase 프로젝트를 환경별로 분리", "확장/앱이 운영에 직결돼 있는데 dev로 돌리고 싶어" 등
  환경 분리·DB 분리·스테이징을 언급하면 — 명시적으로 "스킬"을 말하지 않아도 — 반드시 이 스킬을 사용한다.
  운영 DB 사고(데이터 오염·dev↔운영 혼용·실수 마이그레이션)를 막는 가드레일까지 포함한다.
---

# 개발/운영 DB 분리 플레이북

처음엔 누구나 하나의 Supabase 프로젝트로 개발+테스트를 한다. 사용자가 늘고 본격 서비스가 되면
**테스트 데이터가 운영 DB에 섞이는 사고**가 치명적이 된다. 이 스킬은 그 시점에 **dev DB와 운영 DB를
완전히 분리**하는 전 과정을, 실제 수행해 검증된 순서로 안내한다.

> 핵심 철학: **코드는 git을 타지만 DB는 안 탄다.** 그래서 분리는 "데이터 이전"이 아니라
> "환경마다 어느 DB를 바라보게 할지 배선하고, 스키마는 양쪽에 각각 적용"하는 일이다.
> 운영 데이터는 어느 방향으로도 복사·이전하지 않는다.

스택 가정: **Next.js (App Router) + Supabase + Vercel**. 다른 스택도 원리는 동일(환경별 연결 분리 + 스키마 양쪽 적용 + 가드레일).

---

## 실행 모드 — 이 스킬이 호출되면 (중요)

이 스킬의 목적은 **사용자가 매번 설명하지 않아도, 호출 한 번으로 dev/운영 DB를 완전히 분리하고
마이그레이션까지 끝내는 것**이다. 따라서 참고만 하지 말고 **끝까지 실행**한다. 원칙:

1. **스스로 파악한다.** 현재 DB 구성·연결 상태·스키마·테이블 접두어·env 파일·확장 유무 등은
   코드베이스와 MCP/REST로 직접 조사한다(§0). 사용자에게 다시 묻지 않는다.
2. **사람 손이 꼭 필요한 것만, 한 번에 모아 묻는다.** AI가 코드/MCP로 알아낼 수 없는 값과 손작업이
   있다 — **추측하지 말고 반드시 물어본다.** 처음에 아래를 한꺼번에 요청하고(가능하면 AskUserQuestion 등으로
   묶어서), 받는 즉시 나머지를 자율로 진행한다:

   **(a) 새 dev 프로젝트 자격증명 — AI가 절대 모름, 반드시 질문:**
   - 새 Supabase **Project URL** (`https://<ref>.supabase.co`)
   - **anon (publishable) key**
   - **service_role (secret) key** — 서버/관리 작업·검증에 필수
   - (확장/외부 클라이언트가 있으면) **개발용 배포본 식별자**(예: 언패킹 확장 ID)와 **운영 배포본 식별자**(예: 웹스토어 확장 ID)

   **(b) 사용자만 할 수 있는 손작업 — AI가 안내, 실행은 사용자:**
   - 새 Supabase **프로젝트 생성**(계정/결제/리전/DB 비밀번호 결정)
   - 새 프로젝트 **SQL Editor에서 스키마 SQL 실행**(MCP가 그 계정에 못 닿을 때). AI가 파일을 만들어 주면 붙여넣고 RUN.
   - **Auth 대시보드 토글**(Email provider ON, Confirm email OFF)
   - **Vercel Preview 스코프 env 입력**(AI가 정확한 값·순서 제공, 클릭은 사용자)
   - **실제 확장/브라우저 최종 동작 테스트**(자동화로 확장을 못 띄움)

   > 받은 키는 **localhost는 `.env.development.local`에 AI가 기록**(gitignore 확인), **Vercel Preview는
   > 사용자가 대시보드 입력**. 운영 키는 절대 dev 자리에, dev 키는 절대 운영 자리에 넣지 않는다(가드레일).
   > 키는 비밀이므로 로그/커밋/외부로 노출하지 않는다.
3. **운영 DB에 쓰기/마이그레이션/삭제는 실행 전 반드시 사용자 확인**을 받는다(가드레일).
4. **각 단계 자가 검증.** "됐을 것"이라 추측해 완료 선언 금지 — 배포본 grep·로그인·업로드 재현 등으로
   실제 확인하고 결과를 보고한다(§6).
5. 작업 끝에 **가드레일을 CLAUDE.md/DEV_PROCESS.md에 박제**(§9)해서 다음 세션도 같은 규칙을 따르게 한다.

> 한 줄: "조사는 알아서, 막힌 손작업만 한 번에 요청, 나머지는 끝까지, 운영 쓰기만 확인."

아래 §0~§10은 그 실행 순서의 상세다.

---

## 0. 먼저 현황 진단 (추측 금지, 사실 확인)

분리 전에 반드시 실제 상태를 본다. 흔히 가정과 현실이 다르다.

- 현재 몇 개의 Supabase 프로젝트가 있고 각각 무엇인지: `list_projects`(MCP) 또는 대시보드.
- **"dev"라고 부르던 게 실제로 분리돼 있나?** 배포본의 client JS에서 어떤 Supabase URL을 쓰는지 직접 확인(§7의 grep 기법). 종종 "Preview가 dev인 줄 알았는데 운영 DB 직결"인 경우가 있다.
- 각 DB에 **다른 앱/테이블이 섞여 있나?** (공유 프로젝트면 내 앱 테이블만 골라 다뤄야 함)
- 운영 DB에 이미 **실데이터/내 테스트 데이터**가 얼마나 있나(`list_tables` 행 수).

진단 결과를 사용자에게 보고하고, 가정과 다르면 그 사실부터 알린다.

---

## 1. 목표 아키텍처 결정

권장: **MIMIC 전용(=내 앱 전용) 새 dev 프로젝트 신설.**

| 옵션 | 설명 | 권장도 |
|------|------|--------|
| A. 기존 공유 DB의 내 테이블만 재사용 | 타 앱과 한집살이 → 남의 마이그레이션이 내 앱을 깨거나 스키마 드리프트 위험 | △ |
| B. **전용 dev 프로젝트 신설** | 깨끗·완전 분리. 결제 등 민감 기능 대비에도 안전 | ✅ |

비용 팁: Supabase 무료는 **조직당 2프로젝트**. 한도가 차면 **새 계정**을 만들면 깨끗한 무료 quota를 얻는다(단 §3의 MCP 접근 제약 발생).

---

## 2. 스키마의 "진실의 원천" = 운영 라이브 스키마 (마이그레이션 파일 아님)

⚠️ **레포의 `supabase/migrations/*.sql`을 그대로 새 dev에 적용하지 말 것.** 마이그레이션 파일은
운영과 **드리프트**돼 있는 경우가 많다(운영이 손으로 패치됐거나, 일부 마이그레이션이 운영에 미적용).
실제 코드가 도는 **운영 라이브 스키마**가 진실이다.

운영(MCP 접근 가능)에서 catalog로 전부 추출한다. 빠뜨리면 dev가 미묘하게 깨진다 — 특히
**RLS 정책·트리거·함수·Storage 버킷/정책**이 가장 잘 누락된다.

추출 쿼리(`execute_sql`로 운영 project_id에 실행, 내 앱 테이블 접두어로 필터):
```sql
-- 컬럼
select table_name, column_name, data_type, is_nullable, coalesce(column_default,'')
from information_schema.columns
where table_schema='public' and table_name like 'PREFIX\_%' order by table_name, ordinal_position;
-- 제약(PK/FK/UNIQUE/CHECK) + 정의
select t.relname, c.contype, c.conname, pg_get_constraintdef(c.oid)
from pg_constraint c join pg_class t on t.oid=c.conrelid join pg_namespace n on n.oid=t.relnamespace
where n.nspname='public' and t.relname like 'PREFIX\_%' order by t.relname, c.contype desc;
-- 인덱스
select tablename, indexname, indexdef from pg_indexes
where schemaname='public' and tablename like 'PREFIX\_%';
-- RLS 정책
select tablename, policyname, cmd, roles::text, coalesce(qual,''), coalesce(with_check,'')
from pg_policies where schemaname='public' and tablename like 'PREFIX\_%';
-- 트리거(테이블 + auth.users 가입 트리거)
select event_object_schema, event_object_table, trigger_name, action_timing, event_manipulation, action_statement
from information_schema.triggers
where (event_object_schema='public' and event_object_table like 'PREFIX\_%')
   or (event_object_schema='auth' and event_object_table='users');
-- 함수 정의(트리거 함수 + RPC)
select proname, pg_get_functiondef(p.oid) from pg_proc p join pg_namespace n on n.oid=p.pronamespace
where n.nspname='public' and proname in (/* 트리거/RPC 이름들 */);
-- Storage 버킷
select id, name, public, file_size_limit, allowed_mime_types::text from storage.buckets;
-- Storage 정책(내 버킷 이름이 들어간 것)
select policyname, cmd, roles::text, coalesce(qual,''), coalesce(with_check,'')
from pg_policies where schemaname='storage' and tablename='objects';
```

이걸로 **하나의 멱등 SQL 파일**(`supabase/dev-setup/01_schema.sql`)을 만든다. 구성 순서:
1) `create extension if not exists pgcrypto;` (gen_random_bytes 등 기본값에 필요)
2) 테이블 생성(컬럼/기본값/NOT NULL/PK/UNIQUE/CHECK 인라인, **FK는 제외** — 순서 의존성 회피)
3) FK 일괄 `alter table ... add constraint`
4) 인덱스
5) 함수 → 트리거
6) `enable row level security` + 정책(정책 없는 테이블은 RLS만 켜서 service-role 전용으로)
7) Storage 버킷 insert + 정책

이 파일을 레포에 커밋(재현성). 데이터는 넣지 않는다(빈 스키마).

---

## 3. 새 dev 프로젝트 생성 + 접근 방식

- 새 프로젝트 생성(리전은 무방 — 기능 아닌 지연시간만 영향. 한국이면 도쿄가 가깝다).
- ⚠️ **새 프로젝트가 별도 Supabase 계정에 있으면 Claude의 Supabase MCP로 접근 불가**
  (MCP는 연결된 계정만 봄). 이 경우 dev 조작은:
  - **대시보드 SQL Editor에 SQL 붙여넣기**, 또는
  - **service_role 키로 REST/Admin API**(curl/python) — 버킷 생성, 사용자 생성/확인, 데이터 점검 등.
- 그래서 §2의 스키마 SQL은 "내가 MCP로 실행"이 아니라 **"사용자가 새 프로젝트 SQL Editor에서 RUN"** 하는 산출물로 전달한다.

---

## 4. Auth 설정(새 프로젝트는 기본이 막혀 있음) + 테스트 계정

새 Supabase 프로젝트는 종종 이메일 로그인이 꺼져 있다. 증상별 해결:
- `Email logins are disabled` (422) → **Authentication → Sign In/Providers → Email → Enable email provider** ON + Save.
- `이메일 인증 필요`(Email not confirmed, 400) → 프로젝트 **Confirm email** 끄거나, 사용자 생성 시 **Auto Confirm** 체크. 이미 만든 미인증 계정은 admin API로 `email_confirm:true` 처리.

테스트 계정은 **플랜별로 2개** 만든다(가입 트리거가 앱 user 행을 자동 생성):
- PRO 계정(유료 기능·게이팅 통과 테스트)
- FREE 계정(무료 구간·한도·게이팅 차단 테스트)

service_role Admin API로 생성/인증/플랜설정 예시(§A 스니펫 참고). 플랜은 앱 user 테이블 `plan` 컬럼을 REST PATCH로 바꾼다.

---

## 5. 연결 배선 전환 (여기가 분리의 본체)

목표: **localhost·Preview = dev DB / Production = 운영 DB.**

1. **localhost**: `.env.development.local`에 dev 프로젝트 값(URL/anon/service_role). Next.js는 dev 모드에서 이 파일을 `.env.local`보다 우선 로드 → 아무것도 안 바꿔도 `npm run dev`는 dev DB 사용.
2. **Vercel Preview**: 대시보드 Environment Variables에서 **Preview 스코프에만** dev 값 추가.
   **Production 스코프는 운영 값 그대로 둔다(절대 변경 금지).**
3. 클라이언트가 쓰는 그 외 식별자(예: 확장 ID, 외부 키)도 환경별로 분리(§8).

⚠️ 함정:
- **`NEXT_PUBLIC_*`는 빌드 타임에 코드에 인라인**된다. Vercel에서 값만 바꾸고 끝내면 적용 안 됨 →
  **반드시 해당 브랜치를 재빌드**(빈 커밋/실커밋 push, 또는 캐시 없이 redeploy).
- Vercel "Redeploy"가 **Production을 재배포**하면 Preview엔 반영 안 된다. **dev 브랜치 빌드**가 새로 돌아야 Preview env가 박힌다.
- Vercel은 Development/Preview/Production **3스코프**가 따로다. Preview만 바꿨는지 재확인.

---

## 6. 검증 (직접 — 추측 금지)

분리는 반드시 "실제로 어느 DB를 보는가"로 검증한다.

- **배포본 client JS grep**: 페이지의 로드된 `<script src>`들을 받아 supabase project ref를
  검색해 dev/운영 어느 쪽인지 확인. 인증 보호된 Vercel Preview는 MCP의
  `get_access_to_vercel_url`로 우회 URL을 받아 접근. (§B 스니펫)
- **로그인/읽기/쓰기**: 로컬·Preview에서 테스트 계정으로 로그인 → 목록/생성 동작 → dev DB에만 쌓이는지.
- ⚠️ **service_role은 서버 전용**이라 client JS grep으로 검증 불가. Preview/서버 라우트가 dev
  service_role을 쓰는지는 별도로 확인(서버 쓰기 동작을 dev에서 한 번 일으켜 어느 DB에 들어가는지).
- 클라이언트가 직접 Storage에 올리면(anon 업로드 등) 그 **업로드 호출을 그대로 재현**해 dev에서
  성공하는지 본다(정책 누락 함정, §8).

---

## 7. 클라이언트/확장이 운영에 하드코딩돼 있을 때

브라우저 확장·데스크톱 앱 등은 운영 URL/키를 **하드코딩**하는 경우가 많다. 그대로 dev에서 쓰면
운영에 기록된다. **빌드 식별자로 dev/운영 자동 판별**이 가장 안전(수동 토글은 깜빡 위험):

- Chrome 확장: `chrome.runtime.id`가 **웹스토어 배포본 ID면 운영, 아니면(개발자 언패킹) dev**.
  ```js
  const IS_DEV = chrome.runtime.id !== 'STORE_EXTENSION_ID';
  const SUPABASE_URL = IS_DEV ? DEV_URL : PROD_URL;  // ANON_KEY, WEBAPP_ORIGIN 동일
  ```
  → 배포본은 자기 ID라 항상 운영을 가리킴(섞일 위험 0). 웹앱 쪽은 확장 ID를 env(`NEXT_PUBLIC_EXTENSION_ID`)로 두고 dev엔 언패킹 ID를 넣는다.
- Storage 버킷 정책 함정: 확장이 **anon 키로 업로드**하면 dev 버킷에 **anon INSERT/UPDATE(upsert)/SELECT 정책**이 있어야 한다. 새로 만든 버킷엔 보통 없어 RLS로 막힌다 → 운영 정책을 복제해 dev에 추가하고, **업로드를 재현해 200 확인**.
- 로컬 앱 포트 고정: 확장의 `externally_connectable`/origin이 `localhost:3000`이면 dev 서버를 **3000으로** 띄워야 연동된다.

---

## 8. 옛 공유 dev 정리 (선택, 신중 — 마지막에)

새 전용 dev로 옮긴 뒤 옛 공유 프로젝트의 내 앱 테이블은 고아가 된다. 정리는 **선택**이며,
**공유 라이브 DB라 매우 신중**해야 한다(타 앱 영향 0이어야 함). 정리 전 확인:
1. **타 앱→내 테이블 FK 없음** 확인:
   ```sql
   select conname, conrelid::regclass, confrelid::regclass from pg_constraint
   where contype='f' and confrelid::regclass::text like 'PREFIX\_%' and conrelid::regclass::text not like 'PREFIX\_%';
   ```
2. **auth.users 트리거가 내 테이블만** 건드리는지(드롭 시 타 앱 가입 안 깨지게).
3. 그다음 **내 테이블만** `drop table ... cascade` + 내 전용 트리거/함수만 제거. **범용 함수(set_updated_at 등)는 남긴다**(타 앱 공유 가능).
4. 정리 후 타 앱 테이블 수가 그대로인지 검증.

---

## 9. 가드레일 박제 (CLAUDE.md) — 가장 중요

분리 자체보다 **재발 방지**가 핵심. 프로젝트 `CLAUDE.md`(모든 세션이 읽는 규칙)에 절대 규칙을 박는다:

- 프로젝트 식별표: 운영 ref / dev ref / (폐기) 옛 공유 ref + "MCP는 운영에 접근 가능 → 주의".
- 연결 배선 고정: localhost·Preview=dev / Production=운영. **변경 금지.**
- ⛔ 금지: 운영에 테스트 데이터 INSERT / dev↔운영 데이터 복사·이전·동기화 / env로 dev·Preview를 운영 DB에 또는 운영을 dev에 연결 / 무확인 운영 DROP·DELETE·마이그레이션 / MCP 호출 전 project_id 미확인.
- ✅ 스키마 변경 = **양쪽에 같은 DDL을 각각 적용**(운영=MCP, dev=SQL Editor). "복사/이전" 아님. 데이터는 안 옮김.
- 운영 쓰기/마이그레이션이 꼭 필요하면 **사용자에게 명시 확인 후** 실행.
- ⚠️ **같은 폴더에서 `npm run dev` 여러 개 금지** — 같은 `.next`를 공유해 캐시 오염(`Cannot read properties of undefined (reading 'clientModules')` 등). 병렬은 worktree(폴더별 별도 `.next`). 복구: 모든 dev 서버 종료 → `.next` 삭제 → 1개만 재기동.

DEV_PROCESS.md(개발→배포 표준 순서)도 같이 갱신: dev push=Preview(dev DB)→검증→main push=Production(운영 DB).

---

## 10. 기능 게이팅 주의 (분리 후 "안 되는데?"의 흔한 원인)

플랜(free/pro)이나 권한에 따라 **기능이 다르게 동작**하면, 새 dev 계정이 free라서 기능이 안 보이는 걸
"마이그레이션 실패"로 오해하기 쉽다. (실제 사례: 자동 어노테이션이 `if(!isFree)`로 유료 전용이라
free 계정에선 안 생김 — 스키마 문제 아님.) 그래서 **dev에 PRO·FREE 계정을 모두** 두고, "안 된다"는
증상은 **코드의 플랜 분기부터** 확인한다. (`grep`으로 `isFree`/`plan` 게이팅 위치를 찾아 근거 제시.)

---

## 부록 A. service_role REST/Admin API 스니펫 (MCP 미접근 dev 조작용)

`.env.development.local`에서 URL/service_role을 읽어 사용. (Windows 콘솔 한글 깨짐 방지: `PYTHONIOENCODING=utf-8`)

```python
import json, urllib.request, urllib.error
env={}
for line in open('.env.development.local', encoding='utf-8'):
    line=line.strip()
    if line and not line.startswith('#') and '=' in line:
        k,v=line.split('=',1); env[k]=v.strip()
url=env['NEXT_PUBLIC_SUPABASE_URL']; key=env['SUPABASE_SERVICE_ROLE_KEY']
H={'apikey':key,'Authorization':'Bearer '+key,'Content-Type':'application/json'}
# 사용자 생성(인증 포함):  POST /auth/v1/admin/users  {email,password,email_confirm:true}
# 기존 사용자 인증 처리:    PUT  /auth/v1/admin/users/<id>  {email_confirm:true}
# 앱 user 행 확인(트리거):  GET  /rest/v1/<user_table>?select=email,plan&email=eq.<EMAIL>
# 플랜 변경:                PATCH /rest/v1/<user_table>?email=eq.<EMAIL>  {plan:'pro',...}
# 버킷 생성:                POST /storage/v1/bucket  {id,name,public:true}
# 업로드 재현(anon 키로!):  POST /storage/v1/object/<bucket>/<path>  (apikey=ANON, x-upsert:true)
```

## 부록 B. 배포본이 보는 DB 확인 (client JS grep)

```bash
# 보호된 Vercel Preview면 먼저 MCP get_access_to_vercel_url로 _vercel_share URL 획득 후 쿠키로 접근.
BASE="https://<branch-alias>.vercel.app"; JAR=$(mktemp)
curl -sL -c $JAR -b $JAR "$BASE/?_vercel_share=<TOKEN>" -o /dev/null
html=$(curl -sL -c $JAR -b $JAR "$BASE/auth/login")
echo "$html" | grep -oE '/_next/static/[^"]+\.js' | sort -u | head -60 \
  | while read c; do curl -sL -c $JAR -b $JAR "$BASE$c"; done \
  | grep -oE 'PROD_REF|DEV_REF' | sort | uniq -c
# 또는 로그인 후 페이지 컨텍스트에서: document.querySelectorAll('script[src]') 받아 fetch+검색
```

---

## 진행 체크리스트
- [ ] 0 현황 진단(실제 연결 grep 포함)
- [ ] 1 전용 dev 프로젝트로 결정
- [ ] 2 운영 라이브 스키마 추출 → dev-setup SQL 작성·커밋
- [ ] 3 새 dev 프로젝트 생성(+MCP 접근 가능 여부 파악)
- [ ] 4 Auth 활성화 + PRO/FREE 테스트 계정
- [ ] 5 localhost + Vercel Preview만 dev로 배선(Production 불변) + 재빌드
- [ ] 6 client JS grep + 로그인/쓰기로 검증(service_role 별도 확인)
- [ ] 7 확장/클라이언트 dev 자동판별 + Storage 정책
- [ ] 8 (선택) 옛 공유 dev 내 테이블만 정리
- [ ] 9 CLAUDE.md/DEV_PROCESS.md 가드레일 박제
- [ ] 10 PRO/FREE 양쪽으로 기능 게이팅 확인
