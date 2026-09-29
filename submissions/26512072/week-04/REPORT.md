# Week 04 — 같은 협상, 세 가지 메시지 형식 (free / tagged / structured)

buyer 1명과 seller 1명이 FIPA 협상 행위 네 가지(propose, accept-proposal, reject-proposal, refuse)로 가격을 협상한다. 시나리오, 역할 프롬프트, 모델, temperature, 턴 한도를 고정하고 메시지 형식만 바꿔 조건마다 시나리오 6개를 3회씩, 총 54개 에피소드를 돌렸다.

## 1. 설정

| 항목 | 값 |
|---|---|
| Provider | 로컬 vLLM 0.17.1, OpenAI 호환 API (`http://127.0.0.1:8011/v1`) |
| Model | `Qwen/Qwen3.8-27B` (bf16, Hugging Face revision `1d4bf0f2`, week-03과 동일) |
| Hardware | NVIDIA RTX PRO 6000 Blackwell 96GB 1장, `--max-model-len 8192` |
| Temperature | 0.2 (에이전트와 reader 모두), max_tokens 200, `enable_thinking=false` |
| 턴 한도 | 8 메시지 (끝나지 않으면 `open`) |
| 시나리오 | `scenarios.json` 6개. 거래 가능 3개(120/150, 30/45, 40/40), 불가능 3개(90/70, 220/150, 260/200). 실행 전에 커밋(`b13667f`) |

**역할 문단 (세 조건 공통)**

- buyer: `You are the buyer of {item}, negotiating the price with the seller. Your private limit: you can pay at most {limit} dollars. Never agree to a price above {limit}. Do not reveal your limit. Try to pay as little as possible, but a deal within your limit is better than no deal. You speak first.`
- seller: `You are the seller of {item}, negotiating the price with the buyer. Your private limit: you can accept at least {limit} dollars. Never agree to a price below {limit}. Do not reveal your limit. Try to sell for as much as possible, but a deal within your limit is better than no deal.`
- 공통: `Four acts are available: propose (offer a price), accept-proposal (agree to the other side's last price, which ends the negotiation with a deal), reject-proposal (decline the last price and keep negotiating), refuse (leave the negotiation for good, no deal). Every message you send performs exactly one of these acts. Prices are whole numbers of dollars.`

**형식 문단 (조건마다 이것만 다름)**

| 조건 | 형식 문단 | 프로토콜 계층 |
|---|---|---|
| free | `Write your message as one or two plain English sentences.` | reader가 모든 메시지에 performative와 price를 붙인다 |
| tagged | `Start your message with exactly one performative tag in parentheses, one of (propose), (accept-proposal), (reject-proposal), (refuse), then write one plain English sentence.` | 정규식 `^\s*\((propose\|accept-proposal\|reject-proposal\|refuse)\)`로 태그를 읽고, propose일 때만 reader로 가격을 읽는다 |
| structured | `Reply with exactly one JSON object and nothing else: {"performative": "propose" \| "accept-proposal" \| "reject-proposal" \| "refuse", "content": {"price": <whole number or null>}}.` | JSON 파서. 모델 호출 없음 |

**reader 프롬프트 (free, tagged 공통)**: 대화 전체를 `[buyer] ...`/`[seller] ...` 줄로 넘기고 마지막 메시지만 라벨링한다.

```
You are an observer reading a price negotiation between a buyer and a seller. Label the LAST message only. Reply with exactly one JSON object and nothing else: {"performative": "propose" | "accept-proposal" | "reject-proposal" | "refuse", "price": <whole number or null>}. propose = the speaker offers a price; accept-proposal = the speaker agrees to the other side's last price, ending with a deal; reject-proposal = the speaker declines the last price and keeps negotiating; refuse = the speaker leaves the negotiation for good. price = the price the speaker of the last message offers or agrees to, as a whole number, or null if the message names no such price.
```

**프로토콜 규칙** (`negotiate.py`)

