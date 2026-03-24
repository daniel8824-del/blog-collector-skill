# Blog Collector Skill

AI가 키워드만으로 네이버 블로그를 자동 수집하고 본문까지 추출합니다.

[![Python 3.10+](https://img.shields.io/badge/Python-3.10%2B-blue)](https://www.python.org/)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet)](https://claude.com/claude-code)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## Features

- 키워드만 말하면 **네이버 블로그 자동 수집** (멀티 키워드 지원)
- 네이버 검색 API 연동 (최대 1,000건)
- **3단계 본문 추출**: 모바일GET → Playwright stealth → httpx 폴백
- 네이버 블로그 전용 노이즈 클리닝 (해시태그, 광고, 위젯, 공유버튼 자동 제거)
- 최종 결과물: **CSV + TXT + JSON** (Excel 바로 열기 가능)
- Claude가 직접 분석 (요약, 트렌드, 감성, 인사이트)

## 사용 방법

### 1. Claude Code 스킬

```bash
git clone https://github.com/daniel8824-del/blog-collector-skill.git ~/.claude/skills/blog-collector
```

Claude Code에서:
```
"맛집 추천 블로그 모아줘"
"서울 카페, 부산 맛집 블로그 5건씩"
"제주 여행 블로그 수집해서 분석해줘"
```

### 2. Google Antigravity

```bash
git clone https://github.com/daniel8824-del/blog-collector-skill.git
cd blog-collector-skill
```

`.agent/` 폴더가 자동 인식됩니다.
Antigravity에서 "네이버 블로그 모아줘"라고 입력하면 스킬이 실행됩니다.

## API Keys

`~/.env` 파일에 설정:
```
NAVER_CLIENT_ID=필수 (네이버 검색 API)
NAVER_CLIENT_SECRET=필수 (네이버 검색 API)
```

API 키 발급: https://developers.naver.com/apps → 애플리케이션 등록 → 검색 API 선택

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

## 분석 기능

수집 후 "분석해줘"라고 하면 Claude가 직접:
- 핵심 요약 (5줄)
- 트렌드 분석 (키워드 빈도, 블로거별 분포)
- 감성 분류 (긍정/부정/중립 비율)
- 인사이트 도출 (핵심 발견 5개)

## License

MIT
