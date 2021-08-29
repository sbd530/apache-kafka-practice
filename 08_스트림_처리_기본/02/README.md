### 8.4.6 InfluxDB에서 데이터 로드

데이터를 시각화하기 위한 준비로 Kafka Streams에 의해 처리한 데이터를 InfluxDB에 기록한다.

InfluxDB는 시계열 데이터 저장에 특화된 데이터 저장소로, 매트릭스의 정보를 시각화하기 위해 사용하는 Grafana의 데이터 소스로 사용된다. SQL과 비슷한 쿼리를 이용하여 시계열 데이터에 액세스할 수 있는 것이 특징인데 이 책에서는 사용자가 직접 이용하는 상황은 가정하고 있지 않다.

Yum 저장소의 정의를 추가하여 `yum` 명령으로 설치한다. 집필 당시에는 InfluxDB 1.5.2를 설치한 환경을 이용했다.

```bash
$ cat <<EOF | sudo tee /etc/yum.repos.d/>influxdb.repo
[influxdb]
name = InfluxDB Repository - RHEL \$releasever
baseurl = https://repos.influxdata.com/rhel\$releasever/\$baseurl/stable
enabled = 1
gpgcheck = 1
gpgkey = https://repos.influxdata.com/influxdb.key
EOF

$ sudo yum install influxdb
```

여기에서는 디폴트 설정 그대로 서비스를 시작한다.

```bash
$ sudo systemctl start influxdb
```

그리고 InfluxDB의 CLI인 `influx` 명령을 사용하여 데이터베이스를 작성한다.

```bash
$ influx -execute 'CREATE DATABASE kmetrics'
```

카프카 토픽에서 InfluxDB로 데이터 로드는 Fluentd를 이용하여 실행한다.

먼저 `td-agent-gem` 명령을 이용해 필요한 플러그인을 설치한다.

```bash
$ sudo td-agent-gem install fluent-plugin-influxdb
```

Fluentd의 설정 파일 (`/etc/td-agent/td-agent.conf`)에는 카프카의 토픽(`kafka.metrics.processed`)에서 처리 완료된 매트릭스 정보를 읽어 InfluxDB에 기록하기 위한 설정을 추가한다.

```xml
<source>
  @type kafka_group
  brokers localhost:9092
  consumer_group kafka-fluentd-influxdb
  topics kafka.metrics.processed
  format json
  offset_commit_interval 60
</source>

<match kafka.metrics.processed>
  @type influxdb
  host localhost
  port 8086
  dbname kmetrics
  measurement kafka.broker
  tag_keys ["hostname"]
  time_key "timestamp"
  <buffer>
    @type memory
    flush_interval 10s
  </buffer>
</match>
```

설정이 끝나면 `td-agent` 서비스를 재시작하여 반영시킨다.

```bash
$ sudo systemctl restart td-agent
```

InfluxDB에 데이터가 기록되었는지의 여부는 `influx` 명령어를 이용하여 확인할 수 있다. 우선 `kmetrics` 데이터베이스에 접속한다.

```bash
$ influx -databse kemtrics
```

이어서 SELECT문을 사용하여 저장된 데이터를 확인한다.

```bash
> SELECT BytesIn, hostname FROM "kafka.broekr" LIMIT 5
name: kafka.broker
time   BytesIn hostname
----   ------- --------
(생략)
```

시계열 데이터를 보관하는 InfluxDB에서는 관계형 데이터베이스와는 다른 개념으로 정보가 보관되어 있다.

데이터 집합을 나타내는 개념이 measurement이며 이는 데이터베이스의 테이블에 해당한다. 위에서는 measurement 이름으로 `kafka.broker`를 지정하고 있다.

데이터 레코드는 타임스탬프/필드값/태그값으로 구성된다. 필드와 태그는 각각 복수 존재한다. 필드와 태그는 모두 데이터베이스 테이블의 열에 해당하는 것이다. 필드가 시계열의 숫자 데이터를 보관하는 부분인데 반해 태그는 필드 값을 그룹화하기 위한 레이블에 해당한다.