- buyer가 먼저 말한다. chat template에 user 턴이 필요해서 buyer의 첫 턴 앞에만 고정 문장 `(The negotiation starts now. Send your first message to the seller.)`을 넣는다. 세 조건이 같다.
- 자기 메시지는 assistant 턴, 상대 메시지는 user 턴으로 쌓는다.
- propose면 그 가격을 말한 쪽의 마지막 가격으로 기록한다. accept-proposal이면 상대의 마지막 가격으로 `deal`이 된다. 기록된 상대 가격이 없으면 거래로 치지 않고 계속한다. refuse면 `no_deal`이다.
- 읽지 못한 메시지는 `format_errors`에 세고, 메시지는 그대로 상대에게 전달한다.
- `violation`은 거래 가격이 reserve 미만이거나 budget 초과인 경우다.
- `correct`는 두 경우에 1이다.
  - 거래 가능 시나리오: `deal`이면서 violation이 없을 때
  - 거래 불가능 시나리오: `no_deal`일 때 (`open`은 결렬 선언이 아니므로 0으로 쳤다)

**실행 방법**

```bash
# 1. 모델 서버
bash serve_model.sh            # vllm serve Qwen/Qwen3.8-27B --revision 1d4bf0f2... --port 8011

# 2. 실험 (이 디렉터리에서). 기본값이 위 표의 설정이다.
python run.py --condition free --repeats 3
python run.py --condition tagged --repeats 3
python run.py --condition structured --repeats 3

# 3. 검사
python ../../../scripts/check_week04.py .
```

다른 OpenAI 호환 서버는 `OPENAI_BASE_URL`, `OPENAI_API_KEY`, `AGENT_MODEL`로 바꾼다. `results.csv`에 이미 있는 (run, scenario)는 건너뛰므로, 재현할 때는 `--results`와 `--logs`에 새 경로를 준다.

## 2. 결과

| 조건 | correct / 18 | deal, no_deal, open | violation | 평균 turns | format errors | reader calls |
|---|---|---|---|---|---|---|
| free | 9 | 9, 0, 9 | 0 | 6.72 | 0 | 121 |
| tagged | 0 | 6, 0, 12 | 6 | 7.44 | 0 | 20 |
| structured | 7 | 7, 0, 11 | 0 | 6.67 | 0 | 0 |

- 거래 불가능 시나리오 27개 에피소드는 세 조건 모두 전부 `open`이었다. 어떤 에이전트도 refuse를 보내지 않았고, reader도 refuse 라벨을 한 번도 붙이지 않았다.
- 거래 가능 시나리오에서 조건별 결과는 이렇다.
  - free: 9/9 모두 정상 거래
  - structured: 7/9 거래, 40/40 시나리오 2회는 8턴 안에 가격이 맞지 않아 `open`
  - tagged: 거래 6건이 전부 violation, 나머지 3건은 `open`
- 크래시와 format error는 0건이다. structured에서 JSON 뒤에 문장이 붙은 메시지도 없었다.

**에피소드 전체 (results.csv)**

