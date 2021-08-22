## 7.7 데이터 관리와 스키마 에볼루션

### 7.7.1 스키마 에볼루션

데이터 허브로 다수의 시스템과 접속하는 경우에는 종종 접속한 시스템의 데이터가 변경되었을 경우를 염두에 둬야 한다. 예를 들어 POS 시스템을 해외 업체의 패키지로 새로 교체함에 따라 단가(unit_price)의 데이터 형태가 double이 되거나, 데이터 분석의 정확도를 높이기 위해 전자상거래 사이트의 매출 데이터에 사용자 ID를 추가하거나 하는 식으로 변경했을 경우, 해당 시스템에서는 외부와 송수신하는 데이터의 스키마를 변경하고 싶은 경우가 있다.

데이터 허브를 사용한 시스템의 경우 스키마 변경은 접속원 시스템, 데이터 허브, 연결 시스템 모두에 영향을 주게 된다. 그러나 모든 시스템의 애플리케이션을 동시에 수정해서 배포하는 것은 현실적이지 않다. 따라서 시간이 경과하면 스키마가 변화할 것을 감안하여 시스템을 설계할 필요하 있다. 스키마가 '진화'하는 것을 스키마 에볼루션이라고 부른다.

데이터 허브에 국한된 것이 아니라 상업용 환경에서 데이터 파이프라인을 구축할 때 이러한 영역을 간과하는 경우가 많은데, 설계 시점에서 스키마 에볼루션까지 주의 깊게 고려하면 나중에 고생하지 않아도 된다.

### 7.7.2 스키마 호환성

스키마를 전환시킬 때 주변 시스템은 일관성을 유지하면서 계속 처리할 수 있어야 한다. 이를 실현하기 위해 대부분의 경우는 진화 전후의 스키마에 어느 정도 호환성을 갖도록 한다. 호환성은 다음과 같은 것을 고려한다.

- 후방(하위) 호환성 backward compatibility
- 전방(상위) 호환성 forward compatibility
- 완전 호환성 full compatibility

  2.를 예로 살펴보자. 판매 예측 시스템의 정밀도 향상을 목표로 향후 POS 시스템부터 연령 정보를 추가해 데이터를 연계하는 구성을 하게 되었다. POS 쪽의 시스템 수정에 앞서 판매 예측 시스템은 연령이라는 칼럼이 늘어난다는 전제로 '연령 데이터가 들어 있으면 분석에 사용'한다는 식의 시스템 수정이 발생했다. 이때 POS 쪽에서 데이터 허브에 넣는 데이터는 아직 연령 칼럼이 존재하지 않는 데이터이지만, 데이터를 수신하는 판매 예측 시스템 쪽은 연령 칼럼이 있다는 가정하에 데이터를 수신한다. 이처럼 예전 스키마의 데이터를 새로운 스키마를 사용해서도 로딩할 수 있다는 성질을 후방 호환성이라고 부른다.

반대로 POS 쪽 시스템 수정이 먼저 실시되어 데이터 허브에 들어가는 데이터가 이미 연령 칼럼이 늘어난 경우라 하더라도, 판매 예측 시스템은 아직 수정되지 않아 데이터에는 연령 칼럼이 없다는 가정에서 데이터를 불러 오는 경우도 있을 것이다. 이처럼 새로운 스키마의 데이터를 예전 스키마를 사용해서도 로드할 수 있는 성질을 전방 호환성이라고 한다.

후방 호환성과 전방 호환성을 모두 갖춘 경우를 완전 호환성이라고 한다.

### 7.7.3 Schema Registry

앞서 6장에서는 카프카에서 스키마 에볼루션을 고려할 때 Schema Registry를 사용하면 편리하다고 설명했다.

Schema Registry는 카프카 클러스터 외부에서 스키마만을 관리하는 기능을 지닌 서비스다. Schema Registry는 카프카에 포함되어 있는 것이 아니라 컨플루언트가 제공하고 있다. Schema Registry가 있는 환경에서는 프로듀서와 컨수머가 데이터 자체에 스키마를 갖게 해 카프카에 전달하는 것이 아니라, 스키마 정보를 Schema Registry에 등록하고 그때 Schema Registry에서 부여되는 ID를 직렬화된 데이터 본체에 부가해서 카프카에 전송한다. 스키마 변경이 없으면 매번 동일한 ID가 데이터에 추가되지만 스키마 변경이 있을 경우 프로듀서가 Schema Registry에 새로운 스키마를 등록하고 새 ID를 데이터에 추가한다.