`fluent-plugin-influxdb` 설정 파라미터에서 `tag_key`로 지정한 요소는 태그로 취급하고, `time_key`로 지정된 요소는 타임스탬프로, 그리고 그 이외의 요소는 필드로 취급한다. 따라서 예제 프로그램이 출력한 다음의 JSON의 경우 `hostname`은 태그로, `BytesIn`은 필드로 InfluxDB에 보관된다.

```json
{ "hostname": "host1", "timestamp": 1532795422, "BytesIn": 94072284 }
```

### 8.4.7 Grafana 설정

Grafana를 이용하여 InfluxDB에 보관된 매트릭스 정보를 시각화해보자. `yum` 명령으로 Grafana를 설치한다.

```bash
$ sudo yum install https://s3-us-west-2.amazonaws.com/grafana-releases/release/grafana-5.2.1-1.x86_64.rpm
```

여기에서는 디폴트 설정대로 서비스를 시작한다. Grafana는 사용자가 만든 대시보드의 정의 등을 데이터베이스에 보관한다. 디폴트 설정으로 SQLite를 사용하는 설정이기 때문에 별도로 데이터베이스 설정을 하지 않아도 실행할 수 있지만, 실제 용도로는 PostgreSQL이나 MySQL 등을 이용해야 할 것이다. 데이터베이스의 설정 및 관리자 암호 등은 Grafana 설정 파일 (`/etc/grafana/grafana.ini`)에 기재한다.

```bash
$ sudo systemctl start grafana-server
```

서비스를 시작하면 8080번 포트로 Grafana 웹 인터페이스에 접속할 수 있다. 웹 브라우저에서 `http://<호스트명>:8080/`을 열어 `admin` 사용자로 로그인한다.

우선 처음 설정에서는 데이터 소스로 InfluxDB 데이터베이스를 추가한다. 화면 왼쪽에 있는 [Configuration] 메뉴에서 [Data Source]를 선택한다.

[Add data source] 버튼을 누르면 데이터 소스의 설정화면이 표시되므로 필요한 값을 입력한다. InfluxDB의 Type에 관해서는 드롭다운으로 선택한다.

설정을 입력한후 [Save & Test] 버튼을 클릭하여 보관한다.

데이터 소스 설정이 완료되면 대시보드를 작성한다. 화면 왼쪽에 있는 [+] 기호의 [Create] 버튼을 클릭하고 [Dashboard]를 선택한다. 그러면 새로운 대시보드가 추가된다.

새로운 대시보드 화면에서는 패널 추가용 탭이 표시되어 있으므로 [Graph]를 선택하여 그래프를 추가한다.

그래프 추가후에는 [Panel Title]이라고 표시되어 있는 타이틀 부분을 클릭하고 [Edit]를 선택하여 그래프의 내용을 편집할 수 있다.

그래프 편집 화면에서는 Data Source로 정의한 influxdb를 선택하도록 한다. 쿼리 편집기 화면에서는 BytesIn 필드값을 가져오는 쿼리 내용을 작성한다.

쿼리 편집기 오른쪽의 메뉴 버튼에서 [Toggle Edit Mode]를 선택하면 InfluxDB에 대한 쿼리를 직접 텍스트를 작성할 수도 있따. 여기에서의 쿼리는 다음 내용을 구성되어 있다.

```sql
SELECT "BytesIn" FROM "kafka.broker" WHERE $timeFilter GROUP BY "hostname"
```

쿼리를 입력하면 InfluxDB에서 취득한 데이터를 바탕으로 그래프가 출력될 것이다.

예제 프로그램에서 추출한 것은 카프카 브로커가 컨슈머에 송신한 데이터의 양(바이트)이다. 이것은 카프카 브로커 실행 다음부터의 누적값이므로 플롯하면 단조 증가하는 우상향 그래프가 된다. 시간당 데이터 사용량이 높은 상태는 그래프의 기울기로 표현되는데 서버의 부하 상태를 확인하기 위해서는 사용하기 불편한 그래프다. 따라서 쿼리를 수정하여 누계 데이터 양 자체가 아니라 시간당 데이터 양(bytes/sec)을 플롯해본다.

Fluentd의 설정에 의해 10초 간격으로 데이터를 얻고 있는데 예에서는 1분 간격으로 값의 차이에 따라 시간당 데이터 양을 계산하고 있다.

```SQL
SELECT non_negative_derivative(las("BytesIn"), 1m) / 60
FROM "kafka.broker" WHERE $timeFilter GROUP BY time(1m), "hostname"
```

