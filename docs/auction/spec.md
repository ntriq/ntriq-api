# 법원경매 데이터 API 명세서

발행 2026-09-03. 엔트릭(ntriq) 문서 버전 1.0

이 문서는 계약 고객에게 나가는 납품 규격이다. 아래에 적힌 필드와 갱신 주기는 약속이며
바뀔 때는 사전 고지한다. 서비스 수준과 이용 조건은 별도 문서 [SLA](sla.md)를 함께 본다.

---

## 1. 접속

| 항목 | 값 |
|---|---|
| 기본 주소 | `https://api.ntriq.co.kr` |
| 경로 접두어 | `/lawink/api/v1/auction` |
| 인증 헤더 | `X-YAP-Key: <발급키>` (경매 경로는 키 없이 호출할 수 없다) |
| 응답 형식 | `application/json`, UTF-8 |
| 한도 안내 헤더 | `X-RateLimit-Limit` `X-RateLimit-Remaining` `X-RateLimit-Reset` |
| 추적 헤더 | `X-Request-ID` (에러 본문의 `request_id`와 같은 값) |

```bash
curl -H "X-YAP-Key: $NTRIQ_KEY" \
  "https://api.ntriq.co.kr/lawink/api/v1/auction/health"
```

---

## 2. 식별자 규약

**한 물건을 가리키는 열쇠는 세 개다. 법원명, 사건번호, 물건번호.**

사건번호(`2025타경102638`)는 법원마다 따로 채번한다. 그래서 전국 단위에서 사건번호만으로는
서로 다른 물건이 같은 번호를 가질 수 있다. 물건 단위 경로가 `/items/{court}/{case_no}/{item_no}`
형태인 이유가 이것이다.

```
/lawink/api/v1/auction/items/서울중앙지방법원/2025타경102638/1
```

- `court`는 아래 정규형을 URL 인코딩해서 넣는다.
- `item_no`는 한 사건에 물건이 여럿일 때를 구분한다. 단일 물건이면 `1`이다.
- 이 세 값은 물건이 존재하는 동안 바뀌지 않는다. 내부 `id`(UUID)는 참고용이며 조인 키로 쓰지 않기를 권한다.

### 법원명 정규형

**정규형은 법원경매정보 검색 화면의 법원 선택 목록에 있는 이름이다.** 지원은 축약형을 쓴다.

| 정규형 | 같은 법원의 다른 표기 |
|---|---|
| `대구서부지원` | 대구지방법원서부지원 |
| `부산동부지원` | 부산지방법원동부지원 |
| `강릉지원` | 춘천지방법원강릉지원 |
| `서울중앙지방법원` | (본원은 전체명이 곧 정규형) |

**입력 표기는 무관하다.** 위 표의 어느 쪽으로 보내도 같은 물건이 나온다. 다만 **응답에 실려
나가는 `court_name`은 언제나 정규형 하나**다. 원문을 되돌려주지 않는 것은 의도이며, 식별자가
둘이면 어느 쪽으로 저장할지가 고객 코드마다 갈리기 때문이다. 자기 데이터베이스에는 응답에
담겨 온 값을 그대로 넣으면 된다.

도시가 빠진 `동부지원` 같은 이름은 그것만으로 법원을 특정할 수 없다. 부산에도 서울에도 있다.
이런 표기는 물건의 소재지가 있어야 확정되므로, 조회할 때는 도시를 붙인 정규형을 쓰기 바란다.

### 물건번호 정렬

`item_no`는 문자열이지만 정렬은 숫자 순서로 한다. 한 사건에 물건이 66개인 사례가 실제로 있고
(광주지방법원 2025타경34066), 문자열로 정렬하면 10번이 2번 앞에 온다. 목록과 사건 조회와
스냅샷 모두 1, 2, 10, 13 순서로 나간다.

---

## 3. 갱신 주기