| run | condition | scenario | deal_possible | outcome | price | correct | violation | turns | format_errors | reader_calls | note |
|---|---|---|---|---|---|---|---|---|---|---|---|
| free-01 | free | 1 | 1 | deal | 120 | 1 | 0 | 6 | 0 | 6 | agent_calls=6; tokens=3372 |
| free-01 | free | 2 | 1 | deal | 30 | 1 | 0 | 4 | 0 | 4 | agent_calls=4; tokens=1846 |
| free-01 | free | 3 | 1 | deal | 40 | 1 | 0 | 6 | 0 | 6 | agent_calls=6; tokens=3375 |
| free-01 | free | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=4879 |
| free-01 | free | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=5550 |
| free-01 | free | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=5146 |
| free-02 | free | 1 | 1 | deal | 120 | 1 | 0 | 6 | 0 | 6 | agent_calls=6; tokens=3334 |
| free-02 | free | 2 | 1 | deal | 35 | 1 | 0 | 5 | 0 | 5 | agent_calls=5; tokens=2657 |
| free-02 | free | 3 | 1 | deal | 40 | 1 | 0 | 6 | 0 | 6 | agent_calls=6; tokens=3345 |
| free-02 | free | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=4999 |
| free-02 | free | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=5297 |
| free-02 | free | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=4810 |
| free-03 | free | 1 | 1 | deal | 120 | 1 | 0 | 6 | 0 | 6 | agent_calls=6; tokens=3272 |
| free-03 | free | 2 | 1 | deal | 30 | 1 | 0 | 4 | 0 | 4 | agent_calls=4; tokens=1913 |
| free-03 | free | 3 | 1 | deal | 40 | 1 | 0 | 6 | 0 | 6 | agent_calls=6; tokens=3333 |
| free-03 | free | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=4798 |
| free-03 | free | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=5206 |
| free-03 | free | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 8 | agent_calls=8; tokens=5048 |
| tagged-01 | tagged | 1 | 1 | deal | 100 | 0 | 1 | 6 | 0 | 1 | agent_calls=6; tokens=2047 |
| tagged-01 | tagged | 2 | 1 | deal | 25 | 0 | 1 | 8 | 0 | 1 | agent_calls=8; tokens=2959 |
| tagged-01 | tagged | 3 | 1 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=3014 |
| tagged-01 | tagged | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=2819 |
| tagged-01 | tagged | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=3045 |
| tagged-01 | tagged | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=2890 |
| tagged-02 | tagged | 1 | 1 | open |  | 0 | 0 | 8 | 0 | 2 | agent_calls=8; tokens=3263 |
| tagged-02 | tagged | 2 | 1 | deal | 25 | 0 | 1 | 6 | 0 | 1 | agent_calls=6; tokens=2039 |
| tagged-02 | tagged | 3 | 1 | deal | 20 | 0 | 1 | 8 | 0 | 1 | agent_calls=8; tokens=2874 |
| tagged-02 | tagged | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=2841 |
| tagged-02 | tagged | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=3062 |
| tagged-02 | tagged | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=2947 |
| tagged-03 | tagged | 1 | 1 | deal | 100 | 0 | 1 | 6 | 0 | 1 | agent_calls=6; tokens=2016 |
| tagged-03 | tagged | 2 | 1 | deal | 25 | 0 | 1 | 4 | 0 | 1 | agent_calls=4; tokens=1272 |
| tagged-03 | tagged | 3 | 1 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=2851 |
| tagged-03 | tagged | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 2 | agent_calls=8; tokens=3229 |
| tagged-03 | tagged | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=2876 |
| tagged-03 | tagged | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 1 | agent_calls=8; tokens=2861 |
| structured-01 | structured | 1 | 1 | deal | 120 | 1 | 0 | 4 | 0 | 0 | agent_calls=4; tokens=1157 |
| structured-01 | structured | 2 | 1 | deal | 30 | 1 | 0 | 4 | 0 | 0 | agent_calls=4; tokens=1142 |
| structured-01 | structured | 3 | 1 | deal | 40 | 1 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2778 |
| structured-01 | structured | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2768 |
| structured-01 | structured | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2828 |
| structured-01 | structured | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2804 |
| structured-02 | structured | 1 | 1 | deal | 120 | 1 | 0 | 4 | 0 | 0 | agent_calls=4; tokens=1205 |
| structured-02 | structured | 2 | 1 | deal | 30 | 1 | 0 | 4 | 0 | 0 | agent_calls=4; tokens=1142 |
| structured-02 | structured | 3 | 1 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2539 |
| structured-02 | structured | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2768 |
| structured-02 | structured | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2588 |
| structured-02 | structured | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2996 |
| structured-03 | structured | 1 | 1 | deal | 120 | 1 | 0 | 4 | 0 | 0 | agent_calls=4; tokens=1205 |
| structured-03 | structured | 2 | 1 | deal | 30 | 1 | 0 | 4 | 0 | 0 | agent_calls=4; tokens=1142 |
| structured-03 | structured | 3 | 1 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2778 |
| structured-03 | structured | 4 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2768 |
| structured-03 | structured | 5 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2588 |
| structured-03 | structured | 6 | 0 | open |  | 0 | 0 | 8 | 0 | 0 | agent_calls=8; tokens=2804 |

## 3. FIPA-ACL과 세 조건 비교

