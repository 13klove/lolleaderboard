# Leaderboard

## 요약

- game leaderboard 아키텍처를 작성하고 기능 개발합니다.

## 요구사항

- 게임이 끝나고 player 들의 leaderboard를 구성해야 합니다.
- player가 하루동안 획득한 score를 기반으로 leaderboard를 구성해야 합니다.
- Season 별로 leaderboard를 구성해야 합니다.
- score는 게임에서 획득한 point를 기반으로 합니다.
- leaderboard에는 사용자 id, 점수, 순위가 노출 되어야 합니다.
- 사용자 id로 점수, 순위를 조회할 수 있어야 합니다.

- 리더보드는 총 3가지를 가집니다.
    - PostMatch Leaderboard -> game 직후 player 들의 leaderboard
    - Daily Leaderboard -> 하루동안 사용자의 leaderboard
    - Season Leaderboard -> season 별 leaderboard

## 아키텍처

![arch.png](arch.png)

## 설명

- 사용자의 요청은 leaderboard service 를 통해서만 가능합니다. point 적재, 랭킹 조회를 담당합니다.
- kafka 에 publish는 leaderboard batch 를 통해 outbox pattern 을 적용합니다.
- consumer 는 MatchUserPoint 를 redis 에 적재하고 leaderboardDB에 적재합니다.
- fail 인 경우 dlq를 통해서 관리합니다.
- cqrs pattern 을 적용해 조회는 1차 집계된 데이터를 통해서 집계합니다. 1차는 redis hit 하지 못하면 leaderboardDB

## 추가 고려사항

- consumer 에서는 transaction 이 실패하는 경우는 어떻게 처리해야하는지 조금 더 고민이 필요합니다.
- aggregation 을 어떻게 해야할지 고민이 필요합니다.
- checkpoint pattern을 고민해 본다.
- outbox 를 대신할 cdc를 고민한다.

## 시퀀스다이어그램

### point 적재 flow
```mermaid
sequenceDiagram
  participant user
  participant leaderboard service
  participant leaderboardDB
  participant leaderboard batch
  participant kafka queue
  participant leaderboard consumer
  participant redis
  participant kafka dlq
  
  user ->> leaderboard service: request POST /v1/matches/{matchedId}/users/{userId}/point
  leaderboard service ->> leaderboardDB: point, outbox 적재
  leaderboard service ->> user: response
  leaderboard batch ->> leaderboardDB: outbox list 조회
  loop
    leaderboard batch ->> kafka queue: publish
  end
  leaderboard batch ->> leaderboardDB: remove outbox data
  leaderboard consumer ->> kafka queue: poll events
  alt
    leaderboard consumer ->> redis: data 적재
    leaderboard consumer ->> leaderboardDB: data 적재
  else fail
    leaderboard consumer ->> kafka dlq: publish
  end
```

### ranking 조회 flow

```mermaid
sequenceDiagram
  participant user
  participant leaderboard service
  participant leaderboardDB
  participant redis
  
  user ->> leaderboard service: request GET
  alt
    leaderboard service ->> redis: data 조회
  else miss
    leaderboard service ->> leaderboardDB: data 조회
    leaderboard service ->> redis: data 적재
  end
  leaderboard service ->> user: response
```

### point 집계 flow

- todo

## 사용기술

- library : spring boot, kotlin, jpa, redisson, kafka
- infra : mysql, redis, kafka

## Domain

## MatchUserPoint

- 사용자 point만 적재합니다.
- matched_id를 기준으로 사용자가 얼마나 포인트를 쌓았는지 확인 합니다.
- index
    - user_id

| field      | type   | pk  | Description    | 
|------------|--------|-----|----------------|
| id         | Number | o   | id             |
| user_id    | String |     | 사용자 id         |
| matched_id | String |     | game 대전 id     |
| point      | Number |     | 획득한 point      |
| pointed_at | Long   |     | point 획득한 시간   |
| created_at | Long   |     | 생성 시간          |

## MatchUserLeaderboard

- match 된 게임 안에서 랭킹을 조회할 수 있습니다.
- matched_id를 기준으로 사용자가 얼마나 포인트를 쌓았는지 확인 합니다.
- index
    - matched_id, point
    - matched_id, user_id(uk)

| field      | type   | pk  | Description    | 
|------------|--------|-----|----------------|
| id         | Number | o   | id             |
| matched_id | String |     | game 대전 id     |
| user_id    | String |     | 사용자 id         |
| point      | Number |     | 획득한 point      |
| pointed_at | Long   |     | point 획득한 시간   |
| created_at | Long   |     | 생성 시간          |

