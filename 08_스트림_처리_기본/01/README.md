# 8장 스트림 처리 기본

## 8.1 이 장의 내용

여기서는 스트림을 처리한느 기본 내용을 다룬다. 여기서 말하는 스트림 처리는 간헐적으로 유입되는 데이터를 수시로 처리하는 처리 모델이다. 이 스트림 처리는 모인 단위 데이터를 한꺼번에 다루는 모델을 가리키는 배치 처리와는 대조되는 형태다.

배치 처리는 일반적으로 **작업**<sup>job</sup>이라는 단위로 실행된다. 한 묶음의 데이터를 입력으로 주고, 그것을 처리한 후에 작업이 완료된다. 이에 반해 스트림 처리에는 분명한 시작과 긑이 없다. 카프카 컨수머의 기본적인 API 사용법을 보면 계속해서 입력되는 데이터를 토픽에서 꺼내 처리하는 것을 무한 반복하는 루프가 있는데 보다시피 원래부터 스트림 처리 모델로 되어 있는 것을 알 수 있다.

```java
while (running) {
    ConsumerRecords<String, String> records = consumer.poll(1000);
    for (ConsumerRecord<String, String> record : records)
      System.out.println(record.value());
}
```

## 8.2 Kafka Streams

Kafka Streams는 카프카가 빌트인으로 제공하는 스트림 처리를 위한 API다. 최근에는 스트림 처리에 특화한 (분산 처리) 프레임워크도 등장하고 있으며 대표적인 것으로 Apache Storm, Apache Flink, Spark Streaming이 있다. 또한 이러한 스트림 처리 기반에서 공통적으로 볼 수 있는 처리 모델을 추상화한 Apache Beam 같은 SDK도 존재한다.

위의 각종 분산 처리 프레임워크에서는 카프카를 데이터 소스로 사용하는 방법이 정평이 나 있는데, 카프카 자체에 포함된 Kafka Streams를 이용함으로써 간편하게 스트림 처리를 할 수 있다.

이 장에서는 Kafka Streams를 이해하기 위한 예로 소프트웨어의 통계 정보인 매트릭스<sup>metrics</sup>를 스트림으로 처리해본다.

## 8.3 컴퓨터 시스템의 매트릭스

컴퓨터 시스템에 있어 CPU 사용량과 메모리 사용량 등을 몇 초에서 몇 분 간격으로 정기적으로 취합하여 이를 관리용 서버에 통합해 값을 확인하는 모니터링이 널리 이용되고 있다.

CPU 사용량과 메모리 사용량은 순간순간의 하드웨어와 스프트웨어 상태를 나타내는 통곗값으로 이를 매트릭스라고 부른다. 매트릭스는 컴퓨터 시스템이 실행되는 동안 간헐적으로 발생하는 데이터로 파악할 수 있다.

네트워크나 디스크 I/O 같은 OS에서 얻을 수 있는 매트릭스 이외에 요청 처리 수와 같은 각종 미들웨어가 제공하는 다양한 매트릭스가 존재한다. 매트릭스 단독으로는 단순한 숫자일 수 있지만 운용하고 있는 소프트웨어의 종류, 서버 노드의 수에 따라 전체 시스템에서 발생하는 매트릭스는 나름 낳은 양의 데이터가 된다. 즉, 매트릭스 데이터 집계 및 처리는 카프카가 자랑한느 사례의 하나로 생각할 수 있다.

## 8.4 카프카 브로커의 매트릭스를 시각화 하기

카프카 자체도 브로커와 프로듀서, 컨수머의 상태를 모니터링하기 위한 매트릭스를 출력한다. 이후에서는 예제로 카프카 브로커의 매트릭스를 카프카로 내보내고, Kafka Streams를 이용하여 스트림 처리를 해본다.

### 8.4.1 매트릭스 처리의 흐름