이로써 서버의 부하 상태 기준이 되는 단위 시간당 데이터 양이 그대로 세로 축에 나타나는 그래프가 된다.

## 8.5 예제 프로그램 살펴보기

이제 앞 절에서 이용한 예제 프로그램(`StreamExample.java`)에 대해 살펴보자.

### 8.5.1 Streams DSL

예제 프로그램 주요 부분에서는 KStream 클래스의 객체를 작성하고 있다.

```java
StreamsBuilder builder = new StreamsBuilder();
KStream<String, String> metrics =
    builder.stream("kafka.metrics",Consumed.with(Serdes.String(), Serdes.String()));
```

KStream은 Kafka Streams가 제공하는 클래스로 Streams DSL이라는 추상도가 높은 API의 일부다. 위 예의 경우 `kafka.metrics`라는 토픽에서 취득한 레코드의 집합을 추상화한 것이다.

Kafka Streams의 KStreams 클래스와 유사한 기능을 제공하는 예로 자바의 Streams API가 있다. 배열이다 목록으로 대표되는 컬렉션 성격의 데이터에 대해 `filter` 메서드로 특정 조건에 맞는 요소를 추출하거나, `map` 메서드로 변환 처리(함수)를 각 요소에 적용하거나 하는 인터페이스를 제공한다. Streams API를 이용함으로써 데이터로의 복잡한 변환을 반복하는 코드를 보기 쉽게 잘 작성할 수 있다. 다음 예제 코드는 자바 문서에서 인용한 것이지만 데이터 추출 조건관 각 요소에 대한 조작할 자바 8부터 사용할 수 있는 람다식으로 작성해 적을 양의 코드로 보다 읽기 쉽게 되었다.

```java
int sum = widgets.stream()
                 .filter(b -> b.getColor() == RED)
                 .mapToInt(b -> b.getWeight())
                 .sum();
```

Kafka Streams의 KStream 클래스도 비슷한 기능을 제공하고 있다. 처리 대상이 되는 KStream의 각 요소는 간헐적으로 카프카의 토픽에 계속 저장되는 레코드이며 처리도 간헐적으로 이루어진다.

다음은 예제 프로그램에서 KStream에 대한 조작을 정의하는 부분이다. KStream에 대해 `flatMapValues` 메서드에 의한 값의 변환과 `filter` 메서드에 의한 요소의 추출을 적용했다. 처리한 데이터는 `to` 메서드에 의해 카프카의 `kafka.metrics.processed`라는 토픽에 써서 내보내도록 지정하고 있다.

```java
metrics.flatMapValues(wrap(text -> mapper.readTree(text)))
        .filter((host, root) -> root.has("value")
                        && root.has("hostname")
                        && root.has("timestamp"))
        .flatMapValues(wrap(root -> {
            ObjectNode newroot = mapper.createObjectNode();
            newroot.put("hostname", root.get("hostname"));
            newroot.put("timestamp", root.get("timestamp"));
            newroot.put("BytesIn", root.get("value")
                        .get("kafka.server:name=BytesInPerSec,type=BrokerTopicMetrics")
                        .get("Count"));
            return mapper.writeValueAsString(newroot);
        }))
        .to("kafka.metrics.processed",
                Produced.with(Serdes.String(), Serdes.String()));
```

토픽에서 추출한 레코드 값은 JSON 문자열이다. 여기에서는 JSON을 처리하기 위한 라이브러리인 Jackson을 이용하고 있으며, 첫 번째 `flatMapValues` 메서드에서 적용하는 조작은 JSON 문자열을 분석해 Jackson 객체로 변환하는 것이다.

다음으로 `filter` 메서드를 이용해 처리에 필요한 필드를 갖지 않는, 즉 처리할 수 없는 레코드를 제외시키고 있다.

두 번째의 `flatMapValues` 메서드에서는 입력되는 JSON 객체에서 일부 데이터를 추출해 새로운 JSON을 만들고 그 문자열 표현을 값으로 하는 변환을 적용했다.

여기까지의 정의를 바탕으로 Kafka Streams 객체를 생성하고 `start` 메서드를 호출함으로써 Kafka Streams 처리의 메인 루프를 실행하는 스레드를 실행한다.

