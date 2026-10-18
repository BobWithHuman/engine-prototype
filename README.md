# WithHuman 엔진 0.1.0

- 엔진 커밋: `653ba1a`
- 이미지: 서버 CPU에 맞는 것 하나를 씁니다. `amd64` / `arm64`

## 구성

```
Claude Code ──훅 이벤트──▶ 엔진 :8080 ──▶ Neo4j :7474 / :7687
```

- 편의성을 위해 AgentGateway를 사용하지 않고 훅 스크립트가 엔진으로 바로 보냅니다.

## 들어 있는 것

| 파일 | 내용 |
| --- | --- |
| `withhuman-engine-0.1.0-amd64.tar.gz` | 엔진 도커 이미지 (x86_64용) |
| `withhuman-engine-0.1.0-arm64.tar.gz` | 엔진 도커 이미지 (ARM용) |
| `compose.yaml` | 엔진 + Neo4j |
| `.env.example` | 설정 예시 |
| `client/withhuman-hook.sh`, `client/settings.json` | Claude Code 쪽 훅 스크립트와 설정 |

## 1. 서버에서 띄우기

```sh
uname -m                                              # x86_64 이면 amd64, aarch64 이면 arm64
docker load -i withhuman-engine-0.1.0-amd64.tar.gz   # 이미지 불러오기 (ARM 서버면 -arm64.tar.gz)
cp .env.example .env                                  # 설정 파일 만들기
#  .env 에 JEV_API_KEY, WITHHUMAN_NEO4J_PASSWORD 를 넣으세요
docker compose up -d                                  # 엔진(8080) + Neo4j(7474, 7687)
docker compose logs -f engine                         # 엔진 로그
```

- 기록은 `./data`(이벤트 원문 `events.jsonl`, 세션별 `sessions/<id>.json`)와 Neo4j(`./neo4j-data`)에 남습니다.
- 끄기: `docker compose down` (기록은 남음). 기록까지 지우려면 `./data`, `./neo4j-data`를 지웁니다.

**모드** (`.env`의 `WITHHUMAN_MODE`, 기본 `shadow`)
- `shadow`: 판정까지 하고 기록, 막지도 묻지도 않음 ← 이번 시험용
- `record`: 그래프만 기록, 판정 없음
- `enforce`: 실제로 막음 (쓰지 않음)

## 2. Claude Code에 훅 붙이기

엔진을 쓸 프로젝트(Claude Code를 실행하는 폴더)에서:

```sh
mkdir -p .claude
cp client/withhuman-hook.sh .claude/
cp client/settings.json .claude/settings.json      # 기존 settings.json이 있으면 "hooks" 부분만 합치세요
export WITHHUMAN_URL=http://<엔진 서버 주소>:8080   # 엔진이 같은 PC면 생략 (기본 localhost:8080)
claude
```

- 훅 스크립트는 `bash`, `curl`, `git`만 씁니다.
- 엔진이 5초 안에 답하지 않거나 꺼져 있어도 호출은 그냥 실행됩니다.

## 3. 그래프 보기

1. 브라우저에서 `http://<서버>:7474` 접속
2. Connect URL `neo4j://<서버>:7687`, 사용자 `neo4j`, 비밀번호는 `.env`의 값
3. 아래 쿼리의 `<세션id>`에 세션 id를 넣어 실행

```cypher
// 세션 목록 (요청 글과 함께)
MATCH (r:Request) RETURN r.session_id, left(r.text, 80) AS request ORDER BY r.observed_at DESC;

// 1. 순서 그래프
MATCH p=(a {session_id:'<세션id>'})-[:NEXT]->() RETURN p;

// 2. 원인 그래프
MATCH p=(a {session_id:'<세션id>'})-[:CAUSE]->() RETURN p;

// 3. 출처 그래프
MATCH p=(a {session_id:'<세션id>'})-[:LINK]->() RETURN p;

// 셋 다 (소속만 빼고)
MATCH p=(a {session_id:'<세션id>'})-[:CAUSE|LINK|NEXT]->() RETURN p;

// 호출별 판정 표
MATCH (c:Call {session_id:'<세션id>'}) RETURN c.node_id, c.tool_name, c.risk, c.verdict, c.rule ORDER BY c.seq;
```

막기·묻기로 기록된 호출만 모아 보기:

```cypher
MATCH (c:Call) WHERE c.verdict IN ['block','ask']
RETURN c.session_id, c.node_id, c.tool_name, c.verdict, c.rule, left(c.input, 120) AS input
ORDER BY c.session_id, c.seq
```

## 주의

- Jev 키는 묶음에 들어 있지 않습니다. 따로 받은 키를 `.env`의 `JEV_API_KEY=`에 넣으면 됩니다. 키가 없어도 엔진은 돌지만 제대로 그래프가 이어지지 않습니다..
