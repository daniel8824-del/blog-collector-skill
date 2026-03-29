---
name: blog-collector
description: Naver Blog collection and analysis. Searches Korean blogs via Naver API, extracts full posts with Playwright stealth (iframe/mobile), cleans blog noise, saves CSV/TXT. Triggers on "블로그 수집", "블로그 검색", "네이버 블로그", "nblog", "블로그 모아줘", "blog collect".
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
  └── 네이버 API 키 설정 → ~/.env

Phase 1: INTAKE (정보 수집)
  └── 사용자 요청에서 키워드/건수/정렬 자연스럽게 파악

Phase 2: COLLECT (블로그 수집)
  ├── PROJECT_DIR 생성
  ├── blog_collect.py 작성 (아래 코드)
  └── python3 blog_collect.py 실행 → CSV + TXT 저장

Phase 3: ANALYZE (선택적 분석)
  └── CSV/TXT를 Read 도구로 읽고 Claude가 직접 분석
```

> **중요:** 매 작업 시작 시 **키워드+날짜 기반 고유 폴더**를 생성합니다.
> ```bash
> PROJECT_DIR="/mnt/c/Users/daniel/Desktop/블로그수집/[keyword_slug]_$(date +%Y%m%d_%H%M)"
> mkdir -p "$PROJECT_DIR"
> ```

---

# Phase 0: SETUP (최초 1회)

API 키가 `~/.env`에 있는지 확인합니다. 없으면 사용자에게 안내:

```
네이버 검색 API 키가 필요합니다:
  발급: https://developers.naver.com/apps
  → 애플리케이션 등록 → 검색 API 선택

  Client ID와 Client Secret을 알려주세요.
```

키를 받으면 저장:
```bash
echo "NAVER_CLIENT_ID=받은키" >> ~/.env
echo "NAVER_CLIENT_SECRET=받은키" >> ~/.env
```

---

# Phase 1: INTAKE

사용자 요청을 자연스럽게 파악하여 파라미터로 변환합니다.

| 사용자 표현 | keywords | count | 기타 |
|------------|----------|-------|------|
| "맛집 추천 블로그 모아줘" | 맛집 추천 | 10 | |
| "서울 카페, 부산 맛집 5건씩" | 서울 카페,부산 맛집 | 5 | |
| "파이썬 블로그 20건 관련순" | 파이썬 | 20 | relevance |
| "제주 여행 블로그 분석해줘" | 제주 여행 | 10 | → Phase 3 |

기본값: 10건, 최신순

---

# Phase 2: COLLECT

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

## 실행

`$PROJECT_DIR/blog_collect.py`로 아래 코드를 저장하고 실행합니다.

```bash
# 의존성 (최초 1회)
pip install -q httpx beautifulsoup4 lxml lxml-html-clean rich python-dotenv openpyxl playwright playwright-stealth 2>/dev/null
playwright install chromium 2>/dev/null

# 실행
cd "$PROJECT_DIR" && python3 blog_collect.py --keywords "맛집 추천" --count 10
```

## blog_collect.py

```python
#!/usr/bin/env python3
"""네이버 블로그 수집기 - 검색 API + 모바일GET/Playwright 본문 추출 + CSV 저장"""

import argparse, asyncio, csv, json, os, random, re, sys, time
from dataclasses import dataclass, field
from datetime import datetime
from html import unescape
from pathlib import Path
from urllib.parse import urlparse, parse_qs, urljoin

import httpx
from bs4 import BeautifulSoup
from dotenv import load_dotenv
from rich.console import Console
from rich.panel import Panel
from rich.progress import Progress, SpinnerColumn, BarColumn, TextColumn, MofNCompleteColumn

console = Console()
MOBILE_BASE = "https://m.blog.naver.com"

# ══════════════════════════════════════════════════════════════
# 검색 (네이버 블로그 검색 API)
# ══════════════════════════════════════════════════════════════