```java
KafkaStreams streams = new KafkaStreams(builder.build(), config);
streams.start();
```

### 8.5.2 스트림 처리 오류 다루기

스트림 처리에서는 간헐적으로 입력되는 데이터를 계속해서 처리하기 때문에 혹여 처리할 수 없는 데이터 레코드가 있을 경우 프로그램을 오류로 종료하면 문제 발생할 수 있다. 오류 내용을 기록하는 등의 조치를 한 후에 그 레코드는 건너뛰고 다음 데이터 레코드를 처리하는 것이 기본적인 오류 핸들링의 방침이다.

데이터 스트림 각 요소에 대한 처리를 람다식으로 작성한 경우 다음과 같이 간결하게 작성할 수 있는것이 이점이다.

```java
stream.mapValues(v -> v.toLowerCase())
```

하지만 예외 처리를 해야할 경우에는 전체적인 모양이 나빠지기 쉽다.

```java
stream.mapValues(v -> {
    try {
        // throws IndexOutOfBoundsException
        return v.substring(3);
    } catch (Exception e) {
        return "";
    }
})
```

이 경우 예외 처리를 포함하는 코드 블록을 메서드로 정의하여 그 메서드를 호출하는 것이 하나의 방법이다.

```java
stream.map((k,v) -> mysub(k, v))
  (중간 생략)
private static String mysub(String v) {
    try {
        // throws IndexOutOfBoundsException
        return v.substring(3);
    } catch (Exception e) {
        return "";
    }
}
```

예제 프로그램에서는 동일한 패턴에서의 예외 처리가 여러 곳에 있는 경우 그것을 대응하는 하나의 예로 람다식 함수를 래핑해, 예외가 발생했을 경우에 빈 목록을 반환하는 함수로 변환하는 메서드를 정의하고 있다. `wrap` 메서드로 래핑한 함수를 호출하는 경우에는 `mapValues` 대신 `flatMapValues` 메서드를 이용한다.

```java
stream.flatMapValues(wrap(v -> v.substring(3)))
  (중간 생략)
@FunctionalInterface
private interface FunctionWithException<T, R, E extends Exception> {
    R apply(T t) throws E;
}

private static <V,VR,E extends Exception>
    ValueMapper<V, Iterable<VR>> wrap(FunctionWithException<V,VR,E> f) {

    return new ValueMapper<V, Iterable<VR>>() {
        @Override
        public Iterable<VR> apply(V v) {
            try {
                return Arrays.asList(f.apply(v));
            } catch (Exception e) {
                e.printStackTrace();
                return Arrays.asList();
            }
        }
    };
}
```

## 8.9 Kafka Streams의 장점

당연한 이야기이지만 Kafka Streams를 이용하여 애플링케이션을 구현하는 것이 기본적인 카프카 API를 이용하는 경우보다 가독성이 좋으며 유지보수 또한 쉬운 코드 작성이 가능하다. 게다가 Kafka Streams는 카프카에 부속된 라이브러리이기 때문에 도입하기 쉽다.

카프카의 기능을 능숙하게 사용할 수 있다는 관점에서 보자면 카프카의 클라이언트 라이브러리 기능을 활용하여 여러 프로세스에 의한 병렬 처리를 쉽게 구현할 수 있다는 점이 핵심일 것이다. 병렬 처리의 어려운 부분인 처리 배분과 배타 제어에 대해서는 Consumer Group 안에서의 컨수머에 대한 Topic Partition 할당을 통해 투명하게 실현된다. 오류를 다루는 관점에서는 스트림 처리 과정에서 오류가 발생하면 중간부터 처리를 재개하는 것에 대해서도 카프카의 오프셋 관리 매커니즘에 의해 간단히 실현할 수 있다.

참고로 Kafka Streams로 처리한 데이터를 반드시 카프카 토픽에 기록할 필요는 없다. 단, 데이터의 입력 처리량이 큰 경우 필연적으로 스트림의 출력 처리량도 커지기 쉬우므로 데이터를 출력하는 곳에서도 고성능이며 확장 가능한 카프카가 유력한 후보가 될것이다. 그러한 점에서 카프카 코어부분과 동일한 프로젝트로 개발된 Kafka Streams는 안심하고 사용할 수 있다.