환경을 설정하면서 다음과 같은 요령으로 카프카 브로커의 매트릭스를 처리해보자. 여기에서는 단순화를 도모하기 위해 단일 노드상에 필요한 서비스를 모두 설치하는 구성으로 되어 있다.

1. Fluentd로 카프카 브로커의 매트릭스를 정기정으로 취득하고 카프카의 토픽에 기록한다.
2. Kafka Streams로 매트릭스 데이터를 가공한다.
3. 처리된 매트릭스를 Fluentd로 꺼내 InfluxDB에 저장한다.
4. InfluxDB에 저장된 매트릭스를 Grafana로 시각화한다.

참고로 다음에 소개하는 설정 내용은 동일한 노드에 카프카, Fluentd, InfluxDB, Grafana 모두를 설치한 환경을 전제로 한 것이며 연결 시스템 노드가 모두 localhost로 되어 있다.

실제 사용에 있어서는 Kafka Streams에 의한 데이터 처리와 InfluxDB와 Grafana에 의한 데이터 시각화를 카프카 브로커 노드와는 다른 별도의 노드에서 실행해야 한다. Fluentd에 대해서는 위의 `1.`과 `3.` 두 개의 처리를 담당하고 있따. `1.`의 카프카 브로커 매트릭스 취득은 카프카 브로커 노드의 Fluentd에서 실시하고, `3.`의 처리 완료 데이터를 InfluxDB에 저자하는 것은 InfluxDB의 Fluentd에서 각각 실시하는 방식으로 사용을 구분할 수 있다.

### 8.4.2 카프카 설정

다음 설명은 하나의 노드에서 예제 프로그램을 가동시키기 위한 필요 최소한의 설치 절차다. 주키퍼도 컨플루언트 플랫폼에 포함되어 있는 것을 기본 설정으로 시작한다고 가정한다. 카프카 클러스터 설치 과정은 3장을 참조한다.

```bash
$ sudo rpm --import https://packages.confluent.io/rpm/5.0/archive.key

$ cat > /tmp/confluent.repo << EOF
[Confluent.dist]
name=Confluent repository (dist)
baseurl=https://packages.confluent.io/rpm/5.0/7
gpgcheck=1
gpgkey=https://packages.confluent.io/rpm/5.0/archive.key
enabled=1

[Confluent]
name=Confluent repository
baseurl=https://packages.confluent.io/rpm/5.0
gpgcheck=1
gpgkey=https://packages.confluent.io/rpm/5.0/archive.key
enabled=1
EOF

$ sudo mv /tmp/confluent.repo /etc/yum.repo.d/
$ sudo yum install confluent-platform-oss-2.11
$ sudo systemctl start confluent-zookeeper
$ sudo systemctl start confluent-kafka
```

### 8.4.3 Jolokia 설정

카프카는 JMX를 이용하여 매트릭스를 제공하고 있다. 자바 이외의 언어로 구현된 도구에서 JMX 정보를 얻기 위해 여기에서는 Jolokia를 사용한다. Jolokia은 자바 에이전트 라이브러리로 자바 프로그램에서 로드되어 HTTP를 통해 JMX 정보를 얻을 수 있는 기능을 제공한다.

먼저 필요한 라이브러리를 Jolokia 웹사이트에서 다운로드하여 브로커 노드에 배치한다.

```bash
$ curl -L -O https://github.com/rhuss/jolokia/releases/download/v1.5.0/jolokia-1.5.0-bin.tar.gz
$ tar zxf jolokia-1.5.0-bin.tar.gz
$ sudo mv jolokia-1.5.0 /opt/
```

다음으로 카프카에서 서비스 설정을 수정하여 JVM 옵션을 추가하고 Jolokia 라이브러리가 카프카에서 로드되도록 한다.

