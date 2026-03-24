# 네이버 블로그 수집기 (Blog Collector)

Claude Code / Antigravity 스킬 — 키워드만 말하면 네이버 블로그를 자동으로 수집합니다.

## 설치

`SKILL.md` 파일을 Claude Code 스킬 폴더에 복사하세요:

```bash
mkdir -p ~/.claude/skills/blog-collector
cp SKILL.md ~/.claude/skills/blog-collector/
```

## API 키 설정 (최초 1회)

1. [네이버 개발자센터](https://developers.naver.com/apps) 가입
2. 애플리케이션 등록 → **검색 API** 선택
3. Client ID와 Client Secret 발급

```bash
echo "NAVER_CLIENT_ID=여기에_키_입력" >> ~/.env
echo "NAVER_CLIENT_SECRET=여기에_키_입력" >> ~/.env
```

## 사용법

Claude Code에서 자연어로 말하면 됩니다:

```
"맛집 추천 블로그 모아줘"
"서울 카페, 부산 맛집 블로그 5건씩"
"파이썬 블로그 20건 관련순으로"
"제주 여행 블로그 수집해서 분석해줘"
```

## 수집 파이프라인

```
네이버 검색 API → 블로그 검색 (제목/URL/snippet)
  ↓
모바일 URL GET → m.blog.naver.com에서 본문 추출 (가장 빠름)
  ↓ 실패 시
Playwright (stealth) → JS 렌더링 + iframe 접근
  ↓ 실패 시
httpx → PC URL 직접 추출 (최후 보루)
```

## 결과

바탕화면 `블로그수집` 폴더에 자동 저장:

- `blog_키워드_날짜.csv` — Excel에서 바로 열 수 있는 CSV
- `blog_키워드_날짜.txt` — 전체 본문 포함 텍스트
- `blog_키워드_날짜.json` — Claude 분석용 JSON

## 분석 기능

수집 후 "분석해줘"라고 하면 Claude가 직접:
- 핵심 요약
- 트렌드 분석
- 감성 분류 (긍정/부정/중립)
- 인사이트 도출

## 호환성

- Claude Code (Anthropic)
- Antigravity
- WSL / macOS / Linux

## 라이선스

MIT
