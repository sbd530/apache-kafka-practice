## 7.5 전자상거래 사이트에 실제 매장의 재고 정보를 표시하기

우선 1.의 재고관리 시스템과 전자상거래 사이트의 파일 연계부터 시작한다.

참고로 이 절 이후에는 실제로 Kafka Connect를 동작시켜보자. 그리고 투입된 데이터를 확인할 때는 여러 명령을 동시에 실행할 수도 있다. 따라서 필요에 따라 여러 터미널을 실행해 각 터미널에서 명령해보기를 바란다.

### 7.5.1 카프카와 Kafka Connect 준비

우선 카프카 클러스터를 준비한다.(이미 구축이 되어 있다면 그것을 사용해도 된다). 준비한 클러스터를 데이터 허브로 사용한다.

다음은 Kafka Connect다. 컨플루언트 플랫폼으로 설치하면 Kafka Connect도 동시에 이용할 수 있으므로 추가로 설치하거나 설정하지 않아도 된다.

커넥터 플러그인도 준비해야 한다. 단, 이번에 사용하는 플러그인은 모두 컨플루언트 플랫폼에 포함된 것을 사용하므로 이것도 추가로 설치할 필요는 없다. 만약 번들 외의 플러그인을 사용하려면 설치해야 한다. 플러그인이 구현된 jar를 적당한 디렉터리에 배치하면 준비는 끝난다.

JDBC에 접속할 때는 추가적으로 각각의 RDBMS 드라이버가 필요하다. 컨플루언트 플랫폼에는 PostgreSQL 드라이버는 있지만 MariaDB 드라이버가 없다. MariaDB의 RDBMS 드라이버에 대한 설치 절차는 2.를 준비하는 과정에서 설명하겠다.

### 7.5.2 데이터 준비

파일 연계를 위한 데이터를 준비하자. Kafka Connect의 FileStream Connectors에 있어서 파일이란 Kafka Connect에서 봤을 때의 로컬 파일을 뜻한다. 그러므로 Kafka Connect가 동작하는 서버상에서 읽을 수 있는 로컬 파일을 재고 관리 시스템과 전자상거래 사이트에 준비한다. 약간 이상한 구성이지만 앞서 언급한 바와 같이 기본을 확실히 익히는 것이 이해하기 쉽다. NAS에 마운트되어 있다고 상상하면 조금은 이해하기 쉽지 않을까 싶다. FileStream Connectors를 상업적으로 사용하는 것을 추천하지 않는 이유도도 여기에 있다. 원래부터 Kafka Connect는 여러 서버에서 실행되는 것이다. 그러나 이 FileStream Connectors라는 커넥터는 로컬 파일을 다루기 떄문에 확장시키고 어렵고 조화도 잘 이루지 못한다. 만일 이것이 NFS 마운트라고 해도 여러 서버에서 동일한 파일을 읽고 쓸 수 있게 되므로 생각대로 동작하지 않을 것임을 쉽게 상상할 수 있다. 그런 이유로 FileStream Connector는 분산 처리에 적합하지 않은, 즉 상업용으로 사용해서는 안된다는 뜻이다. 참고로 이 책에선느 FileStream Connector를 사용하는 경우에 Kafka Connect를 Kafka-broker01이라는 서버 하나에서만 실행하고 있다.

**_재고 관리 시스템용 파일 작성_**

재고 관리 시스템은 사용한는 커넥터 플러그인의 특성상 재고가 변동하면 특정 파일에 최신의 재고 정보를 추가하는 방식으로 한다. 이 파일은 Kafka-broker01의 로컬 파일시스템에 둔다. 따라서 재고 관리 시스템은 kafka-broker01에 파일을 만들면 완성이다.

/zaiko 디렉터리를 만들어보자.

```bash
(kafka-broker01)$ sudo mkdir /zaiko
(kafka-broker01)$ sudo chown $(whoami) /zaiko
```

파일을 만들고 재고 데이터를 적당히 투입하자. 파일 형식은 CSV를 가정하고 있는데 원하는 형태로 데이터를 작성하면 된다. 상품ID/재고를 갖고 있는 점포ID/재고수/재고 변동 시간의 정보를 가진 형식을 가정하면 다음과 같은 데이터가 된다. 이러한 데이터를 만들고 파일에 쓴다.

```bash
(kafka-broker01)$ cat << EOF > /zaiko/latest.txt
ITEM001,SHOP001,929,2018-10-01 01:01:01
ITEM002,SHOP001,480,2018-10-01 01:01:01
ITEM001,SHOP001,25,2018-10-02 02:02:02
ITEM003,SHOP001,6902,2018-10-02 02:02:02
EOF
```

재고 관리 시스템이 만들어 졌다.

**_전자상거래 사이트용 파일 작성_**

