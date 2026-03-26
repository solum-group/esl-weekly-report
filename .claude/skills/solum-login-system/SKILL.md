---
name: solum-login-system
description: "ESL Weekly Report 프로젝트의 로그인/인증 시스템 전체 구조와 구현 지침. index.html 또는 admin.html의 로그인, 계정 관리, 자동 로그아웃, 비밀번호 변경, 어드민 부트스트랩과 관련된 작업을 할 때 반드시 이 스킬을 참고할 것. '로그인', '계정', '비밀번호', '어드민', '자동 로그아웃', '세션', 'Firebase 인증', '접근 권한' 등의 키워드가 나오면 즉시 이 스킬을 활성화할 것."
---

# ESL Weekly Report — 로그인 시스템 레퍼런스

## 프로젝트 파일 구조

```
/sessions/adoring-eloquent-wright/mnt/esl-weekly-report/
├── index.html      ← 사용자(일반 독자) 화면 + 로그인
├── admin.html      ← 어드민 대시보드 (접속 현황, 계정 관리)
└── data/
    ├── index.json  ← 주차 목록
    └── {weekId}.json
```

---

## Firebase 설정

```
FIREBASE_DB_URL = 'https://esl-weekly-report-default-rtdb.asia-southeast1.firebasedatabase.app'
```

Firebase Realtime Database REST API 방식 사용 (SDK 없음).
모든 읽기/쓰기는 `fetch(URL + '/path.json', { method, body })` 형태.

### 데이터 경로

| 경로 | 용도 |
|------|------|
| `/accounts/{emailKey}` | 일반 사용자 계정 |
| `/admins/{emailKey}` | 어드민 계정 |
| `/logins/{emailKey}` | 로그인 이력 (접속 통계) |
| `/likes/{articleId}` | 기사 좋아요 카운트 |

`emailKey` = `email.replace(/[@.]/g, '_')`
예: `user@solum.com` → `user_solum_com`

---

## 계정 데이터 구조

### 사용자 계정 (`/accounts/{emailKey}`)

```json
{
  "email": "user@solum.com",
  "passwordHash": "sha256_hex_string",
  "mustChangePassword": false,
  "createdAt": "2026-03-01T00:00:00.000Z"
}
```

### 어드민 계정 (`/admins/{emailKey}`)

```json
{
  "email": "admin@solum.com",
  "passwordHash": "sha256_hex_string",
  "mustChangePassword": false,
  "createdAt": "2026-03-01T00:00:00.000Z"
}
```

> **중요**: 사용자 계정과 어드민 계정은 완전히 독립적. 동일한 이메일이라도 각자의 비밀번호를 가짐.

---

## 비밀번호 처리

### SHA-256 해싱 (Web Crypto API)

```javascript
async function sha256(str) {
    const buf  = await crypto.subtle.digest('SHA-256',
                   new TextEncoder().encode(str));
    return Array.from(new Uint8Array(buf))
                .map(b => b.toString(16).padStart(2,'0'))
                .join('');
}
```

- 평문은 절대 Firebase에 저장하지 않음
- 로그인 시: `inputHash === account.passwordHash` 비교

### 초기 비밀번호

```
INIT_PW = 'thffndpa2026'
```

계정 생성 및 비밀번호 초기화 시 모두 이 값의 SHA-256 해시로 설정.

### 비밀번호 강도 조건 (isStrongPassword)

- 8자 이상
- 영문자 포함 (`/[a-zA-Z]/`)
- 숫자 포함 (`/[0-9]/`)
- 특수문자 포함 (`/[!@#$%^&*()\-_=+...]/`)

---

## 사용자 로그인 흐름 (index.html)

```
1. 이메일 입력 → emailKey 생성
2. GET /accounts/{emailKey}.json
3. account 없음 → "등록되지 않은 계정" 오류
4. sha256(입력 비밀번호) vs account.passwordHash 비교
5. 불일치 → "비밀번호 오류"
6. 일치 → sessionStorage 세팅 + recordLogin() 호출
7. account.mustChangePassword === true → 비밀번호 변경 오버레이 강제 표시
```

---

## sessionStorage 키 (index.html — 사용자)

| 키 | 값 | 용도 |
|----|----|------|
| `esl_auth_v1` | `'true'` | 인증 여부 |
| `esl_auth_email` | 이메일 문자열 | 로그인된 사용자 |
| `esl_must_change_pw` | `'true'` | 비밀번호 변경 필요 여부 |
| `esl_auth_login_time` | timestamp(ms) 문자열 | 최초 로그인 시각 |
| `esl_auth_active_time` | timestamp(ms) 문자열 | 마지막 활동 시각 |

---

## sessionStorage 키 (admin.html — 어드민)

