# 🛡️ 보안 뉴스 자동화 & 취약점 탐지 시스템

> **n8n** 기반의 자동화 워크플로우로 매일 아침 보안 뉴스를 수집·분석하고,  
> 내 자산(CMDB)과 연관된 취약점을 즉시 탐지하여 경보를 발송하는 시스템입니다.

<br>

## 📌 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **도구** | n8n (자동화), PostgreSQL (DB), Groq AI (분석) |
| **실행 주기** | 매일 오전 자동 실행 (Schedule Trigger) |
| **뉴스 소스** | DailySec, TheHackerNews, BleepingComputer, KISA |
| **핵심 기능** | 뉴스 수집 → AI 분석 → CMDB 매칭 → 이메일 경보 |

<br>

## 💡 Motivations

보안 담당자로서 매일 아침 수십 개의 보안 뉴스를 직접 찾아보는 건 비효율적입니다.  
특히 **"이 취약점이 우리 시스템에도 해당되는가?"** 를 수동으로 판단하는 것은 놓치기 쉽고 위험합니다.

이 시스템은 다음 문제를 해결합니다:

- ✅ 매일 4개 소스의 보안 뉴스를 **자동 수집**하여 뉴스레터로 발송
- ✅ AI(Groq)가 각 뉴스를 **구조화된 데이터**로 분석 (CVE ID, 심각도, 대응방안 등)
- ✅ **CMDB(내부 자산 목록)** 와 자동 비교하여 우리 자산에 해당하는 위협 즉시 탐지
- ✅ 위협 탐지 시 **즉시 이메일 경보** 발송 → 빠른 대응 가능

> 예: `EFM-Networks ipTIME 취약점` 뉴스 수신 → CMDB에 ipTIME 장비 존재 → **자동 보안 경보 발송!**

<br>

---

## 🗺️ 전체 워크플로우

![전체 워크플로우](./전체_워크플로우.png)

전체 파이프라인은 크게 **두 가지 흐름**으로 구성됩니다:

| 흐름 | 설명 |
|------|------|
| 🗞️ **뉴스레터 발송** | RSS 수집 → 필터링 → Merge → JavaScript 처리 → 이메일 발송 |
| 🔍 **취약점 탐지** | RSS 수집 → AI 분석 → DB 저장 → CMDB 매칭 → 경보 발송 |

<br>

---

## 📰 워크플로우 1 — 보안 뉴스레터 발송

![뉴스 워크플로우](./뉴스워크플로우.png)

### 동작 순서

```
Schedule Trigger
    │
    ├─► RSS Read1 (DailySec)   → Filter2 ──┐
    ├─► RSS Read2 (TheHackerNews)          ├─► Merge (append)
    ├─► RSS Read3 (BleepingComputer)       │       │
    └─► RSS Read4 (KISA) ─────────────────┘       │
                                                   ▼
                                        Code in JavaScript2
                                        (HTML 뉴스레터 생성)
                                                   │
                                                   ▼
                                           Send email2 📧
```

### 핵심 로직 (Code in JavaScript2)

소스별로 최대 7개씩 선별 후 랜덤 셔플하여 균형 잡힌 뉴스레터를 생성합니다.

```javascript
// 소스별 분류 및 색상 태그 지정
if (link.includes('boannews'))        boan.push({ ...data, tag: '[보안뉴스]',     color: '#d32f2f' });
else if (link.includes('dailysecu'))  daily.push({ ...data, tag: '[데일리시큐]',  color: '#1a73e8' });
else if (link.includes('krcert'))     kisa.push({ ...data, tag: '[KISA]',        color: '#2e7d32' });
else if (link.includes('thehackernews')) hackers.push({ ...data, tag: '[TheHackerNews]', color: '#f57c00' });
else if (link.includes('bleepingcomputer')) bleeping.push({ ...data, tag: '[BleepingComp]', color: '#455a64' });

// 각 소스에서 최대 7개 선택 후 랜덤 믹스
const finalSelection = [
    ...boan.slice(0, 7), ...daily.slice(0, 7),
    ...kisa.slice(0, 7), ...hackers.slice(0, 7), ...bleeping.slice(0, 7)
];
finalSelection.sort(() => 0.5 - Math.random());
```

### 실행 결과 — 이메일 수신 화면

![뉴스 메일](./뉴스_메일.png)
![뉴스 내용](./뉴스.png)

<br>

---

## 🔍 워크플로우 2 — AI 취약점 분석 & CMDB 매칭 경보