이러한 구조로 되어 있기 때문에 프로듀서가 변경하려고 하는 스키마가 필요한 호환성을 충족하는지 여부를 Schema Registry에서 확인할 수 있다. 호환성을 충족하지 않는 스키마 변경은 Schema Registry에서 받아들이지 않으며 프로듀서는 카프카에 데이터를 보낼 수 없다. 이렇듯 Schema Resgistry는 스키마를 집중 관리하고 스키마 에볼루션을 올바로 실행하기 위해 사용한다.

### 7.7.4 Schema Registry 준비

여기서부터는 2에서 사용한 예를 응용하여 Schema Resgistry를 Kafka Connect에서 사용하는 흐름을 간단히 살펴보고자 한다. 단순화를 위해 소스 쪽은 POS에 있는 MariaDB만을 사용하고, 전자상거래 사이트에 있는 PostgreSQL은 사용하지 않겠다. 싱크 쪽은 판매 예측 시스템의 S3을 사용한다.

Schema Registry는 컨플루언트 플랫폼을 설치하면 이미 설치되어 있기 때문에 곧바로 사용할 수 있다. 그럼 바로 시작해보자. 상업용 환경에서 Schema Resgistry를 사용하는 경우에는 중복 구성이 필요하므로 여러 서버에서 실행한다. 여기에서는 Schema Resgistry를 브로커가 설치된 서버에 함께 설치해서 카프카 클러스터의 모든 서버(kafka-broker01,02,03)에서 실행한다. 우선은 설정 파일을 만든다.

```bash
$ sudo vim /etc/schema-registry/schema-registry.properties
```

다음과 같이 설정한다.

```shell
# kafkastore.connection.url=kafka-broker01:2181,kafka-broker02:2181,kafka-broker03:2181
kafkastore.connection.url=localhost:2181
```

Schema Resgistry를 여러 서버에서 실행하는 경우는 내부에서 자동으로 마스터를 선정한다. 이때 주키퍼를 사용하기 때문에 kafkastore.connection.url에 주키퍼 앙상블을 지정한다. 설정을 마치면 Schema Resgistry를 실행한다.

```bash
$ sudo systemctl start confluent-schema-registry
```

Schema Resgistry도 REST API 인터페이스가 있다. 예를 들어 설정 확인은 다음과 같이 한다.

```bash
# (kafka-client)$ curl -X GET http://kafka-broker01:8081/config
(kafka-client)$ curl -X GET http://localhost:8081/config
{"compatibilitylevel":"BACKWARD"}
```

후방 호환성을 보장하는 설정으로 되어 있다. 이것은 전체 설정이지만 이와는 별도로 개별적으로 다른 호환성 설정도 가능하다.

### 7.7.5 Schema Resgistry 사용

Schema Resgistry 준비가 끝났으므로 Kafka Connect에서 Schema Resgistry를 사용해보자. 먼저 Kafka Connect를 시작한다. Kafka Connect를 시작하기 전에 Schema Resgistry에 항목을 추가해도 좋지만 없으면 자동으로 추가되므로 이번에는 Kafka Connect를 시작해보자. 여기서부터의 절차는 2.와 같다. 이번에는 Kafka Connect를 시작하는 서버(kafka-broker01,02,03)에서 실행한다. 실행에 필요한 설정 파일은 Schema Resgistry를 사용하도록 수정해둔다. 2.에서 만든 것을 사용해보자.

```bash
$ cp connect-distributed-2.properties connect-distributed-2.sr.properties
$ vim connect-distributed-2.sr.properties
```

다음과 같이 설정한다.

```shell
# bootstrap.servers=kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092
bootstrap.servers=localhost:9092

group.id=connect-cluster-datahub-2-sr

key.converter=io.confluent.connect.avro.AvroConverter

# key.converter.schema.registry.url=http://kafka-broker01:8081,http://kafka-broker02:8081,http://kafka-broker03:8081
key.converter.schema.registry.url=http://localhost:8081

value.converter=io.confluent.connect.avro.AvroConverter

# value.converter.schema.registry.url=http://kafka-broker01:8081,http://kafka-broker02:8081,http://kafka-broker03:8081,
value.converter.schema.registry.url=http://localhost:8081
```

key.converter와 value.converter는 2.에서는 특별히 설정을 변경하지 않았지만 JSONConverter라는 설정이 들어 있다. 이것은 Kafka Connect가 데이터를 카프카에 넣을 때 JSON으로 직렬화됐음을 의미한다. 그런데 Schema Registry는 Avro로 사용되는 것을 전제로 하기 때문에 여기서는 Avro를 지정한다. key.converter.schema.registry.url, value.converter.schema.registry.url에는 조금 전 실행한 Schema Registry의 URL을 지정한다.