| 키 | 값 | 용도 |
|----|----|------|
| `esl_admin_auth` | `'true'` | 어드민 인증 여부 |
| `esl_admin_email` | 이메일 문자열 | 로그인된 어드민 |
| `esl_admin_must_change` | `'true'` | 비밀번호 변경 필요 여부 |

> admin.html은 별도의 로그인 시각 타임스탬프 없이 운영 중 (자동 로그아웃 미적용).

---

## 자동 로그아웃 (index.html)

### 타임아웃 상수

```javascript
ACCESS_TOKEN_MS  = 60 * 60 * 1000;   // 1시간 — idle timeout
REFRESH_TOKEN_MS =  8 * 60 * 60 * 1000;  // 8시간 — hard limit
```

### 체크 시점 3가지

1. **페이지 로드** (`initAuth` IIFE): 타임스탬프 즉시 확인
2. **60초마다**: `setInterval(checkTokenExpiry, 60 * 1000)`
3. **탭 전환·복귀 시**: `visibilitychange` + `pageshow` 이벤트

### 활동 감지 이벤트 (idle 타이머 리셋)

```javascript
['click', 'keydown', 'scroll', 'touchstart']
```

> `mousemove`는 **제외** — 마우스 단순 이동으로 타이머가 리셋되는 문제 방지.
> 1분 이내 중복 갱신 방지 (`_lastActiveWrite` 변수 사용).

### 로그아웃 구분

| reason | 조건 | 모달 메시지 |
|--------|------|-------------|
| `'access'` | 마지막 활동 후 1시간 초과 | "1시간 동안 활동이 없어..." |
| `'refresh'` | 최초 로그인 후 8시간 초과 | "로그인 후 8시간이 경과하여..." |

---

## 어드민 부트스트랩 (admin.html)

Firebase `/admins` 경로가 비어 있을 때(최초 설정 또는 모든 어드민 삭제 시) 부트스트랩 오버레이 표시.

```
BOOTSTRAP_CODE = 'solum_admin_2026'
```

- 이 코드를 입력해야만 최초 어드민 계정 생성 가능
- 어드민이 1명이라도 존재하면 부트스트랩 코드는 무효화됨

---

## 로그인 이력 기록 (`/logins/{emailKey}`)

`recordLogin(email)` — 로그인 성공 시 자동 호출 (index.html).

```json
{
  "email": "user@solum.com",
  "count": 15,
  "firstSeen": "2026-01-10T09:00:00.000Z",
  "lastSeen":  "2026-03-24T10:30:00.000Z",
  "monthly": { "2026-03": 8, "2026-02": 5 },
  "weekly":  { "2026-W12": 3, "2026-W11": 4 }
}
```

- `monthly`/`weekly` 서브카운터: 접속 추이 차트에 활용
- 주차 키(`YYYY-WXX`): ISO 8601 기준 월요일 시작 주

---

## 비밀번호 변경 강제 플로우

계정 생성 또는 비밀번호 초기화 후 `mustChangePassword: true` 설정.
다음 로그인 시 변경 오버레이가 자동 표시되고 닫기 불가.
변경 성공 시 Firebase에서 `mustChangePassword: false`로 업데이트.

---

## 어드민 탭 구성 (admin.html)

| 탭 | 기능 |
|----|------|
| 접속 현황 | 기간 필터, 스탯 카드, 접속 추이 차트(월별/주차별), 이메일별 테이블 |
| 사용자 계정 | `/accounts` — 생성 / 초기화 / 삭제 |
| 어드민 관리 | `/admins` — 생성 / 초기화 / 삭제 (본인 삭제 불가) |

---

## 변경 시 주의사항

1. **비밀번호 해시 변경 금지**: SHA-256 알고리즘과 해싱 방식 유지 필요
2. **sessionStorage vs localStorage**: 인증은 반드시 `sessionStorage` 사용 (탭 종료 시 자동 만료)
3. **사용자/어드민 계정 분리**: `/accounts`와 `/admins`는 별개 경로, 코드도 분리 유지
4. **자동 로그아웃 타임스탬프**: `AUTH_LOGIN_TIME_KEY`와 `AUTH_ACTIVE_TIME_KEY` 둘 다 로그인 시 반드시 세팅
5. **어드민 자기 삭제 방지**: 현재 로그인된 어드민 이메일과 삭제 대상 이메일 비교 후 차단

---

## 기사 필터링 지침

크롤링 결과에서 **Align Partners**가 언급된 기사는:
- 화면에 표시하지 않음 (숨김 처리)
- 기사 카운트에 포함하지 않음
- 향후 크롤링 이후에도 동일하게 적용

구현 위치: `renderReport()` 내 섹션별 기사 렌더링 직전 필터링.

```javascript
// 기사 배열 필터링 예시
const filtered = articles.filter(a =>
    !/(align\s*partners)/i.test(a.title + ' ' + (a.summary || '') + ' ' + (a.implications || ''))
);
```