| 단계 | 시각(KST) | 채우는 것 |
|---|---|---|
| 목록 수집 | 매일 06:00 | 전국 법원의 매각기일이 잡힌 물건 전부. 완전성은 당일 manifest와 상태 응답으로 확인 |
| 회차 수집 | 매일 08:00 | 기일내역. 유찰과 변경, 재매각 회차 |
| 상세 수집 | 매일 12:00 | 물건 상세와 문건·송달. 하루 예산 3,000사건 |
| 명세서 열람 | 매일 08:15 | 매각물건명세서 표 판독. 하루 300건 |
| 스냅샷 생성 | 매일 08:30 | 그날의 전량을 시·도별 파일로 고정 |
| 권리분석 | 매일 09:30 | 명세서 판독 결과로 `/rights` 생성·갱신 |

증분과 스냅샷은 쓰임이 다르다. 처음 적재하거나 자기 데이터베이스가 어긋났을 때는 스냅샷을
기준으로 삼고, 그 뒤로는 `/changes?since=`로 바뀐 것만 받는다.

---

## 4. 엔드포인트

모두 `GET`이며 접두어 `/lawink/api/v1/auction`이 붙는다.

| 경로 | 주요 파라미터 | 핵심 응답 필드 |
|---|---|---|
| `/search` | `sido` `usage` `min_price` `max_price` `status` `court_name` `keyword` `sale_date` `sale_date_from` `sale_date_to` `sort_by` `sort_order` `page` `limit` | `items[]` `total` `page` `limit` `total_pages` `next_page` |
| `/today` | 없음 | `date` `items[]` `total` |
| `/items/{court}/{case_no}/{item_no}` | `include_rounds` | `item` `detail` `rounds[]` `rounds_total` `analysis` `sources`, `detail`에는 `source_availability` `sale_result_observations[]` 포함 |
| `/items/{court}/{case_no}/{item_no}/rounds` | `page` `limit` | `rounds[]` `total` `next_page` |
| `/items/{court}/{case_no}/{item_no}/history` | `page` `limit` | `entries[]` `total` `next_page` |
| `/items/{court}/{case_no}/{item_no}/rights` | 없음 | `status` `pending_reason` `rights_analysis` `input_availability` `risk_score` `risk_level` `notice` |
| `/changes` | 최초 `since`, 이후 `cursor`, `sido` `limit` | `changes[]` `count` `next_since` `next_cursor` |
| `/case/{case_no}` | `court` | `items[]` `analyses[]` `count` `courts[]` |
| `/snapshots` | `date` `limit` | `snapshots[]` `latest` |
| `/snapshots/{date}/{filename}` | 없음 | 파일 본문 |
| `/stats` | 없음 | `total_items` `by_sido` `by_usage` `last_crawled_at` |
| `/regions` | 없음 | `regions[]` |
| `/courts` | 없음 | `courts[]`(법원명·물건 수, 물건 수 내림차순) |
| `/usages` | 없음 | `usages[]` |
| `/regional-risk` | 지역 필터 | `regions[]` `disclaimer` |
| `/risk-heatmap` | 지역 필터 | `regions[]` `national_stats` |
| `/health` | 없음 | `status` `total_items` `last_crawled_at` |

기계 판독용 정본은 [OpenAPI](../../openapi/auction.json)(OpenAPI 3.1)이다. 위 표와 어긋나면 OpenAPI가 맞다.
회귀 테스트가 두 목록의 일치를 매번 검사한다.

`/search`의 `court_name`은 2절의 법원명 정규형 규칙을 그대로 쓴다. `부산지방법원동부지원`처럼
다른 표기로 보내도 `부산동부지원`으로 정규화되어 매칭되며, 정규형으로 확정되지 않는 이름은
원문 그대로 비교해 결과가 없을 수 있다.

### 공통 페이지네이션

목록을 돌려주는 모든 응답은 같은 네 필드에 `next_page`를 더해 쓴다.