설정 파일이 완성됐으므로 Kafka Connect를 시작하자

```bash
$ connect-distributed ./connect-distributed-2-sr.properties
```

POS의 MariaDB에서 로드할 커넥터를 시작한다. 단, 이전과 같은 토픽을 사용하면 알 수 없게 되므로 커넥터명과 토픽명은 바꾸어둔다.

```bash
(kafka-client)$ echo '
{
  "name" : "load-possales-data-sr",
  "config" : {
    "connector.class" : "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url" : "jdbc:mysql://pos-data-server/pos",
    "connection.user" : "connectuser",
    "connection.password" : connectpass",
    "mode" : "incrementing",
    "incrementing.column.name" : "seq",
    "table.whitelist" : "pos_uriage",
    "topic.prefix" : "possales_sr_",
    "tasks.max" : "3"
  }
}
' | curl -X POST -d @- http://localhost:8083/connectors \
# http://kafka-broker01:8083/connectors
> --header "content-Type:application/json"
```

카프카에 들어간 데이터는 JSON이 아니라 Avro로 직렬화되어 있기 때문에 kafka-console-consumer로 봐도 사람이 이해할 수 없다. 대신 이런 경우는 kafka-avro-console-consumer를 사용한다.

```bash
(consumer-client)$ LOG_DIR=./logs kafka-avro-console-consumer \
# --bootstrap-server kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092 \
> --bootstrap-server localhost:9092
> --topic possales_sr_pos_uriage --from-beginning --property schema.registry.url=\
# http://kafka-broker01:8081,http://kafka-broker01:8082,http://kafka-broker03:8081
> http://localhost:8081
```

schema.registry.url에 Schema Registry를 지정한다. 이것이 없으면 Avro를 역직렬화할 수 없기 때문이다. 또한 kafka-avro-console-consumer는 로그를 출력하기 때문에 LOG_DIR에 로그 디렉터리를 지정하고 있다(원하는 디렉터리를 지정한다). 참고로 이후 절차를 계속할 때는 kafka-avro-console-consumer를 열어두고 진행할 때마다 데이터가 표시되는 모습을 확인하면 보다 이해하기 쉬울 것이다.

계속해서 S3에 기록할 커넥터를 실행해보자.

```bash
(kafka-client)$ echo '
{
  "name" : "sink-sales-data-sr",
  "config" : {
    "connector.class" : "io.confluent.connect.s3.S3SinkConnector",
    "s3.bucket.name" : "datahub-sales",
    "s3.region" : "ap-northeast-1",
    "storage.class" : "io.confluent.connect.s3.storage.S3Strage",
    "format.class" : "io.confluent.connect.s3.format.json.JsonFormat",
    "flush.size" : 3,
    "topics" : "possales_sr_pos_uriage",
    "tasks.max" : "3"
  }
}
' | curl -X POST -d @- http://localhost:8083/connectors \
# http://kafka-broker01:8083/connectors
> --header "content-Type:application/json"
```

싱크 쪽까지 동작하게 되었는가? 그럼 S3에 데이터가 들어 있는지 확인한다.

지금까지의 절차는 2.와 동일했다. 이번에는 여기서 Schema Registry 상태도 확인해본다. Kafka Connect가 이미 실행되고 있기 때문에 소스 쪽 커넥터에 의해 Schema Registry에 자동으로 스키마 정보가 기록되어 있을 것이다. 확인해보자.

```bash
(kafka-client)$ curl -X GET http://localhost:8081/subjects
# http://kafka-broker01:8081/subjects

["possales_sr_pos_uriage-value"]
```

내용을 살펴보면 스키마는 진화하기 때문에 버전 관리되고 있다.

```bash
(kafka-client)$ curl -X GET http://localhost:8081/subjects\
# http://kafka-broker01:8081/subjects
> possales_sr_pos_uriage-value/versions
[1]

(kafka-client)$ curl -X GET http://localhost:8081/subjects\
# http://kafka-broker01:8081/subjects
> possales_sr_pos_uriage-value/versions/1 | python -m json.tool

{
  "id": 1,
  "schema": "<schemaJSON>", # shema Json 형식 내용 생략
  "subject": "possales_ur_pos_uriage-value",
  "version": 1
}
```

schema 요소를 알아 보기 어려우므로 읽기 쉬운 포맷으로 바꾸어 살펴보자.