![취약점 워크플로우](./취약점_워크플로우.png)

### 동작 순서

```
Filter (보안 관련 키워드 포함 뉴스만 통과)
    │
    ▼
AI Agent (Groq) — 뉴스 분석 및 구조화
    │  - CVE ID 추출
    │  - 심각도(Severity) 판단
    │  - 영향받는 서비스명 추출
    │  - 대응방안 요약
    ▼
Code in JavaScript — AI 응답 JSON 파싱
    │
    ▼
Insert rows in a table — vulnerability_news 테이블에 저장
    │
    ▼
Execute a SQL query — CMDB 전체 자산 목록 조회
    │
    ▼
Code in JavaScript1 — CMDB vs 취약점 매칭 비교
    │
    ├─ 매칭 O ─► Insert rows in a table1 (security_alerts 저장)
    │                    │
    │                    ▼
    │               Send email 📧 (보안 경보 발송!)
    │
    └─ 매칭 X ─► (종료)
```

### AI Agent 프롬프트 (Groq 모델)

Groq AI가 뉴스 원문을 분석해 아래 형태의 JSON을 반환합니다:

```json
{
  "cve_id": "CVE-2026-24498",
  "target_service": "ipTIME",
  "severity": "HIGH",
  "summary": "EFM-Networks ipTIME 유무선공유기의 보안 기능이 우회되어 내부 네트워크 접근 가능",
  "solution": "최신 펌웨어 업데이트 설치 권장"
}
```

### CMDB 매칭 로직 (Code in JavaScript1)

```javascript
for (const asset of cmdbData) {
    const serviceName   = (assetJson.service_name || "").toLowerCase();
    const targetService = (news.target_service   || "").toLowerCase();

    // 양방향 포함 관계 확인 (부분 일치도 탐지)
    if (targetService.includes(serviceName) || serviceName.includes(targetService)) {
        alerts.push({
            json: {
                is_matched:   true,
                asset_id:     assetJson.id,
                news_id:      news.id,
                alert_level:  news.severity,
                asset_name:   assetJson.asset_name,
                service_info: `${assetJson.service_name} (v${assetJson.service_version})`,
                news_title:   news.title,
                cve_id:       news.cve_id,
            }
        });
    }
}
return alerts.length > 0 ? alerts : [{ json: { is_matched: false } }];
```

### 실행 결과 — 보안 경보 이메일

![보안 알림](./보안알림.png)

<br>

---

## 🗄️ 데이터베이스 스키마

### ERD 구조

```
┌─────────────────────┐        ┌──────────────────────────┐
│        CMDB         │        │    vulnerability_news     │
├─────────────────────┤        ├──────────────────────────┤
│ id (PK)             │        │ id (PK)                  │
│ asset_name          │        │ title                    │
│ ip_address          │        │ summary                  │
│ os_info             │        │ cve_id                   │
│ service_name        │        │ target_service           │
│ service_version     │        │ severity                 │
│ last_scan_date      │        │ solution                 │
└──────────┬──────────┘        │ link                     │
           │                   │ created_at               │
           │           ┌───────┴──────────────────────────┐
           │           │                                  │
           └───────────►        security_alerts           │
                       ├──────────────────────────────────┤
                       │ id (PK)                          │
                       │ asset_id (FK → CMDB)             │
                       │ news_id  (FK → vulnerability_news)│
                       │ alert_level                      │
                       │ detected_at                      │
                       │ status (기본값: 'Open')           │
                       └──────────────────────────────────┘
```

### 테이블 상세

#### `CMDB` — 내부 자산 관리