```json
{ "total": 3227, "page": 1, "limit": 20, "total_pages": 162, "next_page": 2 }
```

`next_page`는 마지막 페이지에서 `null`이다. 반복문 종료 조건으로 그대로 쓰면 된다.
`/changes`는 같은 시각의 변경 행도 빠뜨리지 않도록 불투명한 `next_cursor`를 쓴다. 값이 `null`이면
최초 요청 때 고정된 범위를 모두 읽은 것이다. 커서를 해석하거나 수정하지 말고 다음 호출에 그대로
보낸다. 자세한 규약은 `auction-cursor-contract.md`를 따른다.

### 공통 에러

```json
{
  "error": {
    "code": "item_not_found",
    "message": "서울중앙지방법원 2025타경102638 물건번호 9를 찾을 수 없습니다.",
    "request_id": "a1b2c3d4"
  }
}
```

| 코드 | 상태 | 뜻 |
|---|---|---|
| `invalid_since` | 400 | `since` 값이 ISO 8601 형식이 아니다 |
| `invalid_date` | 400 | 날짜가 `YYYY-MM-DD` 형식이 아니다 |
| `invalid_date_range` | 400 | 시작일이 종료일보다 늦다 |
| `invalid_item_identity` | 400 | 법원명, 사건번호, 물건번호 중 하나가 명시적으로 비어 있다 |
| `invalid_filename` | 400 | 스냅샷 파일 이름이 올바르지 않다 |
| `item_not_found` | 404 | 세 열쇠로 특정되는 물건이 없다 |
| `case_not_found` | 404 | 해당 사건번호가 없다 |
| `rights_service_unavailable` | 503 | 권리분석 저장소를 일시적으로 조회할 수 없다 |
| `snapshot_not_found` | 404 | 그 날짜의 스냅샷이 없다 |
| `snapshot_file_not_found` | 404 | 그 스냅샷 파일이 없다 |

`message`는 사람이 읽는 문장이라 문구가 바뀔 수 있다. 분기 조건으로는 `code`만 쓴다.
문의할 때 `request_id`를 함께 알려 주면 서버 로그에서 그 요청을 바로 찾는다.

예외가 둘 있다. 파라미터 타입이 아예 맞지 않아 프레임워크가 걸러내는 422 응답과, 게이트웨이가
인증과 한도에서 돌려주는 401과 429다. 이 셋은 위 봉투 대신 `detail` 또는 `error` 문자열을
담은 다른 형태로 온다. 상태 코드로 먼저 갈라 처리하기를 권한다.

---

## 5. 요청과 응답 예시

아래는 모두 2026-09-03 운영 서버에서 실제로 받은 응답이다. 개인 이름은 `○`으로 가렸고,
길이 때문에 일부 배열은 줄였다. 줄인 곳은 그 자리에 밝혀 두었다.

### 5.1 헬스체크

```bash
curl -H "X-YAP-Key: $NTRIQ_KEY" \
  "https://api.ntriq.co.kr/lawink/api/v1/auction/health"
```

```json
{
  "status": "ok",
  "timestamp": "2026-09-03T01:48:44.681129+00:00",
  "service": "court-auction",
  "regions": 17,
  "total_items": 20488,
  "last_crawled_at": "2026-09-02T21:22:40.795056"
}
```

### 5.2 검색

```bash
curl -H "X-YAP-Key: $NTRIQ_KEY" -G \
  --data-urlencode "sido=부산" --data-urlencode "limit=1" \
  "https://api.ntriq.co.kr/lawink/api/v1/auction/search"
```