CentOS7에 컨플루언트 플랫폼의 RPM 패키지를 설치한 환경인 경우 카프카 브로커 서비스는 `system` 경유로 실행되기 때문에 `confluent-kafka` 서비스의 설정 파일(`/lib/systemd/system/confluent-kafka.service`)을 편집해 [Service] 섹션 내 KAFKA_OPTS라는 환경 변수의 정의를 추가한다. `KAFKA_OPTS`의 값은 카프카의 실행 스크립트에 의해 프로세스 실행시 추가 JVM 옵션으로 취급된다.

```bash
$ cp /lib/systemd/system/confluent-kafka.service /tmp/
$ sudo vi /lib/systemd/system/confluent-kafka.service
$ diff /tmp/confluent-kafka.service /lib/systemd/system/confluent-kafka.service
12a12
 >
Environment=KAFKA_OPTS=javaagent:/opt/jolokia-1.5.0/agents/jolokia-jvm.jar
```

설정 변경 후 카프카 브로커를 정상적으로 다시 실행할 수 있다면 카프카 브로커 프로세스는 8778번 포트로 접속을 기다리고 있을 것이다.

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl restart confluent-kafka
$ sudo ss -anlp | grep 8778
tcp    LISTEN   0        10     ::fff:127.0.0.1:8778     :::* users:(("java",pid=2150,fd=100))
```

확인을 위해 curl 명령으로 JMX의 매트릭스를 취득한다. 취득한 데이터의 형식은 JSON으로 되어 있기 때문에 내용을 추출하거나 보기 쉽게 하기 위해 파이썬 명령어나 `jq` 명령어와 같은 도구를 이용하면 편리하다.

```bash
$ curl -s 'http://localhost:8778/jolokia/read/kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions' | python -m json.tool
{
    "request": {
        "mbean": "kafka.server:name=UnderReplicatedPartitions,type=ReplicaManager",
        "type": "read"
    },
    "status": 200,
    "timestamp": 1533793373,
    "value": {
        "Value": 0
    }
}
```

JMX에서 얻을 수 있는 매트릭스 JMX MBean이라 불리는 객체 단위로 관리되고 있으며, 위의 예와 같이 `type`과 `name` 같은 속성을 지정함으로써 취득 대상을 좁힐 수 있다.

또한 다음과 같이 와일드 카드를 지정하여 여러 JMX MBean에 대한 정보를 함께 얻을 수 있다.

```bash
http://localhost:8778/jolokia/read/kafka.server:*
```

카프카의 매트릭스에서는 JMX MBean의 객체명에서 도메인 부분에 해당하는 클래스의 패키지명이 사용되고 있으므로 다음과 같은 와일드 카드 지정으로 카프카 브로커의 모든 매트릭스의 내용을 확인할 수 있다. 출력 내용과 그 크기에서 카프카가 매우 다양한 매트릭스를 제공하고 있음을 알 수 있다.

```bash
$ curl -s 'http://localhost:8778/jolokia/read/kafka.*:*' | python -m json.tool
```

### 8.4.4 Fluentd 설치

다음으로 Fluentd를 설치한다. Fluentd는 데이터 수집기로 자리매김한 미들웨어로, 플러그인을 조합하여 다양한 데이터 저장소 사이에서 연계할 수 있다는 점이 특징이다. 커뮤니티 기반에서 수많은 플러그인이 개발되고 있기에 이 장에서 사용하는 카프카, JMX, InfluxDB 모두에 대응할 수 있다는 점도 이번 사례에 알맞다. 여기에서는 Fluentd의 패키지 버전인 `td-agent`를 이용한다.

```bash
$ curl -L https://toolbelt.treasuredata.com/sh/install-redhat-td-agent3.sh | sh
```

설치 후 Fluentd 설정 파일 (`/etc/td-agent/td-agent.conf`)에 카프카 브로커 매트릭스를 얻기 위한 설정 항목을 추가한다. 이번에는 `exec input plugin(in_exec)`을 이용하여 정기적으로 `curl` 명령을 실행해 매트릭스를 얻는 설정을 이용한다. 이 단계에서는 JMX MBean을 필터링하지 않고 한꺼번에 취득하도록 한다.

```xml
<source>
    @type exec
    tag kafka.metrics.broker
    command curl -s 'http://localhost:8778/jolokia/read/kafka.server:*'
    run_interval 10s
    <parse>
        @type json
    </parse>