```sql
CREATE TABLE IF NOT EXISTS CMDB (
    id               SERIAL PRIMARY KEY,
    asset_name       VARCHAR(100),
    ip_address       VARCHAR(45),
    os_info          VARCHAR(50),
    service_name     VARCHAR(50),    -- 매칭의 핵심 컬럼
    service_version  VARCHAR(20),
    last_scan_date   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### `vulnerability_news` — AI 분석 취약점 정보

```sql
CREATE TABLE IF NOT EXISTS vulnerability_news (
    id              SERIAL PRIMARY KEY,
    title           TEXT,
    summary         TEXT,
    cve_id          VARCHAR(50),
    target_service  VARCHAR(50),     -- CMDB.service_name 과 비교
    severity        VARCHAR(20),
    solution        TEXT,
    link            TEXT,
    created_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

#### `security_alerts` — 탐지된 위협 경보 로그

```sql
CREATE TABLE IF NOT EXISTS security_alerts (
    id          SERIAL PRIMARY KEY,
    asset_id    INTEGER REFERENCES CMDB(id),
    news_id     INTEGER REFERENCES vulnerability_news(id),
    alert_level VARCHAR(20),
    detected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status      VARCHAR(20) DEFAULT 'Open'
);
```

| 테이블 | 역할 | 성격 |
|--------|------|------|
| `CMDB` | 내부 자산(서버, IP, 서비스 버전) 기준 정보 | 수동 관리 |
| `vulnerability_news` | AI가 분석한 외부 위협 정보 | 자동 적재 |
| `security_alerts` | CMDB × 뉴스 매칭 탐지 결과 | 자동 생성 |

<br>

---

## 📚 용어 사전 (Glossary)

> 이 프로젝트를 만들며 학습한 핵심 용어들을 정리했습니다.

| 용어 | 설명 |
|------|------|
| **n8n** | 노드 기반의 오픈소스 워크플로우 자동화 도구. 코드 없이도 API·DB·AI를 연결 가능 |
| **RSS** | Really Simple Syndication. 웹사이트의 최신 콘텐츠를 구독할 수 있는 XML 형식의 피드 |
| **Schedule Trigger** | 지정된 시간마다 워크플로우를 자동으로 실행시키는 n8n 노드 |
| **Merge (append)** | 여러 노드의 출력을 하나의 스트림으로 합치는 n8n 노드 |
| **CVE** | Common Vulnerabilities and Exposures. 공개된 보안 취약점에 부여되는 고유 식별 번호 (예: CVE-2026-24498) |
| **CMDB** | Configuration Management Database. 조직의 IT 자산(서버, 네트워크 장비 등) 구성 정보를 관리하는 DB |
| **KISA** | 한국인터넷진흥원. 국내 사이버 보안 위협·취약점 정보를 공식 발표하는 기관 |
| **Groq** | 고속 AI 추론 서비스. LLaMA 등 오픈소스 LLM을 빠르게 실행할 수 있는 플랫폼 |
| **AI Agent (n8n)** | LLM을 활용해 입력 데이터를 분석·판단하고 구조화된 출력을 생성하는 n8n 노드 |
| **Severity** | 취약점의 심각도 등급. CRITICAL / HIGH / MEDIUM / LOW 로 분류 |
| **Security Alert** | 탐지된 위협을 담당자에게 알리는 경보. 본 시스템에서는 DB 저장 + 이메일 발송으로 구현 |
| **Air-Gapped Network** | 인터넷과 물리적으로 분리된 폐쇄망. 고보안 환경에서 사용 |
| **RAT** | Remote Access Trojan. 공격자가 원격으로 시스템을 제어하기 위해 사용하는 악성 소프트웨어 |

<br>

---

## 🛠️ 기술 스택

| 분류 | 기술 |
|------|------|
| 자동화 | n8n |
| AI 분석 | Groq (LLaMA 기반) |
| 데이터베이스 | PostgreSQL |
| 알림 | SMTP 이메일 |
| 언어 | JavaScript (n8n Code 노드) |

<br>

---

## 🚀 실행 방법

1. **n8n 설치 및 실행**
   ```bash
   npx n8n
   ```

2. **PostgreSQL 테이블 생성** — 위 DDL 쿼리 순서대로 실행
   ```
   CMDB → vulnerability_news → security_alerts
   ```

3. **CMDB에 자산 등록** — 모니터링할 서비스 정보 입력

4. **n8n에서 워크플로우 Import** — `.json` 파일 업로드

5. **크리덴셜 설정**
   - PostgreSQL 연결 정보
   - Groq API Key
   - SMTP 이메일 계정

6. **Schedule Trigger 활성화** → 매일 자동 실행 🎉

<br>

---

## 📁 프로젝트 구조

```
📦 security-news-automation
 ┣ 📄 workflow_full.json          # n8n 전체 워크플로우 export
 ┣ 📄 schema.sql                  # DB 테이블 생성 쿼리
 ┣ 📄 README.md
 ┗ 📁 screenshots/
    ┣ 전체_워크플로우.png
    ┣ 뉴스워크플로우.png
    ┣ 취약점_워크플로우.png
    ┣ 뉴스_메일.png
    ┣ 뉴스.png
    ┗ 보안알림.png
```

<br>

---

<div align="center">

**"자동화로 보안을 일상에 녹여내다"**  
매일 아침 커피 한 잔을 마시는 동안, 이 시스템이 오늘의 위협을 분석합니다. ☕🛡️

</div>