```json
{
  "items": [
    {
      "id": "f05a84bb-8542-4a2a-9865-dfbe718b0a31",
      "case_no": "2025타경23252",
      "court_name": "부산지방법원",
      "item_no": "1",
      "location": "부산광역시 부산진구 부전동 226-1 위클리스타 11층1110호[집합건물 철근콘크리트구조 26.78㎡]",
      "usage": "오피스텔",
      "sido": "부산",
      "sigungu": "부산진구",
      "appraisal_price": 200000000,
      "minimum_bid_price": 200000000,
      "bid_ratio": 100.0,
      "sale_date": "2026-06-16",
      "sale_date_str": "2026.06.16",
      "bid_method": null,
      "court_dept": "경매4계",
      "status": "진행",
      "failed_count": 0,
      "remarks": "신건",
      "collected_at": "2026-09-01T21:04:40.970301"
    }
  ],
  "total": 3227,
  "page": 1,
  "limit": 1,
  "total_pages": 3227,
  "next_page": 2
}
```

부산 한 곳만 3,227건이다. 시·도 전체를 받으려면 `next_page`가 `null`이 될 때까지 돌리거나,
스냅샷 파일 하나를 받는 편이 빠르다.

### 5.3 통계

```json
{
  "total_items": 20488,
  "active_items": 18470,
  "failed_items": 1551,
  "sold_items": 28,
  "by_sido": {
    "서울": 2654, "부산": 3227, "인천": 2139, "대구": 1269, "광주": 1517,
    "대전": 830, "울산": 918, "세종": 574, "경기": 921, "강원": 588,
    "충북": 688, "충남": 826, "전북": 298, "전남": 1363, "경북": 254,
    "경남": 1250, "제주": 1165
  },
  "last_crawled_at": "2026-09-02T21:22:40.795056"
}
```

### 5.4 물건 통합 조회

```bash
curl -H "X-YAP-Key: $NTRIQ_KEY" \
  "https://api.ntriq.co.kr/lawink/api/v1/auction/items/%EC%84%9C%EC%9A%B8%EC%A4%91%EC%95%99%EC%A7%80%EB%B0%A9%EB%B2%95%EC%9B%90/2025%ED%83%80%EA%B2%BD102638/1"
```

```json
{
  "court_name": "서울중앙지방법원",
  "case_no": "2025타경102638",
  "item_no": "1",
  "item": {
    "id": "59f052b5-ed8b-425c-aa87-ee70b5e74e93",
    "case_no": "2025타경102638",
    "court_name": "서울중앙지방법원",
    "item_no": "1",
    "location": "서울특별시 관악구 신림동 1568-5 더위일관악파크뷰 8층806호[집합건물 철근콘크리트구조 17.3㎡]",
    "usage": "상가,오피스텔,근린시설",
    "sido": "서울",
    "sigungu": "관악구",
    "appraisal_price": 240000000,
    "minimum_bid_price": 192000000,
    "bid_ratio": 80.0,
    "sale_date": "2026-09-03",
    "sale_date_str": "2026.09.03",
    "bid_method": null,
    "court_dept": "경매2계",
    "status": "유찰",
    "failed_count": 5,
    "remarks": "임차인 및 임차보증금반환채권승계인 주택도시보증공사의 매수인에 대한 대항력 포기조건 매각",
    "collected_at": "2026-03-07T21:02:21.036310"
  },
  "detail": {
    "item_kind": "상가,오피스텔,근린시설",
    "bid_method": "기일입찰",
    "court_dept": "경매2계",
    "electronic": true,
    "appraisal_price": 240000000,
    "minimum_bid_price": 192000000,
    "bid_deposit": 19200000,
    "sale_date": "2026-09-03",
    "sale_time": "10:00",
    "sale_place": "경매법정(제4별관211호)",
    "case_received_at": "2025-04-08",
    "auction_start_at": "2025-04-11",
    "dividend_deadline": "2025-06-25",
    "claim_amount": 255551370,
    "bidder_count": null,
    "sale_price": null,
    "sale_result": "유찰",
    "failed_count": 5,
    "source_availability": {
      "sale_date": "observed",
      "sale_result": "observed",
      "sale_price": "not_observed",
      "bidder_count": "not_present_in_result_view"
    },
    "sale_result_observations": [],
    "fetched_at": "2026-09-03T01:21:32.554240"
  },
  "rounds": [],
  "rounds_total": 7,
  "analysis": {
    "investment_score": 85,
    "investment_grade": "A",
    "risk_level": "low",
    "recommendation": "적극 검토 추천",
    "factors": { "failed_count_bonus": 25, "bid_ratio_adjustment": 80.0, "status": "유찰" }
  },
  "sources": {
    "list_collected_at": "2026-03-07T21:02:21.036310",
    "detail_fetched_at": "2026-09-03T01:21:32.554240"
  }
}
```