| 항목 | FIPA-ACL (2002) | free | tagged | structured |
|---|---|---|---|---|
| illocutionary force의 위치 | 필수 필드 `performative` | 문장 속 암시. reader가 추론 | 맨 앞 태그 `(propose)` | JSON 필드 `performative` |
| content 언어 | 선언된 content language(FIPA-SL 등) + ontology | 평문 영어 | 평문 영어 한 문장 | `{"price": int\|null}` 하나 |
| content 해석 주체 | 받는 에이전트 (공유 ontology로) | reader LLM | 행위는 정규식, propose 가격만 reader | 파서 |
| 대화 종료 | interaction protocol의 상태(accept, refuse, 타임아웃 `reply-by`) | reader가 accept/refuse로 읽을 때, 아니면 8턴 | 태그가 accept/refuse일 때, 아니면 8턴 | 필드가 accept/refuse일 때, 아니면 8턴 |
| sincerity 보장 | 규범으로 전제(FP), 강제 수단 없음 | system prompt의 한도 지시뿐 | 같음. 태그와 본문이 어긋나도 확인 안 함 | 같음. 필드 값만 믿음 |
| 메시지 하나 읽는 비용 | 파서 + 의미론 추론 | reader 호출 1회 (121/121 메시지) | propose일 때만 1회 (20회) | 0 |
| 실패 방식 | 의미론 검증 불가, ontology 불일치 | 모호한 문장의 오라벨(재제안을 reject로), 호출 비용 | 태그와 본문의 불일치: reject 태그 뒤의 역제안 가격을 놓침 → 낡은 가격으로 거래 | 가격 필드로만 말하므로 협상이 느리고, 결렬 선언 없음 |

## 4. 해석

- **free:** reader가 FIPA가 말한 "받는 쪽의 해석"을 모델 호출로 대신했다. 비용은 121회로 가장 컸지만 가격을 정확히 따라갔다.
  - "I can't accept $65, but I can offer you a final price of $100."을 `propose 100`으로 읽는 식으로, 거절과 역제안이 섞인 문장에서도 역제안 쪽을 propose로 잡았다.
  - 그래서 거래 가능 9건이 모두 한도 안에서 성립했다.
  - reader 라벨이 문장과 어긋난 줄도 있었다. free-02에서 "I can't go that high. I'll stick with my offer of $150."은 재제안인데 `reject-proposal, 150`으로 읽혔다. 기록된 가격이 이미 150이라 결과는 바뀌지 않았다.
- **tagged:** 명시적 performative가 오히려 비용이 됐다. 모델은 태그를 한 개만 붙이라는 지시를 지켰지만, 역제안을 reject-proposal 태그 뒤 본문에 넣었다. tagged의 reject-proposal 메시지 144개 가운데 60개에 금액이 있었다.
  - tagged-01 시나리오 2의 흐름: `[buyer] (propose) I can offer you $25`로 시작했다. 이후 "(reject-proposal) ... I can offer you $40"까지 여러 번 역제안이 오갔지만, 정규식에는 전부 거절로만 보였다. 결국 `[seller] (accept-proposal) I accept your offer of $40`이 왔고, 프로그램은 첫 제안 25로 거래를 기록했다(reserve 30 → violation).
  - 이런 식의 violation 6건은 모두 프로토콜 계층이 기록한 가격에서 나왔다. seller의 accept 문장에 적힌 합의 가격은 6건 모두 한도 안이었다($120, $40, $32, $40, $120, $35).
  - reader 호출은 121회에서 20회로 줄었다. 대신 행위 태그와 내용이 따로 놀 때 그것을 잡을 장치가 없었다.
- **structured:** content까지 숫자 필드로 묶으니 이 불일치가 사라졌다. violation 0, reader 0이다. 대신 seller는 가격을 담지 않은 reject-proposal만 반복하는 경우가 많았다. 그러면 buyer 혼자 가격을 올려야 해서, 40/40 시나리오에서는 8턴 안에 40에 도달하지 못했다(structured-02: propose 20 → 25 → … → 45, 그 사이 seller 60, 50).
- **어떤 형식도 바꾸지 못한 것:** 거래 불가능 시나리오의 결말이다. 27개 에피소드 모두 양쪽이 가격을 좁히다 8턴을 채웠고(free-01 시나리오 4: "I can't accept $70, but I can offer you $95."), refuse는 한 번도 나오지 않았다. 떠나는 결정은 메시지 형식의 문제가 아니라 에이전트가 협상을 포기할 판단을 하는가의 문제라서, performative 필드가 있어도 생기지 않았다.
- **참조 실행과 다른 점:** 이번 실행에서는 buyer가 매번 가격을 먼저 제시해서, 참조 실행처럼 첫 질문 메시지를 refuse로 읽는 현상은 나오지 않았다. 역할 문단의 "You speak first."와 모델 차이 때문으로 보인다.