</source>
```

매트릭스를 이용하여 서비스를 모니터링할 경우 매트릭스를 서버 단위로 집계하고 싶기 때문에 record transformer filter plugin (`filter_record_transformer`)을 이용하여 호스트명의 정보를 추가한다. 또한 매트릭스 정보를 기록할 곳인 카프카의 토픽명에도 이 단계에서 추가해둔다.

`td-agent.conf`에는 다음의 내용을 추가한다.

```xml
<filter kafka.metrics.raw.*>
    @type record_transformer
    <record>
        topic kafka.metrics
        hostname "#{Socket.gethostname}"
    </record>
</filter>
```

JMX에서 취득해 호스트명을 부가한 정보는 Kafka output plugin을 이용하여 카프카의 토픽에 기록한다.

출력할 곳의 토픽명으로 앞서 `filter plugin`에 부가한 토픽 키의 값이 사용된다. 또한 이후 스트림을 처리할 때 매트릭스를 호스트 단위로 집계하기 위해 파티션 키로 `hostname` 값을 이용함으로써 동일 노드의 매트릭스가 동일 컨수머에 전달되도록 설정한다. 또한 알기 쉽게 확인할 수 있도록 데이터는 JSON 형식의 문자열 그대로 보관한다.

`td-agent.conf`에는 다음의 내용을 추가한다.

```xml
<match kafka.metrics.raw.*>
    @type kafka2
    brokers localhost:9092
    topic_key topic
    partition_key_key hostname
    default_message_key nohostname
    max_send_retries 1
    required_acks -1
    <format>
        @type json
    </format>
    <buffer topic>
        flush_interval 10s
    </buffer>
</match>
```

설정이 끝나면 `td-agent` 서비스를 다시 시작해서 반영한다.

```bash
$ sudo systemctl restart td-agent
```

디폴트 설정으로 동작하고 있다면 토픽의 자동 생성이 활성화되어 있기 때문에 이 시점에서 카프카 브로커의 매트릭스가 `kafka.metrics`에 기록되어 있을 것이다. `kafka-console-consumer` 명령을 실행하여 토픽에 정기적으로 정보가 기록되어 있는지 확인해보자.

```bash
$ kafka-console-consumer --bootstrap-server localhost:9092 --topic kafka.metrics

