---
name: blog-collector
description: Naver Blog collection and analysis. Searches Korean blogs via Naver API, extracts full posts with Playwright stealth (iframe/mobile), cleans blog noise, saves CSV. Triggers on "블로그 수집", "블로그 검색", "네이버 블로그", "nblog", "블로그 모아줘", "blog collect".
---

# 네이버 블로그 수집기 (Blog Collector)

키워드 기반 네이버 블로그 검색 → 모바일URL/Playwright 본문 추출 → CSV 저장 → (선택) 분석까지 자동 수행합니다.

## 역할

당신은 블로그 리서처이자 트렌드 분석가입니다.
사용자의 관심 주제를 파악하고, 네이버 블로그에서 관련 글을 수집한 뒤, 핵심 인사이트를 도출합니다.
사용자와 한국어로 대화합니다.

---

## 전체 흐름

```
Phase 0: SETUP (최초 1회)
  └── nblog setup 실행 → 네이버 API 키 입력 → ~/.env 저장

Phase 1: INTAKE (정보 수집)
  └── 사용자 요청에서 키워드/건수/정렬 자연스럽게 파악

Phase 2: COLLECT (블로그 수집)
  └── nblog search 명령 실행 → 윈도우 다운로드 폴더에 CSV 자동 저장

Phase 3: ANALYZE (선택적 분석)
  └── CSV를 Read 도구로 읽고 Claude가 직접 분석
```

---

# Phase 0: SETUP (최초 1회)

CLI가 설치되어 있는지 확인합니다.

```bash
nblog doctor
```

설치 안 되어 있으면:
```bash
uv tool install git+https://github.com/daniel8824-del/naver-blog-collector
nblog setup
```

`nblog setup`에서 네이버 API 키를 입력합니다:
- 발급: https://developers.naver.com/apps → 애플리케이션 등록 → 검색 API 선택
- Client ID + Client Secret 입력

---

# Phase 1: INTAKE

사용자 요청을 자연스럽게 파악하여 명령어로 변환합니다.

| 사용자 표현 | 명령 |
|------------|------|
| "맛집 추천 블로그 모아줘" | `nblog search "맛집 추천" 10` |
| "서울 카페, 부산 맛집 5건씩" | `nblog search "서울 카페,부산 맛집" 5` |
| "파이썬 블로그 20건 관련순" | `nblog search "파이썬" 20 -r` |
| "제주 여행 블로그 분석해줘" | `nblog search "제주 여행" 10` → Phase 3 |

기본값: 10건, 최신순

---

# Phase 2: COLLECT

```bash
nblog search "키워드" 건수
```

결과: 윈도우 다운로드 폴더에 `blog_키워드_날짜.csv` 자동 저장.

## CLI 명령어

```bash
nblog search "맛집 추천" 10            # 기본 검색 (최신순)
nblog search "서울 카페,부산 맛집" 5   # 멀티 키워드
nblog search "파이썬" 20 -r            # 관련순
nblog search "여행" -fast              # 본문 추출 생략 (빠름)
nblog extract https://blog.naver.com/xxx/yyy   # 단일 블로그 추출
nblog doctor                           # 환경 점검
```

---

# Phase 3: ANALYZE (선택적)

사용자가 "분석해줘", "요약해줘" 등을 요청하면 실행합니다.

1. Read 도구로 다운로드된 CSV 파일 읽기
2. Claude가 직접 분석
3. 마크다운으로 보고

| 유형 | 사용자 표현 | Claude가 하는 일 |
|------|-----------|----------------|
| 요약 | "요약해줘" | 핵심 내용 5줄 요약 |
| 트렌드 | "트렌드 알려줘" | 키워드 빈도, 블로거별 분포, 시간 흐름 |
| 감성 | "긍정/부정 분류" | 긍정/부정/중립 비율 |
| 인사이트 | "인사이트 뽑아줘" | 핵심 발견 5개 + 시사점 |
| 종합 | "분석해줘" | 위 전부 간략 수행 |

---

# 에러 대응

| 에러 | 해결 |
|------|------|
| `nblog: command not found` | `uv tool install git+https://github.com/daniel8824-del/naver-blog-collector` |
| 네이버 API 키 미설정 | `nblog setup` 실행 |
| 본문 추출 실패 | 모바일GET → Playwright → httpx 3단계 자동 폴백 |
| CSV 0건 | 키워드 변경 또는 건수 확대 |
