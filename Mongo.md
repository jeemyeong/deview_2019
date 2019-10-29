https://deview.kr/data/deview/2019/presentation/[214]Deview2019_%EB%B0%95%EA%B7%BC%EB%B0%B0.pdf

# Why MongoDB?

- FAST + SCALABLE + HIGHLY AVAILABLE = MongoDB
- 네이버 통합검색의 1초 rule, 현재 시카고 플랫폼 평균 응답속도 10ms미만.
- 일 평균 4~5억건 이상 서버호출, 초당 6천건 수준, 몽고DB는 초당 만건 이상

# 몽고 DB속도 올리기 - Index

## MongoDB Index 이해하기

- MongoDB는 컬렉션당 최대 64개의 인덱스만 생성 가능.
- 너무 많은 인덱스를 추가하면,오히려 side effects가 발생한다.
  - Frequent Swap.
  - Write performance 감소 -> read에도 영향을 미친다.
  - 인덱스가 너무 많으면 쿼리가 들어왔을때, 쿼리 플래너가 어떤 쿼리를 실행할지 결정하는 순간에, 엉뚱한 인덱스를 고를 수도 있다.

## Index Prefix를 사용하자

- Compound index
- `db.nbaGame.createIndex({teamCode:1, season:1, gameDate:1})`
- beginning subset이 지원이 되어야 한다.
- `db.nbaGame.find({teamCode:“gsw”,gameDate:“20180414”});`
- compund index의 지원을 받는데, teamCode에 한해서만 지원을 받는다. (Half Covered!)

## 멀티 소팅

- Sort key들은 반드시 인덱스와 같은 순서로 나열 되어야만 한다.
- `db.nbaGame.createIndex({gameDate: 1,teamName: 1});`
- `db.nbaGame.find().sort({gameDate:1, teamName:1});` //covered
- `db.nbaGame.find().sort({teamName:1, gameDate:1});` //notcovered

- single-field 인덱스의 경우, 소팅 방향을 걱정할 필요 없음.
- 그러나 compound 인덱스의 경우, 소팅 방향이 중요함. (멀티소팅의 방향은 반드시 index key pattern또는 index key pattern의 inverse와 일치 해야함)
- Covered
```javascript
db.nbaGame.createIndex({gameDate:1,teamName:1});
db.nbaGame.find().sort({gameDate:1,teamName:1 });
db.nbaGame.find().sort({gameDate:-1,teamName:-1 }); //theinverse
db.nbaGame.find().sort({gameDate:1}); //index prefix
db.nbaGame.find().sort({gameDate:-1});
```
- Not Covered
```javascript
db.nbaGame.createIndex({gameDate:1,teamName:1});
db.nbaGame.find().sort({gameDate:-1,teamName:1 });//notmatched
db.nbaGame.find().sort({gameDate:1,teamName:-1 });
db.nbaGame.find().sort({teamName :1});//notindexprefix
db.nbaGame.find().sort({teamName :-1});
```

# 몽고 DB속도 올리기 - ^Index

## 하나의 컬렉션을 여러 컬렉션으로 나누자

- 하나의 컬렉션이 너무 많은 문서를 가질 경우,인덱스 사이즈가 증가하고 인덱스 필드의 cardinality가 낮아질 가능성이 높다
- 이는 lookup performance에 악영향을 미치고 slow query를 유발한다.
- 한 컬렉션에 너무 많은 문서가 있다면 반드시 컬렉션을 나눠서 query processor가 중복되는 인덱스 key를 lookup하는 것을 방지해야 한다.

## 쓰레드를 이용해 대량의 Document를 upsert
- 여러 개의 thread에서 BulkOperation으로 많은 document를 한번에 write.
- documenttransaction과 arelation은 지원 안됨.
- Writingtime을 획기적으로 개선 가능.

## MongoDB 4.0으로 업그레이드
- 몽고DB 4.0이전에는 non-blocking secondary read 기능이 없었음.
- Write가 primary에 반영 되고 secondary들에 다 전달 될때까지 secondary는 read를 block해서 데이터가 잘못된 순서로 read되는 것을 방지함.
- 그래서 주기적으로 높은 global lock acquire count가 생기고 read성능이 저하됨.
- 몽고DB4.0부터는 data timestamp와 consistent snapshot을 이용해서 이 이슈를 해결함.
- 이게 non-blocking secondary read.

