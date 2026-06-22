# claude-code-skills

나만의 Claude Code 글로벌 스킬 저장소입니다.

각 스킬은 `스킬이름/SKILL.md` 형태로 저장되며, 아래 경로에 복사하면 Claude Code에서 바로 사용할 수 있습니다.

**Windows:** `C:\Users\[사용자명]\.claude\skills\`  
**Mac/Linux:** `~/.claude/skills/`

---

## 스킬 목록

| 스킬 | 설명 |
|------|------|
| [term-wiki](./term-wiki/) | 비개발자를 위한 개발·비즈니스 용어 자동 설명 + Notion 위키 기록 |
| [dev-prod-db-separation](./dev-prod-db-separation/) | Next.js+Supabase+Vercel에서 개발(dev) DB와 운영(prod) DB를 안전하게 분리 + 마이그레이션하는 자율 실행 플레이북. 호출하면 현황을 스스로 조사하고 모르는 값(새 프로젝트 URL/키)만 물어본 뒤 끝까지 진행. 운영 DB 사고 방지 가드레일 포함 |

---

## 설치 방법

1. 원하는 스킬 폴더의 `SKILL.md`를 다운로드
2. `~/.claude/skills/스킬이름/SKILL.md` 경로에 저장
3. Claude Code 재시작 → 자동으로 스킬 인식