전사상거래 사이트용 파일도 동일하다. 이것은 데이터를 받을 뿐이므로 kafka-broker01에 /ec라는 디렉터리만 만들자.

```bash
(kafka-broker01)$ sudo mkkir /ec
(kafka-broker01)$ sudo chown $(whoami) /ec
```

전자상거래 사이트 준비가 완료됐다.

### 7.5.3 Kafka Connect 실행

이제 Kafka Connect와 연계 시스템의 준비가 모두 끝났다. 드디어 Kafka Connect를 구동하여 재고관리 시스템과 전자상거래 사이트를 연결해보자.

재고 관리 시스템의 파일을 카프카에 투입하는 부분에서는 Kafka Connect에서 보면 파일이 소스가 된다. 그래서 FileStream Connetors 중 FileSource Connector를 사용한다.

한편 카프카의 데이터를 전자상거래 사이트의 파일에 투입하는 부분에서는 Kafka Connect에서 보면 파일은 싱크가 된다. 그래서 여기에는 FileStream Connectors 중에서 FileSink Connector를 사용한다.

어쨌든 처음에 필요한 것은 Kafka Connect를 시작하는 것이다. Kafka Connect는 카프카가 동작하고 있는 클러스터상에는 움직인다. 먼저 카프카 클러스터가 동작하고 있는지 확인하자.

Kafka Connect의 동작에는 Standalone 모드와 Distributed 모드가 있다. Standalone 모드는 Kafka Connect가 1개만 움직이는 모드이면, 개발 환경에서 사용할 때나 1개의 서버만을 연계할 때 사용한다. 대부분 상업용 환경에서는 여러 개의 서버에서 동작하는 Distributed 모드를 사용한다. 여기서는 Distributed 모드이지만 앞서 설명한 대로 파일 연계를 하기 때문에 kafka-broker01의 단일 서버에서만 Kafka Connect를 실행한다.

설정 파일을 준비해보자. 컨플루언트 플랫폼에서는 디폴트 설정 파일이 함께 만들어지므로 그것을 복사하여 사용하기로 한다.

```bash
(kafka-broker01)$ cp /etc/kafka/connect-distributed.properties \
> connect-distributed-1.properties
(kafka-broker01)$ vim connect-distributed-1.properties
  (중간 생략)
# bootstrap.servers=kafka-broker01:9092
bootstrap.servers=localhost:9092
group.id=connect-cluster-datahub-1
  (이후 생략)
```

Kafka Connect는 카프카의 브로커와 동일한 서버에서 실행한다. 일반적으로 브로커는 여러개로 구성되는데 처음에 어떤 서버와 접속할지를 bootstrap.servers라는 파라미터에 설정한다. 상업용 환경에서 이용할 때는 3개 이상 설정할 것을 권장한다. Kafka Connect는 여러 서버로 하나의 클러스터를 구성하는데, 동일한 클러스트 내의 서버는 동일한 group.id가 설정되어 있어야 한다.

이 설정 파일을 사용하여 Kafka Connect를 Distributed 모드로 실행한다. 다음과 같은 명령을 사용하여 설정 파일을 인수로 전달한다.

```bash
(kafka-broker01)$ connect-distributed ./connect-distributed-1.properties
```

실행되면 동작 상황을 확인한다. Kafka Connect는 실행하면 REST API로 액세스할 수 있다. API 중에서 현재 실행 중인 버전을 반환하는 API가 있으므로 실행해서 확인해보자.

```bash
# (kafka-broker01)$ curl http://kafka-broker01:8083/
(kafka-client)$ curl http://localhost:8083/
{"version":"2.0.0-cp1","commit":"fc9a81e8d72f61be",
"kafka_cluster_id":"r3kI5RU0TWukdSpcJ5q0Tg"}
```

실행 중인 Kafka Connect에서 사용할 수 있는 커넥터 플러그인을 조사하는 API도 있으므로 테스트해보자. 출력되는 양이 많으면 1행으로 표시되는 JSON은 읽기 어렵기 때문에 jq 명령이나 파이썬의 json.tool 등으로 읽기 쉽게 표시하면 좋을 것이다.