## DailyLeaderboard

- 일별로 사용자별 랭킹을 조회할 수 있습니다.
- index
    - date, user_id(uk)
    - date, user_id, point
    - user_id, date

| field      | type     | pk  | Description  | 
|------------|----------|-----|--------------|
| id         | Number   | o   | id           |
| date       | Instant  |     | game 대전 id   |
| user_id    | String   |     | 사용자 id       |
| point      | Number   |     | 획득한 point    |
| created_at | Long     |     | 생성 시간        |
| updated_at | Long     |     | 수정 시간        |

## SeasonLeaderboard

- 시즌 사용자별 랭킹을 조회할 수 있습니다.
- index
    - season, user_id (uk)
    - season, user_id, point
    - user_id, season

| field        | type    | pk  | Description  | 
|--------------|---------|-----|--------------|
| id           | Number  | o   | id           |
| season       | String  |     | 시즌           |
| season_start | Instant |     | 시즌 시작일       |
| season_end   | Instant |     | 시즌 종료일       |
| user_id      | String  |     | 사용자 id       |
| point        | Number  |     | 획득한 point    |
| created_at   | Long    |     | 생성 시간        |
| updated_at   | Long    |     | 수정 시간        |

## OutBox

- outbox pattern 을 지원하기 위한 테이블
- index
    - id, created_at

| field          | type   | pk  | Description      | 
|----------------|--------|-----|------------------|
| id             | Number | o   | id               |
| aggregate_type | String |     | domain full name |
| aggregated_id  | Number |     | domain id        |
| payload        | json   |     | 데이터              |
| created_at     | Long   |     | 생성 시간            |

## Api

- [POST] /v1/matches/{matchedId}/users/{userId}/point
- [GET] /v1/match-leaderboard/{matchedId}/score
- [GET] /v1/daily-leaderboard/users/{userId}
- [GET] /v1/season-leaderboard/users/{userId}
- [GET] /v1/daily-leaderboard
- [GET] /v1/season-leaderboard

### [POST] /v1/matches/{matchedId}/users/{userId}/point

#### description

- game match로 획득한 point를 적립한는 api

#### query

- none

#### request body

```json
{
  "point": 4
}
```

#### response

- none

### [GET] /v1/match-leaderboard/{matchedId}/score

#### description

- match 에 참여한 player 들의 랭킹을 조회 합니다.

#### query

- none

#### request body

- none

#### response

```json
{
  "matchedId": "werwfasdfasdf",
  "rankings": [
    {
      "userId": "asdfdsa",
      "point": 1231,
      "ranking": 1
    },
    {
      "userId": "asd44sa",
      "point": 121,
      "ranking": 2
    }
  ]
}
```

### [GET] /v1/daily-leaderboard/users/{userId}

#### description

- 일별 사용자 point 와 ranking 을 조회 합니다.

#### query

- none

#### request body

- none

#### response

```json
{
  "date": "2025-03-22",
  "userId": "asdfdsa",
  "point": 1231,
  "ranking": 1
}
```

### [GET] /v1/season-leaderboard/users/{userId}

#### description

- 시즌별 사용자 point 와 ranking 을 조회 합니다.

#### query

- none

#### request body

- none

#### response

```json
{
  "season": "GOLD",
  "userId": "asdfdsa",
  "point": 1231,
  "ranking": 1
}
```

### [GET] /v1/daily-leaderboard

#### description

- 일별 랭킹을 조회 합니다.

#### query

- ?date=2025-03-22&size=100&curssor=123

#### request body

- none

#### response

```json
{
  "date": "2025-03-22",
  "rankings": [
    {
      "userId": "asdfdsa",
      "point": 1231,
      "ranking": 1
    },
    {
      "userId": "asd44sa",
      "point": 121,
      "ranking": 2
    }
  ],
  "cursor": "123123"
}
```

### [GET] /v1/season-leaderboard

#### description

- 일별 랭킹을 조회 합니다.

#### query

- ?season=GOLD&size=100&curssor=123

#### request body

- none

#### response

```json
{
  "season": "GOLD",
  "rankings": [
    {
      "userId": "asdfdsa",
      "point": 1231,
      "ranking": 1
    },
    {
      "userId": "asd44sa",
      "point": 121,
      "ranking": 2
    }
  ],
  "cursor": "123123"
}
```