# 미운 Index

## Slow Queries..

- `db.concert.count({serviceCode:{$in:[2,3]},startDate:{$lte:20190722},endDate:{$gte:20190722});`
- => `{serviceCode:1,weight:1}` 인덱스를 이용 (key examined:88226)
  - key examined 가 8000을 넘어가면 100ms가 넘게 된다.
- 123개의 문서를 찾기 위해 8만개의 문서를 탐색 한다???
- serviceCode가 2,3인 8만개의 인덱스 키를 훑어보면서, 메모리에서 startDate,endDate의 범위 검색을 실행.
- 사실 startDate,endDate부터 실행하고 serviceCode를 뒤지면 123개만 탐색하면 가능함..근데 왜..?
- hint를 이용해 강제로 인덱스를 태워봤더니….
```javascript
db.concert.count({
  serviceCode:{$in:[2,3]}, startDate:{$lte:20190722},endDate:{$gte:20190722},
  {hint:”serviceCode_1_startDate_1_endDate_e_1”}
);
```
- 270ms->3ms

## Index가 미운짓을 하는 이유

- 왜 자꾸 엉뚱한 인덱스를 타는걸까? 에서 의문시작.
- 몽고 DB는 쿼리가 들어왔을때 어떻게 최선의 쿼리플랜을 찾을까?
- 여러개의 쿼리 플랜들을 모두 실행해 볼 수도 없고…

- 답은 QueryPlanner 에서 찾을 수 있었음.
- 이전에 실행한 쿼리 플랜을 **캐싱** 해놓음.
- 캐싱된 쿼리 플랜이 없다면 가능한 모든 쿼리 플랜들을 조회해서 **첫 batch(101개)를 가장 좋은 성능으로** 가져오는 플랜을 다시 캐싱함.
- 성능이 너무 안좋아지면 위 작업을 반복.
- * 캐싱의 사용 조건은 같은 QueryShape일 경우
- 결국 { serviceCode:1, weight:1 } 인덱스를 사용하더라도
```javascript
db.concert.count({
  serviceCode:{$in:[2,3]},startDate:{$lte:20190722},endDate:{$gte:20190722}
});
```
- 이 쿼리는 전체 조회가 필요해 느릴 수 있지만 (keyexamined:88226)
```javascript
db.concert.find({
  serviceCode:{$in:[2,3]},startDate:{$lte:20190722},endDate:{$gte:20190722}
}).limit(101);
```
- 101개의 오브젝트만 가져온다면 순식간에 가져옴.
- (101개를 찾자 마자 return 하므로 keyexamined:309)

- 그래서 QueryPlanner가 count() 쿼리의 QueryShape로 첫번째 batch를 가져오는 성능을 테스트 했을때
- {serviceCode:1,weight:1} 같은 엄한 인덱스가 제일 좋은 성능을 보일 수 있는거임.

- * 만약에 동점이 나올 경우, in-memorysort를 하지 않아도 되는 쿼리 플랜을 선택함.

## 솔로몬의 선택
- Hint이용 VS 엄한 인덱스를 지우기
- 확실한 쿼리플랜 보장 vs 데이터에 따라 더 효율적인 인덱스가 생기면 자동 대응
- 더 효율적인 인덱스가 생겨도 강제 고정 vs 또다른 엄한 케이스가 생길 수 있음
- 데이터 분포에 대한 정보를 계속 follow 해야함 vs 삭제로 인해 영향 받는 다른 쿼리가 생길 수 있음
- 32MB넘는 응답결과를 sort해야 할 경우 에러

- hint를 사용했더니, 장애가 발생. 다른데서 슬로우 쿼리가 생김

# QA
- 몽고 4.0으로 가기가 아직 좀 두려운데요 혹시 업그레이드 시 이슈는 없었나요?
  - No pain No gain