```bash
# (kafka-client)$ curl http://kafka-broker01:8083/connector-plugins | python -m json.tool
(kafka-client)$ curl http://localhost:8083/connector-plugins | python -m json.tool
[
   {
      "class": "io.confluent.connect.elasticsearch.ElasticsearchSinkConnector"
      "type": "sink"
      "version": "5.0.0"
   },
   {
      "class": "io.confluent.connect.hdfs.HdfsSinkConnector"
      "type": "sink"
      "version": "5.0.0"
   },
   {
      "class": "io.confluent.connect.hdfs.tools.SchemaSourceConnector"
      "type": "source"
      "version": "2.0.0-cp1"
   },
   {
      "class": "io.confluent.connect.jdbc.JdbcSinkConnector"
      "type": "sink"
      "version": "5.0.0"
   },
   {
      "class": "io.confluent.connect.jdbc.JdbcSourceConnector"
      "type": "source"
      "version": "5.0.0"
   },
   {
      "class": "io.confluent.connect.s3.S3SinkConnector"
      "type": "sink"
      "version": "5.0.0"
   },
   {
      "class": "io.confluent.connect.storage.tools.SchemaSourceConnector"
      "type": "source"
      "version": "2.0.0-cp1"
   },
   {
      "class": "org.apache.kafka.connect.file.FileStreamSinkConnector"
      "type": "sink"
      "version": "2.0.0-cp1"
   },
   {
      "class": "org.apache.kafka.connect.file.FileStreamSourceConnector"
      "type": "source"
      "version": "2.0.0-cp1"
   }
]
```

컨플루언트 플랫폼으로 설치한 카프카는 디폴트로 다수의 커넥터를 포함하고 있기 때문에 다수의 커넥터 플러그인을 확인할 수 있다. 사용하려고 했단 FileStreamSourceConnector와 FileStreamSingConnector도 찾을 수 있다.

그러면 FileSource Connector를 실행하자. 커넥터 플러그인을 실행하는 것도 REST API를 통해 실시한다. 커넥터를 실행하려면 커넥터 설정을 함께 전달해야 한다. 설정은 JSON 형식으로 작성하여 REST API에 투입한다.

```bash
(kafka-client)$ echo '
{
  "name" : "load-zaiko-data",
  "config" : {
    "connector.class" : "org.apache.kafka.connect.file.FileStreamSourceConnector",
    "file" : "/zaiko/latest.txt",
    "topic" : "zaiko-data"
  }
}
' | curl -X POST -d @- http://localhost:8083/connectors \
# http://kafka-broker01:8083/connectors
> --header "content-Type:application/json"

{"name":"load-zaiko-data","config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSourceConnector","file":"/zaiko/latest.txt","topic":"zaiko-data","name":"load-zaiko-data"},"tasks":[],"type":null}
```

설정 내용은 다음과 같다.

- name<br>
  \- 실행하는 커넥터의 이름을 지정한다. 알기 쉬운 이름을 사용하면 좋다.
- connector.class<br>
  \- 사용할 커넥터의 클래스명을 지정한다.
- file<br>
  \- 읽을 파일명을 지정한다.
- topic<br>
  \- 커넥터가 카프카에 투입할 때 사용할 토픽명을 지정한다.

REST API로 커넥터의 설정을 투입하면 즉시 커넥터가 실행된다. 실행 중인 커넥터 목록을 표시하는 REST API도 있으므로 확인해보자.

```bash
# (kafka-client)$ curl http://kafka-broker01:8083/connectors
(kafka-client)$ curl http://localhost:8083/connectors
["load-zaiko-data"]
```

방금 전에 투입한 커넥터가 실행되고 있음을 알 수 있다. 즉, 재고 데이터는 이미 카프카에 들어와 있을 것이다. 카프카의 내부에 들어가 데이터를 확인하고 싶지 않은가? 이럴 때는 kafka-console-consumer를 사용하면 편리하다.

```bash
(consumer-client)$ kafka-console-consumer \
> --bootstrap-server=localhost:9092 \
# --bootstrap-server=kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092
> --topic zaiko-data --from-beginning
{"schema":{"type":"string","optional":false},"payload":"ITEM001,SHOP001,929,2018-10-01 01:01:01"}
{"schema":{"type":"string","optional":false},"payload":"ITEM002,SHOP001,480,2018-10-01 01:01:01"}
{"schema":{"type":"string","optional":false},"payload":"ITEM001,SHOP001,25,2018-10-01 01:01:01"}
{"schema":{"type":"string","optional":false},"payload":"ITEM003,SHOP001,6902,2018-10-01 01:01:01"}
```

zaiko-data의 토픽 내용이 표시되었다. 재고 데이터로 생성한 파일의 내용이 행마다 JSON으로 변환되어 zaiko-data에 로드되어 있는 것을 확인할 수 있다. 소스 쪽 커넥터는 제대로 동작하는 것 같다.

이어서 싱크 쪽의 커넥터를 설정해보자. 이번에는 이 zaiko-data에 들어온 데이터를 전자상거래 사이트로 보낸다. 이번에는 카프카를 데이터 소스로 사용하여 파일에 출력하는 커넥터를 앞과 마찬가지로 실행한다. 이미 Kafka Connect는 작동하고 있기 때문에 커넥터를 실행한다. 이번에도 FileStream Sink Connector를 실행하기 위한 설정을 JSON 형식으로 REST API에 투입한다.