```bash
$echo "<schemaJSON>" | python -m json.tool

{
  "connect.name": "pos_uriage",
  "fields": [
    {
      "name": "seq",
      "type": "long"
    },
    {
      "name": "sales_time",
      "type": {
        "connect.name": "org.apache.kafka.connect.data.Timestamp",
        "connect.version": 1,
        "logicalType": "timestamp-millis",
        "type": "long"
      }
    },
    {
      "default": null,
      "name": "sales_id",
      "type": [
        "null",
        "string"
      ]
    },
    {
      "default": null,
      "name": "shop_id",
      "type": [
        "null",
        "string"
      ]
    },
    {
      "default": null,
      "name": "item_id",
      "type": [
        "null",
        "string"
      ]
    },
    {
      "default": null,
      "name": "amount",
      "type": [
        "null",
        "int"
      ]
    },
    {
      "default": null,
      "name": "unit_price",
      "type": [
        "null",
        "int"
      ]
    },
  ],
  "name": "pos_uriage",
  "type": "record"
}
```

이러한 스키마가 등록되어 있었다.

다음으로 이 스키마를 진화시키는 것에 대해 생각해보자. POS에서 데이터 허브에 보낼 데이터에 연령이 추가되게 되었다고 가정하자. 그러므로 POS의 데이터를 저장하고 있던 테이블에 연령 칼럼을 추가하는 변경을 해보자.

그런데 Shema Registry는 조금 전에 확인했듯이 후방 호환성을 보장하는 설정으로 동작하고 있다. 후방 호환성을 무너뜨리는 변경을 넣으려고 하면 어떻게 될까? 후방 호환성을 무너뜨리는 변경을 여러 가지가 있는데 그 중 하나는 '칼럼을 추가하지만, 그 칼럼의 디폴트 값이 설정되어 있지 않는 경우'다. 예전 스키마에서 투입된 데이터를 새로운 스키마에서 읽으려고 하면 추가된 칼럼의 값을 결정할 수 없기 때문에 후방 호환성을 유지할 수 없다.

Kafka Connect에서는 카프카에 보낼 때의 스키마를 커넥터가 자동적으로 결정하지만, 이 커넥터는 다음과 같이 연령 칼럼 `age`를 추가하면, 카프카에 보내는 데이터의 스키마에 디폴트 값을 설정하지 않게 된다.

```sql
ALTER TABLE pos_uriage ADD COLUMN age INT NOT NULL;

INSERT INTO pos_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price, age)
VALUES (10, '2018-10-21 11:11:11', 'POSSALES00008', 'SHOP001', 'ITEM008', 1, 422, 25);
```

칼럼을 추가한 후에 데이터를 삽입하면 Kafka Connect가 그 데이터를 읽기 때문에 즉시 다음과 같은 오류 메시지가 출력될 것이다.

```bash
[2018-08-09 19:05:07,752] ERROR WokerSourceTask{id=load-possales-data-sr2-0} Task threw an uncaugth and unrecoverable exception
  (중간 생략)
Caused by: org.apache.kafka.common.errors.SerializationException: Error registering Avro shcema: {...}
  (이후 생략)
```

Kafka Connect는 새롭게 삽입된 데이터의 스키마를 Schema Registr에 등록하려고 했지만 Schema Registry의 호환성 체크에서 반려되었다. 새로 등록하려고 한 스키마 정보도 함께 로그에 남아 있는데, 역시 `age`라는 필드에 기본값이 설정되어 있지 않아 후방 호환성이 없는 것을 알 수 있다.

그렇다면 이것을 후방 호환성이 있는 상태로 고쳐보자. 이 커넥터에서는 NOT NULL 제약 조건이 없으면 스키마에 기본값으로 null이 설정된다.

```sql
ALTER TABLE pos_uriage MODIFY age INT;
```

조금 전 오류에서 Kafka Connect의 태스크가 종료됐기 때문에 태스크 또는 작업을 재시작하자.

```bash
(kafka-client)$ curl -X POST http://localhost:8083/connectors\
# http://kafka-broker01:8083/connectors
> load-possales-data-sr/restart
(kafka-client)$ curl -X POST http://localhost:8083/connectors\
# http://kafka-broker01:8083/connectors
> load-possales-data-sr/tasks/0/restart
```

이 시점에서 Kafka Connect는 조금 전 실패한 데이터를 다시 로드하고 있을 것이다. `kafka-avro-console-consumer`를 열고 있는 사람은 출력을 확인해본다. 이번에는 새로운 스키마도 등록되어 있을 것ㅇ시다. Schema Registry를 살펴보자.

