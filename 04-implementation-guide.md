# 04. Step-by-Step 구현 가이드

> **총 예상 소요**: 4~5일 (반나절씩)  
> **순서 중요**: Phase 0 → 1 → 2 → 3 → 4 → 5 → 6

---

## Phase 0: 사전 준비

### 0-1. 라이선스 확인 

---

### 0-2. 네이버 Open API 키 발급 (10분)

1. [developers.naver.com](https://developers.naver.com) → **내 애플리케이션 → 애플리케이션 등록**
2. **애플리케이션 이름**: `BNK 뉴스 수집`
3. **사용 API 체크**: `검색` → 뉴스 항목 선택
4. **환경 추가**: `WEB` → 서비스 URL: 실제 도메인 혹은 우선 Local host로 등록
5. 등록 완료 → **Client ID**, **Client Secret** 복사해 안전한 곳에 저장

---

### 0-3. 솔루션 생성 및 환경 변수 설정 (10분)

> **중요**: 환경 변수는 솔루션 안에 만들어야 이후 Flow와 Custom Connector를 같은 솔루션에 포함시켜 환경 변수를 참조할 수 있음.

1. [make.powerapps.com](https://make.powerapps.com) → **Solutions** → **+ New solution**
   - 이름: `BNK Daily News Agent`
   - Publisher: 조직 기본값 선택
2. 솔루션 내 → **+ New → More → Environment variable**
3. 아래 1개 생성 (Client Secret은 Custom Connector 안에 직접 입력):

| 표시 이름 | 이름 | 데이터 유형 | 현재 값 |
|---|---|---|---|
| Naver Client ID | `NaverClientId` | 텍스트 | 발급받은 Client ID |

> ⚠️ 이후 만드는 Flow와 Custom Connector는 반드시 이 솔루션 내에서 생성해야 환경 변수 참조 가능

---

## Phase 1: Dataverse 준비

### 1-1. 솔루션 내 테이블 생성 — Copilot 프롬프트 사용

**접속**: [make.powerapps.com](https://make.powerapps.com) → 솔루션 `BNK Daily News Agent` 진입 → **+ New → Table → Table (설명으로 생성)**  
또는 좌측 하단 Copilot 채팅창에 아래 프롬프트 붙여넣기:

```
Create 3 Dataverse tables with the following structure.

Table 1: Keyword_Master
Columns:
- department (Choice, required): options are auto, retail, industrial, synergy, general
- keyword (Single line text, max 100, required)
- max_articles (Whole number, default 3)
- is_active (Yes/No, default Yes)

Table 2: News_RawLog
Columns:
- article_title (Single line text, max 500, required)
- article_url (Single line text, max 500)
- description (Multiple lines of text, max 2000)
- published_at (Date and time, required)
- search_keyword (Single line text, max 100, required)
- department (Choice, required): same options as Keyword_Master — auto, retail, industrial, synergy, general
- collected_at (Date and time, required)

Table 3: News_Summary
Columns:
- report_date (Date only, required)
- department (Choice, required): same options — auto, retail, industrial, synergy, general
- article_count (Whole number, default 0)
- summary_content (Multiple lines of text, max 10000)
- created_at (Date and time)
- is_sent (Yes/No, default No)

For the department Choice column in all 3 tables, please use the same global choice set so they share the same option values.
```

> 테이블 간 관계(Relationship)는 설정하지 않는다.

---

### 1-2. Keyword_Master 목업 데이터 입력

**접속**: Tables → `Keyword_Master` → **Edit data** → 기존 행 모두 삭제 후 아래 16개 입력

| department | keyword | max_articles | is_active |
|---|---|---|---|
| auto | 신차할부 | 3 | true |
| auto | 중고차금융 | 3 | true |
| auto | 전기차금융 | 3 | true |
| auto | 자동차리스 | 3 | true |
| retail | 개인신용대출 | 3 | true |
| retail | 가계대출 | 3 | true |
| retail | 소비자금융 | 3 | true |
| retail | 가계부채 | 3 | true |
| industrial | 중소기업대출 | 3 | true |
| industrial | 부동산PF | 3 | true |
| industrial | 제조업경기 | 3 | true |
| industrial | 기업금융동향 | 3 | true |
| synergy | BNK금융지주 | 5 | true |
| synergy | 캐피탈업계 | 3 | true |
| synergy | 금융규제 | 3 | true |
| synergy | 지역금융 | 3 | true |

또는 `data/keyword-master.csv` 파일을 **Import → Import from Excel/CSV** 로 일괄 업로드

> 입력 후 `is_active` 컬럼 값이 모두 **Yes** 인지 한 번 더 확인

---

## Phase 2: Custom Connector 만들기

### 2-1. 네이버 뉴스 검색 Custom Connector 생성

**접속**: 솔루션 `BNK Daily News Agent` → **+ New → Automation → Custom connector**

**1단계 General**

| 항목 | 값 |
|---|---|
| Connector name | `NaverNewsSearch` |
| Description | 네이버 뉴스 검색 Open API |
| Scheme | HTTPS |
| Host | `openapi.naver.com` |
| Base URL | `/v1/search` |

**2단계 Security**

| 항목 | 값 |
|---|---|
| Authentication type | API Key |
| Parameter label | X-Naver-Client-Secret |
| Parameter name | `X-Naver-Client-Secret` |
| Parameter location | Header |

> Client ID는 각 호출 시 헤더로 직접 전달 (Flow에서 환경 변수로 주입)

**3단계 Definition → + New action**

| 항목 | 값 |
|---|---|
| Summary | Search News |
| Operation ID | `searchNews` |
| Verb | GET |
| URL | `/news.json` |

**Request → + Import from sample**
```
GET https://openapi.naver.com/v1/search/news.json?query=전기차금융&display=3&sort=date
Headers:
X-Naver-Client-Id: {your-client-id}
```

**Response → + Add default response → 아래 JSON 붙여넣기**
```json
{
  "lastBuildDate": "Tue, 07 Apr 2026 13:32:25 +0900",
  "total": 1216776,
  "start": 1,
  "display": 3,
  "items": [
    {
      "title": "[해남소식] '무소유' 법정 스님 삶과 철학 잇는 에세이 공모전",
      "originallink": "https://www.yna.co.kr/view/AKR20260407089300054?input=1195m",
      "link": "https://n.news.naver.com/mnews/article/001/0016007005?sid=103",
      "description": "참여를 희망하는 경우 A4 기준 1장 반 내외분량의 에세이를 작성해 이메일(booksan25@<b>naver</b>.com)로 제출하면 된다. 기간은 오는 10일부터 6월 10일까지이며 국내 거주 내외국인 누구나 응모할 수 있다. 수상작은 심사를 거쳐 6월... ",
      "pubDate": "Tue, 07 Apr 2026 13:28:00 +0900"
    },
    {
      "title": "현대차, 상품성·경제성↑ ‘2027 코나’ 출시",
      "originallink": "http://www.breaknews.com/1198200",
      "link": "http://www.breaknews.com/1198200",
      "description": "break9874@<b>naver</b>.com *아래는 위 기사를 '구글 번역'으로 번역한 영문 기사의 [전문]입니다. 구글번역'은 이해도 높이기를 위해 노력하고 있습니다. 영문 번역에 오류가... ",
      "pubDate": "Tue, 07 Apr 2026 13:08:00 +0900"
    },
    {
      "title": "현대차, 소형 스포츠 옵션 강자..‘2027 코나’ 출시",
      "originallink": "https://kpenews.com/View.aspx?No=4032649",
      "link": "https://kpenews.com/View.aspx?No=4032649",
      "description": "한국정경신문 문상혁 기­자 nasa7457@<b>naver</b>.com [한국정경신문=문상혁 기자] 현대자동차가 경제성을 강화한 소형 SUV를 선보인다. 현대차는 소형 스포츠유틸리티차(SUV) 연식변경 모델 '2027 코나'를 출시한다고 7일... ",
      "pubDate": "Tue, 07 Apr 2026 12:54:00 +0900"
    }
  ]
}
```

**4단계 Test → Create connector → New connection**
- API Key 값에 네이버 Client Secret 입력
- Test operation 실행 → 200 OK 확인

---

## Phase 3: Flow 1 구현 — Daily News Scraper

**목적**: 키워드별 네이버 뉴스 수집 → News_RawLog 저장  
**실행**: 매일 07:00 KST

> **중요**: 이 Flow는 반드시 솔루션 `BNK Daily News Agent` 안에서 생성해야 환경 변수를 참조할 수 있음.

### 3-1. Flow 생성

솔루션 `BNK Daily News Agent` → **+ New → Automation → Cloud flow → Scheduled**
- **이름**: `[BNK] Flow1 - Daily News Scraper`
- **시작**: 내일 오전 7시
- **반복**: 1일
- **시간대**: (UTC+09:00) Seoul

---

### 3-2. Step 1 — Keyword_Master 조회

**Add an action** → `Dataverse` → **List rows**

| 설정 | 값 |
|---|---|
| Table name | Keyword_Master |
| Filter rows | `new_is_active eq true` |
| Select columns | `new_keyword,new_department,new_max_articles` |

> **꼭 열의 Logical name 사용**: 표시 이름(keyword)이 아닌 논리 이름(`new_keyword`)을 입력해야 함.  
> publisher prefix는 환경마다 다를 수 있음 (`new_`, `cr_`, `bnk_` 등) → Tables에서 컬럼 클릭 후 **Advanced options → Logical name** 확인

---

### 3-3. Step 2 — 각 키워드 반복 처리

**Add an action** → `Control` → **Apply to each**
- **Select an output**: `value` (List rows 결과)
- **Settings (상단 ⚙️)** → **Degree of Parallelism**: `3`

> 병렬 3개로 속도 향상 (10이상 시 API 429 오류 위험)

---

### 3-4. Step 3 — 네이버 뉴스 검색 (Apply to each 내부)

**Add an action** → Custom 탭 → **NaverNewsSearch** → **Search News**

| 설정 | 값 |
|---|---|
| query | `@{items('Apply_to_each')?['new_keyword']}` |
| display | `@{items('Apply_to_each')?['new_max_articles']}` |
| sort | `date` |
| X-Naver-Client-Id | Dynamic content → 환경 변수 `NaverClientId` 선택 |

> **Parse JSON 불필요**: Custom Connector는 Response JSON을 자동 파싱하므로 dynamic content에 `title`, `originallink`, `description`, `pubDate` 등이 바로 듸다.

---

### 3-5. Step 4 — 각 기사 처리 (두 번째 Apply to each)

**Add an action** → `Control` → **Apply to each**
- **Select an output**: `items` (Search News 커스텀 커넥터 결과)

> 커스텀 커넥터는 `items` 배열을 자동 인식하므로 dynamic content에서 바로 선택 가능

---

### 3-6. Step 5 — Dataverse에 저장 (Apply to each 1 내부)

**Add an action** → `Dataverse` → **Add a new row**

| Dataverse 컬럼 | 값 |
|---|---|
| Table name | News_RawLog |
| article_title | dynamic content → `title` |
| article_url | dynamic content → `originallink` |
| description | dynamic content → `description` |
| published_at | dynamic content → `pubDate` |
| search_keyword | `@{items('Apply_to_each')?['new_keyword']}` |
| department | `@{items('Apply_to_each')?['new_department']}` |
| collected_at | `@{utcNow()}` |

---

### 3-7. 저장 및 테스트

1. **Save** → **Test → Manually → Run flow**
2. `make.powerapps.com` → Tables → `News_RawLog` → **Edit data** 에서 데이터 확인
3. 예상 결과: display=3&sort=date 기준 최신 3건 저장, `<b>` 태그는 그대로 포함됨

---

## Phase 4: Flow 2 구현 — Daily AI Analyzer

**목적**: 부서별 원본 기사 묶음 → AI Builder 리포트 생성 → News_Summary 저장 → 메일 발송  
**실행**: 매일 08:00 KST

### 4-1. Flow 생성

솔루션 `BNK Daily News Agent` → **+ New → Automation → Cloud flow → Scheduled**
- **이름**: `[BNK] Flow2 - Daily AI Analyzer`
- **시작**: 내일 오전 8시
- **반복**: 1일
- **시간대**: (UTC+09:00) Seoul

---

### 4-2. Step 1 — varReports 변수 초기화

**Add an action** → `Variables` → **Initialize variable**

| 설정 | 값 |
|---|---|
| 이름 | `varReports` |
| 유형 | Array |
| 값 | (비워두기) |

> 부서 루프를 돌면서 HTML 리포트를 수집한 뒤, 루프 종료 후 한 번에 이메일 발송

---

### 4-3. Step 2 — 오늘 수집된 부서 목록 조회 (동적 중복 제거)

**Add an action** → `Dataverse` → **List rows**

| 설정 | 값 |
|---|---|
| Table name | News_RawLog |
| Filter rows | `new_collected_at ge @{startOfDay(utcNow())}` |
| Select columns | **(비워두기)** |

> ⚠️ Select columns를 비워야 `@OData.Community.Display.V1.FormattedValue` 어노테이션이 응답에 포함됨

---

**Add an action** → `Data Operation` → **Select**
- 입력: `value` (List rows 결과)
- Map → Value: `@{item()?['new_department@OData.Community.Display.V1.FormattedValue']}`

> 각 결과는 `{"value": "auto"}` 형태의 object로 반환됨. 이후 `items('Apply_to_each')?['value']`로 string 추출

---

**Add an action** → `Data Operation` → **Compose**
- 입력: `@{union(body('Select'), body('Select'))}`

> `union(array, array)`에 같은 배열을 두 번 넣으면 중복 제거된 distinct 목록 반환

---

### 4-4. Step 3 — 각 부서 반복

**Add an action** → `Control` → **Apply to each**
- **Select an output**: `outputs('Compose')`

> 각 item은 `{"value": "auto"}` object. 이후 단계에서 `items('Apply_to_each')?['value']`로 string "auto" 추출

---

### 4-5. Step 4 — 부서별 기사 조회 + 필터 (Apply to each 내부)

**Add an action** → `Dataverse` → **List rows**

| 설정 | 값 |
|---|---|
| Table name | News_RawLog |
| Filter rows | `new_collected_at ge @{startOfDay(utcNow())}` |
| Row count | `30` |
| Select columns | **(비워두기)** |

---

**Add an action** → `Data Operation` → **Filter array**
- 입력: `value` (위 List rows 결과)
- 조건: `item()?['new_department@OData.Community.Display.V1.FormattedValue']` **is equal to** `items('Apply_to_each')?['value']`

> `items('Apply_to_each')?['value']`는 string "auto" / "retail" 등

---

### 4-6. Step 5 — 기사 텍스트 합치기

**Add an action** → `Data Operation` → **Select**
- 입력: `body('Filter_array')`
- Map → Value: `@{concat('[제목] ', item()?['new_article_title'], '\n[내용] ', item()?['new_description'])}`

**Add an action** → `Data Operation` → **Join**
- 입력: 위 Select 결과
- Join with: `\n\n---\n\n`

---

### 4-7. Step 6 — AI Builder 리포트 생성

**Add an action** → `AI Builder` → **Create text with GPT using a prompt**

| 설정 | 값 |
|---|---|
| `{{articles}}` | Dynamic content → **Join** 결과 |
| `{{department}}` | `@{items('Apply_to_each')?['value']}` |
| `{{today}}` | `@{formatDateTime(utcNow(), 'yyyy-MM-dd')}` |

**AI Builder 출력 경로 (중요)**:
```
body('Run_a_prompt')?['responsev2']?['predictionOutput']?['text']
```

> `body('Run_a_prompt')?['text']` 로 하면 null 반환됨. 위 전체 경로 사용 필수.

---

### 4-8. Step 7 — varReports 배열에 추가

**Add an action** → `Variables` → **Append to array variable**

| 설정 | 값 |
|---|---|
| 이름 | `varReports` |
| 값 | `@{concat('<h2>', items('Apply_to_each')?['value'], ' 전략 인사이트</h2>', body('Run_a_prompt')?['responsev2']?['predictionOutput']?['text'])}` |

---

### 4-9. Step 8 — News_Summary 저장

**Add an action** → `Dataverse` → **Add a new row**

| 컬럼 | 값 |
|---|---|
| Table name | News_Summary |
| 리포트 날짜 (new_report_date) | `@{formatDateTime(utcNow(), 'yyyy-MM-dd')}` |
| 부서 (new_department) | `@{first(body('Filter_array'))?['new_department']}` |
| 기사 수 (new_article_count) | `@{length(body('Filter_array'))}` |
| 요약 전문 (new_summary_content) | `@{body('Run_a_prompt')?['responsev2']?['predictionOutput']?['text']}` |
| 생성일시 (new_created_at) | `@{utcNow()}` |

> ⚠️ `new_department`는 정수 코드로 저장. `first(body('Filter_array'))?['new_department']`로 원본 integer 값 그대로 사용  
> `is_sent` 컬럼 없음 — POC에서는 추적하지 않음

---

### 4-10. Step 9 — [루프 바깥] 메일 발송

> Apply to each 루프가 완전히 끝난 뒤 아래 2개 액션 추가

**Add an action** → `Data Operation` → **Join**
- 입력: `variables('varReports')`
- Join with: `<hr style="margin:30px 0;">`

---

**Add an action** → `Office 365 Outlook` → **Send an email (V2)**

| 설정 | 값 |
|---|---|
| To | 수신자 이메일 (POC: 본인 이메일) |
| Subject | `@{formatDateTime(utcNow(), 'yyyy-MM-dd')} BNK 전략 인사이트 일일 브리핑` |
| Body | `@{body('Join')}` |
| Is HTML | **Yes** (토글 켜기) |

> ⚠️ Body에 `<pre>` 태그 절대 사용 금지 — HTML이 텍스트로 그대로 출력됨

---

## Phase 5: 통합 테스트

### 테스트 시나리오

```
□ 1. Flow 1 수동 실행 → News_RawLog 레코드 생성 확인 (최대 120개)
□ 2. News_RawLog 샘플 확인: HTML 태그 없음, URL 정상, 부서 분류 정확
□ 3. Flow 2 수동 실행 → News_Summary 4개 생성 확인
□ 4. AI 리포트 품질 확인: 한국어, McKinsey 스타일, BNK 영향 섹션 존재
□ 5. 이메일 수신 확인 — 4개 부서 리포트 HTML 정상 렌더링 (마크다운 ## ** 보이면 오류)
□ 6. 다음날 자동 실행 까지 대기 → 정상 수신 확인
```

### 자주 발생하는 오류

| 오류 | 원인 | 해결 |
|---|---|---|
| HTTP 401 Unauthorized | 네이버 API 키 오류 | Client ID/Secret 재확인, 앱 등록 상태 확인 |
| HTTP 429 Too Many Requests | API 호출 한도 초과 | 병렬 처리 수 줄이기 (3→1), 키워드 수 축소 |
| AI Builder 크레딧 부족 | credits 소진 | 환경 관리자에게 크레딧 추가 요청 |
| Dataverse 필드 없음 오류 | 컬럼 이름 불일치 | publisher prefix 확인 (e.g. new_, cr_) |
| Filter array 결과 빈값 | Choice FormattedValue 대소문자 불일치 | News_RawLog에서 실제 레코드 클릭 → `new_department@OData.Community.Display.V1.FormattedValue` 실제값 확인 후 Select Map과 일치시킴 |
| 이메일 본문에 `##`, `**` 그대로 노출 | AI Builder가 Markdown 출력 | `02-report-generation.txt` 프롬프트에 HTML 출력 지시 포함 확인 |
| AI Builder 출력 null | 출력 경로 오류 | `body('Run_a_prompt')?['responsev2']?['predictionOutput']?['text']` 전체 경로 사용 |
| Apply to each 내 동적 참조 깨짐 | 중첩 루프 명칭 충돌 | 각 Apply to each에 명확한 이름 부여 |

---

## Phase 6: Copilot Studio 에이전트 구축

**목적**: 구축된 Dataverse 저장소(News_RawLog, News_Summary)를 기반으로 자연어 질의에 답변하는 에이전트 생성  
예) "오늘 오토 부서 뉴스 요약해줘", "지난주 산업 리포트 보여줘", "부동산PF 관련 기사 있어?"

---

### 6-1. 에이전트 생성

**접속**: [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com) → **+ Create** → **New agent**

| 항목 | 값 |
|---|---|
| 이름 | `BNK 전략 인사이트 에이전트` |
| 설명 | 일일 수집된 뉴스를 분석하여 BNK 캐피탈 임원진과 담당자에게 전략적 인사이트를 제공합니다 |
| 언어 | 한국어 |

> Instructions(지시문) 입력창에 아래 내용 붙여넣기:
```
당신은 BNK 캐피탈의 전략 인사이트 에이전트입니다.
매일 수집된 시장 뉴스를 분석하여 임원진과 사업 담당자가 빠르게 전략적 판단을 내릴 수 있도록 돕습니다.
Dataverse에 저장된 News_RawLog(원본 기사)와 News_Summary(AI 생성 전략 리포트)를 기반으로 답변합니다.
- 부서명: auto(오토금융), retail(소매금융), industrial(산업금융), synergy(시너지)
- 답변 시 무엇이 BNK에 의미가 있는지, 어떤 리스크와 기회가 있는지 전략적 관점에서 해석합니다.
- 사실 나열이 아닌 So What 중심으로 답변합니다.
- 오늘 날짜 기준으로 답변하되, 날짜를 명시하면 해당 날짜 기준으로 조회합니다.
```

---

### 6-2. Knowledge — Dataverse 연결

**좌측 메뉴 → Knowledge → + Add knowledge → Dataverse**

아래 2개 테이블 추가:

| 테이블 | 검색 허용 컬럼 |
|---|---|
| News_Summary | report_date, department, summary_content, article_count |
| News_RawLog | article_title, description, search_keyword, department, published_at |

> **중요**: 각 테이블 추가 후 **Searchable columns** 에서 위 컬럼들을 활성화해야 에이전트가 내용을 읽을 수 있음.

---

### 6-3. 채널 연결 — Teams 또는 웹

**좌측 메뉴 → Channels**

| 채널 | 설정 |
|---|---|
| Microsoft Teams | **Add to Teams** → 팀 채널에 앱으로 추가 |
| Custom website | Embed snippet 복사 → 내부 포털에 삽입 가능 |

> POC에서는 **Teams** 채널 추가가 가장 빠름.

---

### 6-4. 테스트

**우측 Test 패널**에서 아래 질문으로 동작 확인:

```
□ "오늘 오토 뉴스 요약해줘" → News_Summary auto 리포트 반환
□ "부동산PF 기사 있어?" → News_RawLog 검색 결과 반환
□ "지난주 retail 리포트 보여줘" → 날짜 범위 질의 처리
□ "오늘 수집된 기사 몇 개야?" → 건수 집계 답변
```
