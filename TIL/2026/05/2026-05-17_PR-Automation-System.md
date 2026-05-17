# PR-Automation-System

> 날짜: 2026-05-17
> 원본 노션: [링크](https://www.notion.so/PR-Automation-System-363edfd5a47680709397c4d6b02bfcce)

---

# PR 자동화 시스템 문서



## 📋 목차

- [시스템 구조도](about:blank#%EC%8B%9C%EC%8A%A4%ED%85%9C-%EA%B5%AC%EC%A1%B0%EB%8F%84)
- [상세 구성요소](about:blank#%EC%83%81%EC%84%B8-%EA%B5%AC%EC%84%B1%EC%9A%94%EC%86%8C)
- [실행 흐름도](about:blank#%EC%8B%A4%ED%96%89-%ED%9D%90%EB%A6%84%EB%8F%84)
- [사용 방법](about:blank#%EC%82%AC%EC%9A%A9-%EB%B0%A9%EB%B2%95)
- [버전 차이](about:blank#%EB%B2%84%EC%A0%84-%EC%B0%A8%EC%9D%B4)
---

## 시스템 구조도

```plain text
┌─────────────────────────────────────────────────────────────┐
│                    사용자 입력: /pr-pipeline                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│              .claude/commands/pr-pipeline.md                  │
│              (커스텀 명령어 - 5단계 실행 지시)                    │
└─────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┴─────────────────────┐
        ↓                                               ↓
┌──────────────────────┐                   ┌──────────────────────┐
│  .claude/agents/     │                   │  .omc/skills/       │
│  (커스텀 에이전트)    │                   │  (OMC 스킬)         │
├──────────────────────┤                   ├──────────────────────┤
│ 1. pr-review.md     │                   │ pr-pipeline/       │
│ 2. pr-fix.md        │                   │   skill.md         │
│ 3. pr-create.md     │                   │ (구버전, 4단계)     │
└──────────────────────┘                   └──────────────────────┘
```

---

## 상세 구성요소

### 1️⃣ 커스텀 에이전트 (.claude/agents/) 이 경로에 만들어야함

### pr-review.md - 코드 리뷰 전문가

```yaml
역할: PR 변경사항에 대한 종합적인 코드 리뷰
모델: opus (가장 높은 지능)
도구: Bash, Read, Grep, Glob, LSP 진단
```

리뷰 영역:

1. 정확성
1. 보안
1. 코드 품질
1. 테스트
1. 성능
출력 형식:

```markdown
## 코드 리뷰: [PR 제목]

### 🚨 치명적 이슈
[반드시 수정해야 하는 blocking 이슈들]

### ⚠️ 주요 이슈
[중요한 개선이 필요한 사항들]

### ℹ️ 사소한 이슈
[개선하면 좋은 사항들]

### 💡 제안사항
[선택적 향상사항들]
```

---

### pr-fix.md - 코드 수정 전문가

```yaml
역할: 코드 리뷰 코멘트와 이슈 수정
모델: opus
도구: Bash, Read, Edit, Write, Grep, Glob
```

작업 흐름:

1. 피드백 이해하기
1. 수정 구현
1. 수정사항 검증
1. 변경사항 보고
가이드라인:

- 보수적으로: 언급된 것만 수정, 리팩토링하지 않기
- 의도 보존: 원본 기능을 그대로 유지
- 테스트: 완료 선언 전 테스트 실행
- 소통: 피드백이 불명확하면 명확화 요청
출력 형식:

```markdown
## 수정 요약

### 적용된 수정사항
-[x] file.ts:123 이슈 수정 - 설명
-[x] 엣지 케이스를 위한 테스트 업데이트

### 명확화 필요
-file.ts:456 코멘트 - ~~~에 대한 질문

### 남은 작업
-[ ] 처리되지 않은 이슈 - 사유
```

---

### pr-create.md - PR 생성 전문가

```yaml
역할: GitHub PR 자동 생성
모델: sonnet
도구: Bash, Read, Grep
```

작업 흐름:

1. 현재 상태 확인
1. PR 정보 수집
1. PR 제목 생성 규칙
1. PR 본문 생성
1. PR 생성
에러 처리:

### GitHub CLI 미설치

```bash
# gh가 없으면 안내
echo "GitHub CLI가 설치되지 않았습니다."
echo "설치: https://cli.github.com/"
echo "또는 웹에서 생성: [URL]"
```

### 인증 필요

```bash
# gh auth 상태 확인
gh auth status
```

출력 형식:

```markdown
## ✅ PR 생성 완료

**PR 제목**: [title]
**베이스**: master → [branch]
**URL**: [pr_url]

### 📊 생성된 PR 내용
[본문 요약]
```

주의사항:
- 항상 최신 원격 정보를 fetch 먼저 수행
- PR이 이미 존재하면 중복 생성 방지
- 에러 발생 시 명확한 에러 메시지와 웹 URL 제공

---

### 2️⃣ 커스텀 명령어 (.claude/commands/)  이 경로에 만들어야함

### pr-pipeline.md - 메인 오케스트라

```yaml
설명: 5개 에이전트가 자동으로 코드 리뷰, 수정, 테스트, PR 생성까지 완료
```

5단계 파이프라인:

### 단계 1: 현재 상태 확인

```bash
git branch --show-current
git fetch origin
git diff origin/main...HEAD --stat
git log origin/main..HEAD --oneline
```

### 단계 2: 코드 리뷰 (pr-review 에이전트)

Agent tool을 사용하여 pr-review 서브에이전트를 호출하세요:
- 현재 브랜치의 모든 변경사항 분석
- 심각도별 이슈 식별 (Critical, Major, Minor)
- 구체적인 수정 제안

### 단계 3: 자동 수정 (pr-fix 에이전트)

리뷰에서 발견된 문제점들을 자동으로 수정하세요:
- Agent tool을 사용하여 pr-fix 서브에이전트 호출
- 리뷰 코멘트 기반으로 코드 수정
- 수정 사항 보고

### 단계 4: PR 코멘트 작성

모든 작업 결과를 정리하여 PR 코멘트 형식으로 출력하세요:
- 리뷰 결과 요약
- 수정 사항 목록

### 단계 5: PR 자동 생성 (pr-create 에이전트)

Agent tool을 사용하여 pr-create 서브에이전트를 호출하세요:
- GitHub PR 자동 생성
- 제목, 본문 자동 작성
- PR URL 반환

⚠️ 중요:

> Claude에게: 사용자가 /pr-pipeline을 입력하면 위 5단계를 즉시 자동으로 실행하세요.

main/master 브랜치에 있는 경우에만 “리뷰할 변경사항이 없습니다”라고 알리고 중단하세요.

---

### 3️⃣ OMC 스킬 (.omc/skills/)

### pr-pipeline/skill.md - 구버전 (4단계)

```yaml
name: pr-pipeline
triggers: pr-pipeline, /pr-pipeline
tags: pr, review, automation, pipeline, git
author: custom
```

4단계 파이프라인 (구버전):

### 단계 1: 현재 상태 확인

```bash
git branch --show-current
git fetch origin
git diff origin/main...HEAD --stat
git log origin/main..HEAD --oneline
```

### 단계 2: 코드 리뷰 (pr-review 에이전트)

Agent tool을 사용하여 pr-review 서브에이전트를 호출:
- 현재 브랜치의 모든 변경사항 분석
- 심각도별 이슈 식별 (Critical, Major, Minor)
- 구체적인 수정 제안

### 단계 3: 문제점 자동 수정 (pr-fix 에이전트)

리뷰에서 발견된 문제점들을 자동으로 수정:
- Agent tool을 사용하여 pr-fix 서브에이전트 호출
- 리뷰 코멘트 기반으로 코드 수정
- 수정 사항 보고

### 단계 4: PR 코멘트 작성 및 요약

모든 작업 결과를 정리하여 PR 코멘트 형식으로 출력:
- 리뷰 결과 요약
- 수정 사항 목록
- PR 생성 URL 안내

---

## 실행 흐름도

```plain text
사용자: /pr-pipeline 입력
    ↓
Claude: 커맨드 파일 읽음 (.claude/commands/pr-pipeline.md)
    ↓
자동 실행 시작 (추가 확인 없음)
    ↓
┌─────────────────────────────────────────────────┐
│ 단계 1: 상태 확인                                │
│ - git branch --show-current                      │
│ - git fetch origin                              │
│ - git diff, git log                             │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ 단계 2: 코드 리뷰 (pr-review 에이전트)           │
│ - 변경사항 분석                                  │
│ - 5가지 영역 리뷰                                │
│ - 심각도별 분류 (Critical, Major, Minor)        │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ 단계 3: 자동 수정 (pr-fix 에이전트)              │
│ - 리뷰 코멘트 기반 수정                           │
│ - Edit tool로 코드 변경                          │
│ - 테스트 실행                                    │
│ - 추가 커밋                                      │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ 단계 4: PR 코멘트 작성                           │
│ - 리뷰 결과 요약                                 │
│ - 수정 사항 정리                                 │
└─────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────┐
│ 단계 5: PR 자동 생성 (pr-create 에이전트)         │
│ - 제목, 본문 작성                                │
│ - gh pr create 실행                             │
│ - PR URL 반환                                    │
└─────────────────────────────────────────────────┘
    ↓
완료: PR 생성됨, 사용자가 GitHub에서 머지
```

---

## 사용 방법

### 1. 기본 사용법

```bash
# 1. 기능 개발 후 커밋
git add .
git commit -m "feat: 새로운 기능 추가"

# 2. 원격에 푸시
git push origin feature/my-feature

# 3. 새로운 Claude Code 세션에서
/pr-pipeline

# 4. 자동으로 실행됨:
#    - 코드 리뷰
#    - 문제점 수정
#    - 추가 커밋
#    - PR 생성

# 5. GitHub에서 PR 머지
```

### 2. GitHub CLI 설치 (필수)

```bash
# Windows (winget)
winget install --id GitHub.cli

# 또는 Chocolatey
choco install gh

# 인증
gh auth login
```

### 3. 사용 예시

```bash
# 개발 완료 후
git add .
git commit -m "feat: 사용자 인증 기능 추가"
git push origin feature/user-auth

# Claude Code에서
/pr-pipeline

# 결과:
# ✅ 코드 리뷰 완료 (Critical: 0, Major: 2, Minor: 1)
# ✅ 자동 수정 완료
# ✅ PR 생성됨: https://github.com/user/repo/pull/123
```

---

## 버전 차이

