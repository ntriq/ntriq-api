# 법원경매 MCP 빠른 시작

REST 로 쓰시던 것과 **같은 키, 같은 데이터, 같은 한도**를 AI 에이전트에서 도구로 부를 수 있습니다. MCP 이용료는 따로 받지 않습니다.

MCP 는 데이터를 도구로 꺼내 줄 뿐 언어모델을 대신 부르지 않습니다. 모델 사용료는 쓰시는 AI 클라이언트 쪽 계약을 따릅니다.

## 접속

| 항목 | 값 |
|---|---|
| 주소 | `https://api.ntriq.co.kr/mcp` |
| 전송 | Streamable HTTP |
| 인증 | `Authorization: Bearer <발급키>` |

REST 와 달리 헤더 이름이 `Authorization` 입니다. 키는 서버 쪽 환경변수나 비밀 저장소에 두시고 프론트엔드 번들에 넣지 마십시오.

초기화와 `tools/list`, `tools/call` 이 모두 같은 주소를 씁니다. MCP 클라이언트가 초기화를 알아서 하므로 직접 부르실 일은 보통 없습니다.

**프로토콜 호출도 한도에 포함됩니다.** REST 와 MCP 가 한 통의 한도를 나눠 씁니다.

## 도구

| 도구 | 하는 일 |
|---|---|
| `search_auction_items` | 조건으로 찾고 페이지를 넘깁니다 |
| `get_auction_item` | 법원과 사건번호, 물건번호로 한 건을 읽습니다 |
| `get_auction_rounds` | 기일 회차를 읽습니다 |
| `get_auction_history` | 상태 변경 이력을 읽습니다 |
| `get_auction_rights` | 권리분석과 입력 가용성을 읽습니다 |
| `get_auction_changes` | `since` 와 커서로 바뀐 것만 따라갑니다 |
| `list_auction_snapshots` | 스냅샷 목록과 완전성 보고를 읽습니다 |
| `auction_health` | 신선도와 상태를 확인합니다 |

인자와 한도, 오류 코드는 [명세서](spec.md)의 같은 이름 엔드포인트와 같습니다.

## 직접 불러 보기

클라이언트 없이 확인만 하실 때 씁니다.

```bash
curl --fail-with-body "https://api.ntriq.co.kr/mcp" \
  -H "Authorization: Bearer $NTRIQ_KEY" \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/call",
           "params":{"name":"search_auction_items",
                     "arguments":{"sido":"부산","limit":2}}}'
```

## 응답을 다룰 때

HTTP 상태와 `result.isError` 를 **둘 다** 보십시오. 전송이 성공해도 도구가 오류를 담아 올 수 있습니다. 실제 데이터는 `result.content[0].text` 안의 JSON 입니다.

전송이 성공했다고 해서 그 물건의 원천 자료가 전부 모였다는 뜻은 아닙니다. 권리분석의 `input_availability` 와 스냅샷의 `completeness` 를 함께 보십시오.

## 알려진 제약

- 체험 키는 한 번에 20건까지 읽습니다. 21건을 요청하면 오류가 납니다.
- 일부 AI 클라이언트는 OAuth 로만 외부 MCP 를 붙입니다. 그런 클라이언트의 연결은 아직 확인하지 못했습니다.