```bash
(kafka-client)$ echo '
{
  "name" : "sink-zaiko-data",
  "config" : {
    "connector.class" : "org.apache.kafka.connect.file.FileStreamSinkConnector",
    "file" : "/ec/zaiko-latest.txt",
    "topics" : "zaiko-data"
  }
}
' | curl -X POST -d @- http://localhost:8083/connectors \
# http://kafka-broker01:8083/connectors
> --header "content-Type:application/json"

{"name":"sink-zaiko-data","config":{"connector.class":"org.apache.kafka.connect.file.FileStreamSinkConnector","file":"/ec/zaiko-latest.txt","topics":"zaiko-data","name":"sink-zaiko-data"},"tasks":[],"type":null}
```

설정값의 내용은 소스 커넥터의 경우와 동일하다. topics라는 설정값은 여러 토픽을 지정할 수도 있으나 이번에는 하나면 충분하다.

제대로 투입되었다면 다시 한 번 실행 중인 커넥터 목록을 살펴보자

```bash
# (kafka-client)$ curl http://kafka-broker01:8083/connectors
(kafka-client)$ curl http://localhost:8083/connectors
["sink-zaiko-data","load-zaiko-data"]
```

소스 쪽 커넥터 외에 싱크 쪽 커넥터도 동작하고 있음을 알 수 있다. 즉, 결과가 파일에 출력되고 있을 것이다. 출력된 결과를 살펴보자.

```bash
(kafka-broker01)$ cat /ec/zaiko-latest.txt
ITEM001,SHOP001,929,2018-10-01 01:01:01
ITEM002,SHOP001,480,2018-10-01 01:01:01
ITEM001,SHOP001,25,2018-10-02 02:02:02
ITEM003,SHOP001,6902,2018-10-02 02:02:02
```

결과가 전자상거래 사이트의 파일에 출력됐다. 이제 Kafka Connect를 통해 재고 관리 시스템 파일에서 전자상거래 사이트의 파일까지 연결됐다.

소스 쪽 재고 관리 시스템의 데이터에 변경이 있다면 어떻게 될까? 그것도 확인해보자.

먼저 싱크 쪽의 전자상거래 사이트 데이터를 모니터링해보자.

```bash
(kafka-broker01)$ tail -f /ec/zaiko-latest.txt
```

재고 관리 시스템의 데이터에 몇 행 정도 추가한다.

```bash
(kafka-broker01)$ cat << EOF > /zaiko/latest.txt
ITEM001,SHOP001,6090,2018-10-03 03:00:00
ITEM004,SHOP001,256,2018-10-03 03:00:00
EOF
```

tail 명령을 사용해서 모니터링을 하고 있는 곳에서도 추가분이 반영되었음을 알 수 있다.

```bash
(kafka-broker01)$ tail -f /ec/zaiko-latest.txt
  (중간 생략)
ITEM001,SHOP001,6090,2018-10-03 03:00:00
ITEM004,SHOP001,256,2018-10-03 03:00:00
```

이처럼 Kafka Connect를 실행하고 있는 동안은 소스 쪽의 데이터에 변경이 있으면 그것이 항상 싱크 쪽까지 전달된다. 이로써 Kafka Connect를 사용하면 상시 변경이 반영되는 데이터 허버를 쉽게 구현할 수 있다는 것을 알게 됐다.

일련의 Kafka Connect 동작을 확인했으므로 이제 뒷정리를 하고 끝내도록 하자. 우선 커넥터를 제거한다. 삭제도 REST API로 실시한다.

```bash
(kafka-client)$ curl -X DELETE http://localhost:8083/connetors/load-zaiko-data
# http://kafka-broker01:8083/connetors/load-zaiko-data
(kafka-client)$ curl -X DELETE http://localhost:8083/connetors/sink-zaiko-data
```

커넥터를 삭제했다면 목록을 확인해둔다. 확인 명령은 전과 동일하다.

```bash
# (kafka-client)$ curl http://kafka-broker01:8083/connectors
(kafka-client)$ curl http://localhost:8083/connectors
[]
```

실행 중인 커넥터가 없어졌다. 또한 Kafka Connect도 정지해두자. 실행한 connect-distributed 터미널을 [Ctrl] + [C]로 종료한다.

이상으로 재고 관리 시스템과 전자상거래 사이트를 파일 연계로 연결하는 실습을 마치겠다. 파일 연계는 재미없을지도 모르지만 Kafka Connect의 기본을 이해하는 데 좋은 예제다. 실제로 카프카를 직접 동작시켜보면 Kafka Connect의 개요를 더 빨리 이해할 수 있다. 이제 보다 실무에 가까운 응용 예를 살펴보자.

[_2 로 이어짐]