@dataclass
class BlogResult:
    title: str; link: str; description: str = ""; bloggerName: str = ""
    bloggerLink: str = ""; postdate: str = ""

def _strip_html(text):
    return unescape(re.sub(r"<[^>]+>", "", text))

def _normalize_url(url):
    parsed = urlparse(url)
    if parsed.netloc == "m.blog.naver.com":
        if "PostView" in parsed.path:
            qs = parse_qs(parsed.query)
            bid, lno = qs.get("blogId",[""])[0], qs.get("logNo",[""])[0]
            if bid and lno: return f"https://blog.naver.com/{bid}/{lno}"
        return url.replace("m.blog.naver.com", "blog.naver.com")
    return url

def search_blogs(query, count=100, sort="date"):
    client_id = os.getenv("NAVER_CLIENT_ID", "")
    client_secret = os.getenv("NAVER_CLIENT_SECRET", "")
    if not client_id or not client_secret:
        console.print("[red]네이버 API 키 미설정. ~/.env에 NAVER_CLIENT_ID, NAVER_CLIENT_SECRET 추가하세요.[/red]")
        console.print("발급: https://developers.naver.com/apps → 검색 API 선택")
        sys.exit(1)
    headers = {"X-Naver-Client-Id": client_id, "X-Naver-Client-Secret": client_secret}
    results, seen = [], set()
    remaining, start = min(count, 1000), 1
    while remaining > 0:
        per = min(remaining, 100)
        resp = httpx.get("https://openapi.naver.com/v1/search/blog",
            headers=headers, params={"query": query, "display": str(per), "start": str(start), "sort": sort}, timeout=15.0)
        resp.raise_for_status(); data = resp.json()
        items = data.get("items", [])
        if not items: break
        for item in items:
            link = _normalize_url(item.get("link", ""))
            if link in seen: continue
            seen.add(link)
            results.append(BlogResult(title=_strip_html(item.get("title","")), link=link,
                description=_strip_html(item.get("description","")), bloggerName=item.get("bloggername",""),
                bloggerLink=item.get("bloggerlink",""), postdate=item.get("postdate","")))
        start += per; remaining -= per
        if start > data.get("total", 0) or len(items) < per: break
    return results[:count]

# ══════════════════════════════════════════════════════════════
# 클리닝 (네이버 블로그 노이즈 제거)
# ══════════════════════════════════════════════════════════════