`rounds` 본문은 아래 5.5와 같으며 길이 때문에 이 예시에서는 비웠다. `include_rounds=false`로
부르면 실제로도 빈 배열이 온다. `detail`이 `null`이면 아직 상세를 수집하지 않은 물건이며
이때도 응답은 200이다. 목록 층만으로 적재를 이어갈 수 있어야 하기 때문이다.

`analysis`는 유찰 횟수와 최저가율로 계산한 참고 점수다. 권리분석이 아니며 투자 자문도 아니다.
`source_availability`는 허용된 네 필드만 내보낸다. `observed`는 값을 원천에서 직접 봤다는 뜻이고,
`not_observed`와 `null`은 아직 모른다는 뜻이다. 특히 가격 `0`은 저장돼 있으면 관측값이며, `null`을
0원으로 바꾸면 안 된다. `sale_result_observations`의 낙찰가와 결과는 각 행의 `sale_date`와
`source_captured_on`에 묶인다. 과거 낙찰가를 이후 재매각 회차의 가격으로 옮겨 쓰지 않는다.
원천의 임의 키, `raw_detail`, 문서 뷰어용 세션 값은 공개하지 않는다.

### 5.5 기일내역

```bash
curl -H "X-YAP-Key: $NTRIQ_KEY" \
  "https://api.ntriq.co.kr/lawink/api/v1/auction/items/%EC%84%9C%EC%9A%B8%EC%A4%91%EC%95%99%EC%A7%80%EB%B0%A9%EB%B2%95%EC%9B%90/2025%ED%83%80%EA%B2%BD102638/1/rounds"
```

```json
{
  "court_name": "서울중앙지방법원",
  "case_no": "2025타경102638",
  "item_no": "1",
  "rounds": [
    { "round_no": 1, "round_date": "2026-03-12", "round_time": "10:00", "round_kind": "매각기일", "round_place": "경매법정(제4별관211호)", "minimum_bid_price": 240000000, "round_result": "유찰", "source": "detail", "fetched_at": "2026-09-03T01:21:32.554240" },
    { "round_no": 2, "round_date": "2026-04-16", "round_time": "10:00", "round_kind": "매각기일", "round_place": "경매법정(제4별관211호)", "minimum_bid_price": 192000000, "round_result": "유찰", "source": "detail", "fetched_at": "2026-09-03T01:21:32.554240" },
    { "round_no": 3, "round_date": "2026-05-21", "round_time": "10:00", "round_kind": "매각기일", "round_place": "경매법정(제4별관211호)", "minimum_bid_price": 153600000, "round_result": "유찰", "source": "detail", "fetched_at": "2026-09-03T01:21:32.554240" },
    { "round_no": 4, "round_date": "2026-06-25", "round_time": "10:00", "round_kind": "매각기일", "round_place": "경매법정(제4별관211호)", "minimum_bid_price": 122880000, "round_result": "유찰", "source": "detail", "fetched_at": "2026-09-03T01:21:32.554240" },
    { "round_no": 5, "round_date": "2026-07-30", "round_time": "10:00", "round_kind": "매각기일", "round_place": "경매법정(제4별관211호)", "minimum_bid_price": 240000000, "round_result": "유찰", "source": "detail", "fetched_at": "2026-09-03T01:21:32.554240" },
    { "round_no": 6, "round_date": "2026-09-03", "round_time": "10:00", "round_kind": "매각기일", "round_place": "경매법정(제4별관211호)", "minimum_bid_price": 192000000, "round_result": null, "source": "detail", "fetched_at": "2026-09-03T01:21:32.554240" },
    { "round_no": 7, "round_date": "2026-09-10", "round_time": "14:00", "round_kind": "매각결정기일", "round_place": "배당법정(제4별관 3층 7호)", "minimum_bid_price": null, "round_result": null, "source": "detail", "fetched_at": "2026-09-03T01:21:32.554240" }
  ],
  "total": 7,
  "page": 1,
  "limit": 50,
  "total_pages": 1,
  "next_page": null
}
```