```bash
(kafka-client)$ curl -X GET http://localhost:8081/subjects\
# http://kafka-broker01:8081/subjects
> possales_sr_pos_uriage-value/versions
[1,2]
```

버전이 증가했기 때문에 새로운 스키마가 등록되어 있다. 내부를 들여다보자.

```bash
(kafka-client)$ curl -X GET http://kafka-broker01:8081/subjects/\
> possales_sr_pos_uriage-value/versions/2 | python -m json.tool

{
  "id": 2,
  "schema": "<schemaJSON>",
  "subject": "posales_sr_pos_uriage-value",
  "version": 2
}

$ echo "<schemaJSON>" | python -m json.tool
{
  "connect.name": "pos_uriage",
  "fields": [
    {
      "name": "seq",
      "type": "long"
    },

    (중간 생략)

    {
      "default": null,
      "name": "age",
      "type": [
        "null",
        "int"
      ]
    },
  ],
  "name": "pos_uriage",
  "type": "record"
}
```

Schema Registry에 두 번째 버전의 스키마가 등록되어 있으며, 두 번째 버전의 스키마에는 `age`라는 칼럼이 추가되어 있는 것을 알 수 있다.

S3에서 출력은 어떻게 되어 있을까? 출력하기 위해 2줄을 추가하자.

```sql
INSERT INTO pos_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price, age)
VALUES (11, '2018-10-21 11:11:11', 'POSSALES00008', 'SHOP001', 'ITEM009', 1, 120, 25);
INSERT INTO pos_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price, age)
VALUES (12, '2018-10-21 11:11:11', 'POSSALES00008', 'SHOP001', 'ITEM010', 1, 140, 25);
```

S3을 확인해보자. 다음과 같은 파일이 만들어져 있을 것이다.

```json
// possales_sr_uriage+0+0000000009.json

{"seq":10,"sales_time":1540120271000,"sales_id":"POSSALES00008","shop_id":"SHOP001","item_id":"ITEM008","amount":1,"unit_price":422,"age":25}
{"seq":11,"sales_time":1540120271000,"sales_id":"POSSALES00008","shop_id":"SHOP001","item_id":"ITEM009","amount":1,"unit_price":120,"age":25}
{"seq":12,"sales_time":1540120271000,"sales_id":"POSSALES00008","shop_id":"SHOP001","item_id":"ITEM010","amount":1,"unit_price":140,"age":25}
```

이제 소스 쪽의 스키마 변경이 싱크 쪽에까지 전해졌다. 이렇듯 카프카에서 Schema Registry를 사용해 후방 호환성을 유지하면서 스키마 진화를 실현할 수 있다. Kafka Connect에서 Schema Registry의 간단한 사용법을 설명했지만, Schema Registry는 Kafka Connect만을 위한 것이 아니며 프로듀서와 컨수머를 직접 구현한 경우에도 사용할 수 있기에 이것도 직접 시도해보면 좋을 것이다.

그럼 뒷정리를 하고 끝내자. 커넥터를 정지한다.

```bash
(kafka-client)$ curl -X DELETE http://kafka-broker01:8083/connectors/load-possales-data-sr
(kafka-client)$ curl -X DELETE http://kafka-broker01:8083/connectors/sink-sales-data-sr
```

Kafka Connect를 [Ctrl] + [C]로 종료한다.

## 7.8 정리

이 장에서는 Kafka Connect를 사용한 데이터 허브 아키텍처 사례를 살펴봤다. 카프카를 데이터 허브로 여러 시스템과 연계하는 모습을 이해할 수 있었다. 다음은 각각의 데이터 허브를 살펴볼 차례다. 주변 시스템을 고려하면서 Kafka Connect를 사용한 구현 방법을 생각해보자. 만약 좋은 소재가 떠오르지 않는다면 이번에 든 소매점의 예 중 아직 구현하지 않은 `3.`(판매 예측 데이터와 재고 관리 시스템 데이터로 자동 주문하기)을 생각해보는 것도 좋을 것이다.

이 장에서는 등장하지 않은 설정의 예를 시도하거나, 다른 커넥터 플러그인을 시도하다 보면 흥미를 느낄 수 있을 것이다. 번들 커넥터 오이에도 인증된 커넥터나 기타 커뮤니티에서 만든 커넥터도 많다. 혹은 직접 커넥터를 구현할 수도 있다.

어떤 데이터 허브를 만들지는 여러분에게 달려 있따. 여러분이 생각하는 멋진 데이터 허브의 세계를 구축해보길 바란다.