def clean_blog_body(text):
    if not text or len(text) < 30: return text or ""
    t = text
    # script/style 제거
    t = re.sub(r"<script\b[^<]*(?:(?!</script>)<[^<]*)*</script>", "", t, flags=re.I)
    t = re.sub(r"<style\b[^<]*(?:(?!</style>)<[^<]*)*</style>", "", t, flags=re.I)
    # HTML 태그/엔티티
    t = re.sub(r"<br\s*/?>", "\n", t, flags=re.I)
    t = re.sub(r"</?(?:p|div|li|h[1-6]|blockquote|section|article)[^>]*>", "\n", t, flags=re.I)
    t = re.sub(r"<[^>]+>", "", t); t = unescape(t)
    t = re.sub(r"&[a-zA-Z]+;|&#\d+;", " ", t); t = re.sub(r"\xa0", " ", t)
    # 이모지
    t = re.sub(r"[\U0001F300-\U0001F9FF\U0001FA00-\U0001FAFF\u2600-\u27BF\uFE00-\uFE0F\u200D\u20E3]", "", t)
    # 본문 시작점 (네이버 블로그 상단 노이즈)
    for pat in [r"공유하기\s*신고하기", r"URL\s*복사\s*이웃추가", r"본문\s*기타\s*기능"]:
        m = re.search(pat, t)
        if m and m.start() < len(t) * 0.3: t = t[m.end():]
    # 라인 필터링
    filtered = []
    for line in t.split("\n"):
        line = line.strip()
        if not line: filtered.append(""); continue
        # 해시태그 라인
        if "#" in line:
            total = len(line.replace(" ","")); hc = line.count("#")
            if total > 0 and hc/total >= 0.05 and re.search(r"#[가-힣a-zA-Z0-9_]+", line): continue
        # 광고/프로모션
        if re.search(r"(체험단|원고료|소정의\s*원고|광고\s*포함|제공\s*받아|협찬)", line): continue
        # 블로그 위젯/UI
        if re.match(r"^(이웃추가|팬하기|블로그 홈|댓글\s*\d*|공감\s*\d*|좋아요\s*\d*|공유하기|구독하기)$", line): continue
        # 공유 버튼
        if re.match(r"^(페이스북|트위터|카카오스토리|밴드|네이버|URL\s*복사)\s*$", line): continue
        # 카테고리/네비
        if re.match(r"^(이전글|다음글|목록으로|맨\s*위로|TOP)$", line): continue
        if re.match(r"^태그\s*:", line): continue
        # 댓글 영역
        if re.search(r"(댓글을\s*입력|댓글\s*등록|비밀\s*댓글)", line): continue
        # 저작권
        if re.search(r"(저작권자|무단\s*(전재|복제|배포)|All Rights Reserved)", line, re.I): continue
        # 특수기호/숫자만
        if re.match(r"^[▶▷●◆■★※▲▼→←↑↓♥♡✓✔☞◎·\-=_~.\s]+$", line): continue
        if re.match(r"^\d+\.?\s*$", line): continue
        if len(line) <= 2 and not re.search(r"[가-힣]", line): continue
        filtered.append(line)
    # 끝부분 노이즈 제거
    while filtered and _is_tail_noise(filtered[-1]): filtered.pop()
    t = "\n".join(filtered)
    t = re.sub(r"https?://[^\s)]+", "", t)
    t = re.sub(r"[▶▷●◆■★※▲▼→←↑↓#♥♡✓✔☞◎|│┃]", "", t)
    t = re.sub(r"\*\*|={4,}|-{4,}|```", "", t)
    t = re.sub(r"\[[^\]]+\]\([^\)]+\)|!\[.*?\]\(.*?\)", "", t)
    t = re.sub(r"[\u200b-\u200f\ufeff\u00a0]", "", t)
    t = re.sub(r"\n{2,}", "\n\n", t); t = re.sub(r" {2,}", " ", t)
    return t.strip() if len(t.strip()) >= 20 else ""

def _is_tail_noise(line):
    if not line or not line.strip(): return True
    l = line.strip()
    if re.match(r"^(공감|좋아요|댓글)\s*\d*$", l): return True
    if re.match(r"^(페이스북|트위터|카카오|밴드|구독하기|이웃추가|팬하기)$", l): return True
    if re.match(r"^\d+\.?\s*$", l): return True
    if re.match(r"^[▶▷●◆■★※▲▼→←↑↓♥♡✓✔☞◎·\-=_~.\s]+$", l): return True
    return False

# ══════════════════════════════════════════════════════════════
# 본문 추출 (모바일GET → Playwright → httpx)
# ══════════════════════════════════════════════════════════════

def _to_mobile_url(url):
    parsed = urlparse(url)
    host = parsed.netloc.replace("www.","")
    if host == "m.blog.naver.com": return url
    if host == "blog.naver.com": return f"{MOBILE_BASE}{parsed.path}"
    if "PostView" in parsed.path or "logNo" in (parsed.query or ""):
        qs = parse_qs(parsed.query)
        bid, lno = qs.get("blogId",[""])[0], qs.get("logNo",[""])[0]
        if bid and lno: return f"{MOBILE_BASE}/{bid}/{lno}"
    return url