5회차에서 최저가가 다시 감정가로 올라간 것은 재매각이다. 목록의 `failed_count`는 요약이고
회차의 정본은 이 표다. 두 값이 다르면 이 표를 믿는다.

### 5.6 변경 이력과 변경 스트림

한 물건의 이력은 `/history`, 전체 증분은 `/changes?since=`로 받는다. 두 응답의 각 행에는
`source`가 붙고 뜻이 서로 다르다.

| `source` | 뜻 | 2026-09-03 기준 누적 |
|---|---|---|
| `seed` | 이력 테이블을 만들 때 넣은 최초 기준선 | 20,378행 |
| `list` | 목록 화면에서 관측한 변경. 우리가 언제 무엇을 봤는지 | 1,784행 |
| `detail` | 상세 화면 기록에서 온 값 | 기일내역 표가 정본 |

```json
{
  "observed_at": "2026-09-02T21:03:10.929359",
  "round_no": 6,
  "sale_date": "2026-09-03",
  "sale_date_kind": "매각기일",
  "minimum_bid_price": 192000000,
  "status": "유찰",
  "status_detail": "유찰",
  "changed_fields": {
    "status": { "old": "진행", "new": "유찰" },
    "status_detail": { "old": null, "new": "유찰" }
  },
  "source": "list"
}
```

증분 조회는 응답의 `next_since`를 다음 호출의 `since`에 그대로 넣으면 이어진다.
`since`는 포함 경계(`observed_at >= since`)라 직전 응답의 마지막 변경 1건이 다음 응답 맨 앞에 다시 온다 — 변경 항목의 `id`로 거르면 된다. 같은 시각 여러 건이 끊기지 않게 하려는 설계다. 페이지를 넘길 때는 `next_cursor`를 쓰면 중복이 없다. `observed_at`·`since`·`next_since`는 UTC이며 `+00:00` 표기를 붙여 내보낸다(시간대 없는 `since`는 UTC로 읽는다).

```bash
curl -H "X-YAP-Key: $NTRIQ_KEY" -G \
  --data-urlencode "since=2026-09-02T00:00:00" --data-urlencode "sido=부산" \
  "https://api.ntriq.co.kr/lawink/api/v1/auction/changes"
```

### 5.7 스냅샷

```bash
# 목록
curl -H "X-YAP-Key: $NTRIQ_KEY" \
  "https://api.ntriq.co.kr/lawink/api/v1/auction/snapshots?date=2026-09-03"
```

```json
{
  "snapshots": [
    {
      "date": "2026-09-03",
      "files": [
        { "filename": "부산.jsonl.gz", "sido": "부산", "bytes": 122847,
          "download_url": "/lawink/api/v1/auction/snapshots/2026-09-03/부산.jsonl.gz" },
        { "filename": "manifest.json", "sido": null, "bytes": 4279,
          "download_url": "/lawink/api/v1/auction/snapshots/2026-09-03/manifest.json" }
      ],
      "total_bytes": 1147923
    }
  ],
  "count": 1,
  "latest": "2026-09-03"
}
```