{"request":{"mbean":"kafka.server:*","type":"read"}, ...
  (생략)
```

### 8.4.5 Kafka Streams에 의한 데이터 처리

토픽에 기록한 매트릭스 정보를 Kafka Streams로 처리한다. 간단한 예로 JSON으로 내보낸 정보의 일부를 추출하는 코드를 작성해보자.

우선 `mvn archetype:generate` 명령을 실행하여 프로젝트의 템플릿을 작성한다. 여기서는 4장의 애플리케이션 개발 환경에 맞추어 설명한다.

```bash
$ mvn archetype:generate \
-DgroupId=com.example \
-DartifactId=kafka-metrics-processor \
-DarchetypeArtifactId=maven-archetype-quickstart \
-DinteractiveMode=false

$ cd kafka-metrics-processor
```

디렉터리 트리의 최상위에 배치되어 있는 POM(`pom.xml`)을 편집하여 프로젝트 설정을 추가한다.

**_pom.xml_**

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
  http://maven.apache.org/xsd/maven-4.0.0.xsd">

  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>kafka-metrics-processor</artifactId>
  <version>1.0-SNAPSHOT</version>
  <name>kafka-metrics-processor</name>
  <url>http://maven.apache.org</url>

  <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
  </properties>

  <dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>3.8.1</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka-streams</artifactId>
      <version>2.0.0</version>
    </dependency>
  </dependencies>

</project>
```

추가한 내용은 다음의 두 가지다.

1. Java 8 이상에서 지원되는 람다식을 이용하기 위해 `properties`에 명시적으로 `1.8`을 지정한다.
2. Kafka Streams를 이용하기 위해 `kafka-streams`에 대한 dependency를 추가한다.

다음으로 애플리케이션의 소스 코드를 작성한다.

템플릿으로 배치된 `src/main/java/com/example/App.java`를 StreamingExample1.java로 이름을 바꾸고 내용을 [예제 8-2]로 치환한다.

**_StreamingExample1.java_**

```java
package com.example;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.Consumed;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.Produced;
import org.apache.kafka.streams.kstream.ValueMapper;

import java.util.Arrays;
import java.util.Properties;

public class StreamingExample1 {

    public static void main( String[] args ) {

        Properties config = new Properties();
        config.put(StreamsConfig.APPLICATION_ID_CONFIG, "streaming-example-1");
        config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        ObjectMapper mapper = new ObjectMapper();
        StreamsBuilder builder = new StreamsBuilder();

        KStream<String, String> metrics =
                builder.stream("kafka.metrics",
                        Consumed.with(Serdes.String(), Serdes.String()));

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

        KafkaStreams streams = new KafkaStreams(builder.build(), config);

        streams.start();
    }

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

}
```

소스 코드를 빌드하기 위해서는 프로젝트 디렉터리의 루트로 이도한 후 `mvn package` 명령을 실행한다. 빌드된 프로그램을 포함하는 jar 파일은 `target` 디렉터리 밑에 작성된다.

```bash
$ cd kafka-metrics-processor
$ mvn package
$ ls target
```

이 예제 프로그램은 카프카 2.0.0을 기준으로 작성되었다. 예제 프로그램에서 사용하는 Consumed 클래스의 패키지명이 변경된 영향으로 카프카 버전 1.0 계열에서는 컴파일되지 않는다.

카프카 버전 1.0 계열에서 이 예제 프로그램을 실행하고 싶은 경우에는 예제 코드 시작 부분에서 Consumed 클래스를 import하는 부분을 다음과 같이 수정하면 된다.

```java
import org.apache.kafka.streams.Consumed;
```

빌드에 성공하면 Kafka Streams의 프로그램을 실행하기 전에 먼저 Kafka Streams가 상태를 보관하기 위해 사용하는 디렉터리를 작성한다. 다음 예에서는 `centos`로 하고 있지만 애플리케이션을 실행하는 사용자오 ㅏ그룹을 사용하길 바란다.

```bash
$ sudo mkdir -p /var/lib/kafka-streams/streaming-example-1
$ sudo chown centos:centos /var/lib/kafka-streams/streaming-example-1
```

준비됐으면 `kafka-run-class` 명령으로 프로그램을 실행한다. jar 파일로의 클래스 경로는 `CLASSPATH` 라는 환경 변수를 통해 지정할 수 있다.

```bash
$ CLASSPATH=target/kafka-metrics-processor-1.0-SNAPSHOT.jar \
kafka-run-class com.example.StreamingExample1
```

위의 프로그램을 실행한 상태에서 별도의 터미널을 실행한 후 `kafka-console-consumer` 명령을 사용하여 `kafka.metrics.processed`의 내용을 출력해본다. 잘 동작하고 있다면 예제 프로그램이 출력한 JSON이 정기적으로 출력될 것이다.

```bash
$ kafka-console-consumer --bootstrap-server localhost:9092 \
--topic kafka.metrics.processed

{"hostname":"host1","timestamp":1533796782,"BytesIn":424114943}
  (생략)
```

예제 프로그램을 정지하려면 [Ctrl] + [C]를 입력한다.