def _parse_blog_html(html, url):
    soup = BeautifulSoup(html, "lxml")
    for tag in soup(["script","style","iframe","noscript"]): tag.decompose()
    # 제목
    title = ""
    og = soup.find("meta", property="og:title")
    if og and og.get("content"): title = og["content"]
    if not title:
        for sel in [".se-title-text",".tit_h3",".pcol1","h3.tit_view"]:
            el = soup.select_one(sel)
            if el: title = el.get_text(strip=True); break
    if not title:
        tt = soup.find("title"); title = tt.get_text(strip=True) if tt else ""
        title = re.sub(r"\s*[:\-|]\s*네이버\s*블로그\s*$", "", title)
    # 썸네일
    thumb = ""
    oi = soup.find("meta", property="og:image")
    if oi and oi.get("content","").startswith("http"): thumb = oi["content"]
    # 블로거명
    blogger = ""
    for sel in [".nickname",".blog_nickname",".nick"]:
        el = soup.select_one(sel)
        if el: blogger = el.get_text(strip=True); break
    if not blogger:
        og_sn = soup.find("meta", property="og:site_name")
        if og_sn and og_sn.get("content"): blogger = og_sn["content"]
    # 작성일
    postdate = ""
    for sel in [".se_publishDate",".blog_date",".date",".post_date"]:
        el = soup.select_one(sel)
        if el: postdate = el.get_text(strip=True); break
    # 본문 (SE3 → SE2 → 모바일 → article)
    content = ""
    se_main = soup.select_one(".se-main-container")
    if se_main:
        parts = []
        for comp in se_main.select("div.se-component"):
            for t in comp.select(".se-text-paragraph,.se-quote-text"):
                txt = t.get_text(strip=True)
                if txt: parts.append(txt)
        if parts: content = "\n\n".join(parts)
    if len(content) < 50:
        for sel in ["#postViewArea","#post-view",".se_component_wrap",".post-view"]:
            el = soup.select_one(sel)
            if el:
                ps = [p.get_text(strip=True) for p in el.find_all("p") if len(p.get_text(strip=True)) > 1]
                if ps: content = "\n\n".join(ps)
                if len(content) < 50: content = el.get_text(separator="\n", strip=True)
                if len(content) >= 50: break
    if len(content) < 50:
        for sel in [".se_textarea",".post_ct",".sect_dsc","div.__se_component_area"]:
            el = soup.select_one(sel)
            if el: content = el.get_text(separator="\n", strip=True)
            if len(content) >= 50: break
    if len(content) < 50:
        for tag in ["article","main"]:
            el = soup.find(tag)
            if el: content = el.get_text(separator="\n", strip=True)
            if len(content) >= 50: break
    content = clean_blog_body(content)
    return title, content, thumb, blogger, postdate

async def extract_mobile_get(url):
    mobile = _to_mobile_url(url)
    try:
        async with httpx.AsyncClient(follow_redirects=True, timeout=60.0, headers={
            "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 Safari/604.1",
            "Referer": "https://section.blog.naver.com/", "Accept-Language": "ko-KR,ko;q=0.9"}) as c:
            resp = await c.get(mobile); resp.raise_for_status(); html = resp.text
        title, content, thumb, blogger, postdate = _parse_blog_html(html, mobile)
        ok = len(content) >= 100
        return {"title": title, "content": content, "length": len(content), "method": "mobile-get",
                "thumb": thumb, "blogger": blogger, "postdate": postdate, "success": ok}
    except Exception as e:
        return {"content": "", "length": 0, "method": "mobile-get", "success": False, "error": str(e)[:200]}