`files` 배열은 실제로 19개다. 위 예시는 두 개만 남기고 줄였다. 2026-09-03 스냅샷은 시도
17개에 소재지 미상 파일과 manifest 를 더해 19개 파일, 전체 20,488건, 압축 후 약 1.1MB다.
부산 파일 하나가 122,847바이트에 3,227줄이며 이 줄 수는 `/stats`의 부산 건수와 같다.

파일은 gzip으로 압축한 JSON Lines다. 한 줄이 물건 한 건이며 목록과 상세와 기일내역을 함께 담는다.

```json
{"court_name":"서울중앙지방법원","case_no":"2025타경102638","item_no":"1","item":{...},"detail":{...},"rounds":[...]}
```

같은 날짜로 다시 생성하면 그 날짜 폴더를 통째로 새로 쓴다. 부분만 갱신된 스냅샷은 기준이 될 수
없기 때문이다. `manifest.json`에는 파일별 건수와 바이트와 압축 전 내용의 sha256이 들어 있어
받은 파일이 온전한지 대조할 수 있다. 보관 기간은 30일이다.
기본 생성은 식별자가 비어 있으면 게시 전에 실패하는 strict 모드다. 운영자가 명시적으로
`--quarantine-invalid-identities`를 쓴 경우에만 잘못된 행을 제외한다. 이때 manifest의
`completeness`는 `false`이고 `quarantine.excluded_count`와 `quarantine-report.json`이 제외 행을
밝힌다. 따라서 파일이 정상적으로 열려도 manifest가 불완전하면 전량 스냅샷으로 간주하면 안 된다.

### 5.8 권리분석

```json
{
  "court_name": "서울중앙지방법원",
  "case_no": "2025타경102638",
  "item_no": "1",
  "status": "pending",
  "pending_reason": "no_analysis",
  "rights_analysis": null,
  "input_availability": {
    "tenants": "unknown",
    "base_right": "unknown",
    "dividend_request": "unknown",
    "assumed_rights": "unknown"
  },
  "notice": "권리분석이 아직 생성되지 않았습니다. 상세·문서 수집 후 생성됩니다."
}
```

분석이 아직 없는 물건에는 404 대신 `status`가 `pending`인 200을 준다. "물건이 없다"와
"물건은 있는데 분석 전이다"를 고객 코드가 다르게 다뤄야 하기 때문이다. 분석이 준비되면
`status`가 `ready`가 되고 `rights_analysis`와 `risk_score`와 `risk_level`이 채워진다.
저장된 분석이 있어도 법률 입력 4종을 하나도 관측하지 못했으면 `ready`가 아니라 `pending`이고
`pending_reason`이 `inputs_unavailable`이다. 이때 `rights_analysis`에는 물건비고에서 뽑은 참고 신호만 있고,
`risk_score`와 `risk_level`은 `null`이다(입력을 못 본 물건을 `safe`·0점으로 내지 않는다).
분석 자체가 없을 때의 `pending_reason`은 `no_analysis`, `ready`일 때는 `null`이다.
`ready`는 입력이 하나 이상 관측됐다는 뜻일 뿐 법률 입력이 완전하다는 뜻은 아니다. `input_availability`가
`unknown`인 항목의 빈 배열이나 `null`은 해당 권리가 없다는 결론이 아니다. 일부 입력만 관측된
응답에는 이런 한계를 `notice`로 함께 적는다. 저장소 조회 자체가 실패하면 `pending`으로 숨기지 않고
공통 에러 봉투의 503 `rights_service_unavailable`을 반환한다.

---

## 6. 원천 데이터의 제약

우리가 만들 수 없고 법원이 정하는 한계다. 계약 전에 밝힌다.

1. **매각 결과는 기일 다음 날부터 7일 동안만 공개된다.** 그 창을 놓치면 그 물건의 낙찰가는
   원천에서 사라진다. 우리는 매일 수집으로 창 안에 들어가려 하지만 놓친 건은 채울 방법이 없다.