async def extract_playwright(url):
    try:
        from playwright.async_api import async_playwright; from playwright_stealth import Stealth
    except ImportError:
        return {"content": "", "length": 0, "method": "playwright", "success": False, "error": "playwright 미설치"}
    mobile = _to_mobile_url(url)
    try:
        async with async_playwright() as p:
            br = await p.chromium.launch(headless=True, args=["--no-sandbox","--disable-dev-shm-usage","--disable-gpu"])
            ctx = await br.new_context(viewport={"width":412,"height":915},
                user_agent="Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X) AppleWebKit/605.1.15 Safari/604.1",
                locale="ko-KR", timezone_id="Asia/Seoul")
            page = await ctx.new_page(); await Stealth().apply_stealth_async(page)
            await page.route("**/*.{png,jpg,jpeg,gif,svg,woff,woff2,ttf,eot}", lambda r: r.abort())
            try:
                await page.goto(mobile, wait_until="domcontentloaded", timeout=60000)
                await page.wait_for_timeout(3000)
                try: await page.wait_for_selector(".se-main-container,#postViewArea,.post-view", timeout=5000)
                except: pass
            except: pass
            html = await page.content(); await br.close()
        title, content, thumb, blogger, postdate = _parse_blog_html(html, mobile)
        ok = len(content) >= 50
        return {"title": title, "content": content, "length": len(content), "method": "playwright",
                "thumb": thumb, "blogger": blogger, "postdate": postdate, "success": ok}
    except Exception as e:
        return {"content": "", "length": 0, "method": "playwright", "success": False, "error": str(e)[:200]}

async def extract_blog(url):
    r1 = await extract_mobile_get(url)
    if r1.get("success") and r1.get("length", 0) >= 100: return r1
    r2 = await extract_playwright(url)
    if r2.get("success") and r2.get("length", 0) > r1.get("length", 0): return r2
    return r1 if r1.get("length", 0) >= r2.get("length", 0) else r2

# ══════════════════════════════════════════════════════════════
# 메인
# ══════════════════════════════════════════════════════════════