2. **매각물건명세서와 현황조사서와 감정평가서는 노출 기간이 정해져 있고 본문이 이미지다.**
   매각물건명세서는 매일 300건까지 열람해 표를 판독하고 그 결과를 `/rights`의 권리분석으로 제공한다.
   본문 이미지와 원문 텍스트 자체는 제공하지 않고 임차인 성명은 마스킹한다. 현황조사서와
   감정평가서는 문서의 존재 여부와 열람 토큰까지만 제공한다.
3. **상세 수집은 하루 3,000사건 예산 안에서 돈다.** 원천 사이트에 부담을 주지 않기 위한
   자체 제한이다. 전량 한 바퀴는 목표가 7일이고 지금은 미달이다. 현재 상태와 우리가
   약속하는 범위는 SLA 3절에 있다. 상세가 아직 없는 물건은 `detail`이 `null`이고 목록
   필드만 나간다. 물건 자체의 존재와 기일은 목록 수집이 매일 채우므로 상세 유무와
   무관하다. 협의한 체험·계약 법원부터 먼저 채운다.
4. **목록과 상세는 서로 다른 시각에 본 두 개의 관측이다.** 값이 어긋날 때 우리는 한쪽을 조용히
   덮어쓰지 않고 이력 한 행을 남긴다. 어느 쪽이 언제 무엇이었는지가 남아야 하기 때문이다.

---

## 7. 출처 표시

**고객 화면에 이 API 데이터를 보여줄 때는 출처를 표시해야 한다.** 법적 근거는 저작권법
제37조(출처 명시 의무)이고, courtauction.go.kr(대한민국 법원 법원경매정보) 저작권 정책이 재사용을
허락하는 조건도 이것 하나다.

아래 응답에는 고정 문구를 담은 `source` 메타 필드를 함께 내려준다.

```json
{
  "source": {
    "name": "대한민국 법원 법원경매정보",
    "url": "https://www.courtauction.go.kr",
    "notice": "출처: 대한민국 법원 법원경매정보(courtauction.go.kr). 경매 참여 판단의 정본은 관할 법원 공고입니다."
  }
}
```

- 대상 경로: `/search`, `/case/{case_no}`, `/courts`, `/changes`, `/snapshots`,
  `/items/{court}/{case_no}/{item_no}/{history,rounds,rights}`.
- 예외 경로: `/items/{court}/{case_no}/{item_no}/rights`는 필드명이 이미 `source`를 분석 근거
  출처 문자열로 쓰고 있다. 이 응답에서는 같은 메타를 `data_source` 필드에 담아 보낸다.
- 같은 값이 응답 헤더 `X-Data-Source: courtauction.go.kr`로도 실린다. 본문을 파싱하지 않고
  헤더만 보고도 출처를 확인할 수 있다.
- 고객 화면에는 `source.notice` 문구(또는 동등한 한국어 안내)를 사람이 읽을 수 있는 자리에 그대로
  노출해야 한다. 숨기거나 API 응답에서만 존재하고 화면에 없으면 출처 표시 의무 위반이다.

## 8. 함께 보는 문서

- [OpenAPI](../../openapi/auction.json) 기계 판독용 정본
- [SLA](sla.md) 서비스 수준과 이용 조건

## 목록·사건 응답의 `bid_method` / `court_dept` (2026-11-10)

검색·사건·물건 목록의 `bid_method`는 입찰방법(기일입찰·기간입찰)이다. 목록 수집원(법원 목록 JSON)은 입찰방법을 주지 않고 담당 경매계(`경매4계`)만 주므로, 그 값은 `court_dept`로 내고 `bid_method`는 null이다. 입찰방법은 상세(`detail.bid_method`)에서 확인한다. 상세의 `court_dept`는 전부터 담당 경매계였다.