def main():
    load_dotenv(); load_dotenv(Path.home()/".env")
    ap = argparse.ArgumentParser(description="네이버 블로그 수집기")
    ap.add_argument("--keywords", required=True); ap.add_argument("--count", type=int, default=10)
    ap.add_argument("--sort", default="date", choices=["date","sim"], help="date=최신순, sim=관련순")
    ap.add_argument("--fast", action="store_true", help="본문 추출 생략"); ap.add_argument("--output", default=None)
    args = ap.parse_args()

    queries = [q.strip() for q in args.keywords.split(",") if q.strip()]
    console.print(f"\n[bold blue]네이버 블로그 검색[/bold blue] {len(queries)}개: {', '.join(queries)}")
    console.print(f"  키워드당 {args.count}건 / 정렬: {'관련순' if args.sort=='sim' else '최신순'}", style="dim")

    # 1. 검색
    results, seen = [], set()
    for qi, query in enumerate(queries, 1):
        if len(queries) > 1: console.print(f"\n  [cyan][{qi}/{len(queries)}][/cyan] '{query}' 검색 중...")
        hits = search_blogs(query, args.count, args.sort)
        for r in hits:
            if r.link not in seen: seen.add(r.link); results.append((query, r))
        if len(queries) > 1: console.print(f"    [green]{len(hits)}건[/green]")
    if not results: console.print("[yellow]검색 결과 없음[/yellow]"); return
    console.print(f"\n  [green]총 {len(results)}건[/green]")

    # 2. 본문 추출
    articles = []
    for i, (kw, r) in enumerate(results):
        articles.append({"keyword": kw, "title": r.title, "url": r.link, "bloggerName": r.bloggerName,
            "postdate": r.postdate, "description": r.description, "content": "", "content_length": 0, "method": "", "success": True})

    if not args.fast:
        total = min(len(articles), 200)
        console.print(f"\n  [cyan]{total}건 본문 추출 중...[/cyan]")
        ok_count = 0
        with Progress(SpinnerColumn(), TextColumn("[progress.description]{task.description}"),
                       BarColumn(), MofNCompleteColumn(), console=console) as progress:
            task = progress.add_task("본문 추출", total=total)
            for i in range(total):
                ext = asyncio.run(extract_blog(articles[i]["url"]))
                if ext.get("success"):
                    articles[i].update(content=ext["content"], content_length=ext["length"], method=ext["method"])
                    if ext.get("blogger"): articles[i]["bloggerName"] = ext["blogger"]
                    if ext.get("postdate"): articles[i]["postdate"] = ext["postdate"]
                    ok_count += 1
                else:
                    articles[i].update(success=False, content=articles[i]["description"],
                                       content_length=len(articles[i]["description"]))
                progress.advance(task)
                time.sleep(random.uniform(1, 2))  # 네이버 rate limit 방지
        console.print(f"  [green]{ok_count}/{total}건 추출 성공[/green]")

    # 3. CSV 저장
    slug = queries[0].replace(" ","_")[:20]; ts = datetime.now().strftime("%Y%m%d_%H%M")
    fn_csv = args.output or f"blog_{slug}_{ts}.csv"
    fn_txt = fn_csv.rsplit(".",1)[0] + ".txt"

    with open(fn_csv, "w", encoding="utf-8-sig", newline="") as f:
        w = csv.writer(f); w.writerow(["키워드","타이틀","본문","블로거","링크","날짜","URL"])
        for a in articles:
            w.writerow([a.get("keyword",""), a.get("title",""), a.get("content",""),
                        a.get("bloggerName",""), "", a.get("postdate",""), a.get("url","")])
    console.print(f"[green]CSV 저장: {fn_csv}[/green]")

    # TXT 저장
    lines = [f"네이버 블로그 수집 결과: {', '.join(queries)}", f"수집일시: {datetime.now().strftime('%Y-%m-%d %H:%M')}",
             f"총 {len(articles)}건", "="*70]
    for i, a in enumerate(articles, 1):
        lines += [f"\n[{i}] {a.get('title','')}", f"    블로거: {a.get('bloggerName','')}",
                  f"    날짜:   {a.get('postdate','')}", f"    URL:    {a.get('url','')}",
                  f"    글자수: {a.get('content_length',0):,}자", "-"*70, a.get("content",""), "="*70]
    Path(fn_txt).write_text("\n".join(lines)+"\n", encoding="utf-8")
    console.print(f"[green]TXT 저장: {fn_txt}[/green]")

    # 결과 출력
    ok = sum(1 for a in articles if a.get("success")); fail = len(articles) - ok
    console.print(Panel(f"[bold]검색어:[/bold] {', '.join(queries)}\n[bold]결과:[/bold] {len(articles)}건  "
        f"[green]성공 {ok}[/green]  [red]실패 {fail}[/red]\n[bold]저장:[/bold] {fn_csv}",
        title="[bold blue]블로그 수집 결과[/bold blue]", border_style="blue"))
    for i, a in enumerate(articles[:10], 1):
        s = "[green]OK[/green]" if a.get("success") else "[red]FAIL[/red]"
        console.print(f"\n  {s} {i}. {a.get('title','')} ({a.get('bloggerName','')})")
        console.print(f"     {a.get('url','')}")
        if a.get("content"):
            console.print(f"     [dim]{a['content'][:120]}...[/dim]")
            console.print(f"     [cyan]{a.get('content_length',0):,}자[/cyan]")

if __name__ == "__main__": main()
```

---

# Phase 3: ANALYZE (선택적)

사용자가 "분석해줘", "요약해줘" 등을 요청하면 실행합니다.

1. Read 도구로 JSON 파일 읽기
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
| 네이버 API 키 미설정 | `~/.env`에 NAVER_CLIENT_ID, NAVER_CLIENT_SECRET 추가 |
| playwright 미설치 | `playwright install chromium` |
| 429 rate limit | 건수 줄이기 또는 잠시 후 재시도 |
| 본문 추출 실패 | 모바일GET → Playwright → httpx 3단계 자동 폴백 |
| CSV 0건 | 키워드 변경 또는 건수 확대 |
