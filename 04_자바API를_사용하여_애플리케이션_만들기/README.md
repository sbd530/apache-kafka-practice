# 4장 자바 API를 사용하여 애플리케이션 만들기

## 4.1 이 장의 내용

이 장에서는 자바 API를 이용하여 애플리케이션을 설명한다. 카프카는 자바 API를 제공하고 있으며 이를 이용하여 카프카와 메시지를 송수신하는 애플리케이션을 만들 수 있다. 카프카의 자바 API에는 프로듀서용과 컨수머용 API가 준비되어 있다. 이 장에서는 각각의 애플리케이션을 작성하여 송신에서부터 수신까지의 일련의 흐름을 살펴본다.

## 4.2 애플리케이션 개발 환경 준비

### 4.2.1 개발 환경

이 책에서는 리눅스 환경에서 애플리케이션을 개발하는 것을 전제로 한다.

카프카에서 자바 API를 이용하여 애플리케이션을 개발할 때 JDK가 필요한데 이 책에서는 Oracle JDK 8을 사용한다(3장 참고). 그리고 애플리케이션 빌드에 필요한 라이브러리를 인터넷에서 다운받으므로 인터넷에 연결된 상태여야 한다.

이 장에서는 개발 환경 호스트명을 dev로 하여 명령어 부분에 작성하며, 앞 장에서 구축한 카프카 환경을 사용하여 애플리케이션을 동작시킨다. 이 서버에서 실행하는 명령을 작성한 부분에 있는 서버의 호스트명은 앞 장의 여러 대의 서버로 구성된 카프카 환경의 호스트명을 따른다.

### 4.2.2 메이븐 설치

이 책에서는 자바 애플리케이션 개발에 아파치 메이븐(이하 메이븐)을 이용한다. 메이븐은 카프카와 마찬가지로 아파치소프트웨어재단에서 개발한 소프트웨어 프로젝트 관리 도구다. 이 책에서는 집필 시점 최신 버전인 3.5.4를 사용한다.

메이븐 사이트에서 다운로드 페이지를 열고 'Binary tar.gz archive'를 다운로드 한다. 여기에서 다운로드나 파일을 /tmp에 배치한다.

> 나의 경우 아래의 URL로 들어가 wget 명령으로 3.8.1 버전을 다운로드했다.
> https://maven.apache.org/download.cgi
>
> ```bash
> $ wget https://mirror.navercorp.com/apache/maven/maven-3/3.8.1/binaries/apache-maven-3.8.1-bin.tar.gz
> ```

이 파일을 다음 명령으로 /opt 아래에 압축을 푼다.

```bash
(dev)$ sudo tar zxvf /tmp/apache-maven-3.5.4-bin.tar.gz -C /opt
```

다음으로 필요한 환경 변수를 정의한다. /etc/profile.d/maven.sh를 작성하고 다음과 같이 실행한다.

```shell
export MAVEN_HOME=/opt/apache-maven-3.5.4
export PATH=$PATH:$MAVEN_HOME/bin
```

설정한 환경 변수를 반영한다.

```bash
(dev)$ source /etc/profile.d/maven.sh
```

메이븐이 제대로 작동하는지 확인하자. 표시 내용은 실행환경에 따라 다를 수 있다.

```bash
(dev)$ mvn -version
Apache Maven 3.5.4 (생략)
Maven home: /opt/apache-maven-3.5.4
(생략)
```

### 4.2.3 메이븐으로 프로젝트 작성하기

애플리케이션을 만드려면 먼저 메이븐으로 프로젝트를 작성한다. 메이븐 프로젝트에는 애플리케이션의 소스 코드 뿐만 아니라 애플리케이션 빌드에 필요한 정보도 포함된다. 이 절에서는 프로젝트 디렉터리는 사용자의 홈 디렉터리 아래에 설정한다.

현재 디렉터리를 사용자의 홈 디렉터리로 하여 메이븐 프로젝트를 작성한다.

```bash
(dev)$ mvn archetype:generate -DarchetypeGroupId=org.apache.maven.archetypes \
> -DarchetypeArtifactId=maven-archetype-simple -DgroupId=com.example.chapter4 \
> -DartifactId=firstapp -Dversion=1.0-SNAPSHOT -DinteractiveMode=false

(생략)
[INFO] BUILD SUCCESS
(생략)
```

빌드된 프로젝트에서 App.java와 AppTest.java는 이 장의 내용에서는 필요하지 않으므로 삭제한다.

```bash
(dev)$ rm ~/firstapp/src/main/java/com/example/chapter4/App.java
(dev)$ rm ~/firstapp/src/test/java/com/example/chapter4/AppTest.java
```

### 4.2.4 빌드 정보 작성

다음으로 메이븐 빌드 정보를 작성한다. 카프카의 자바 API를 이용하기 위해서 필요한 라이브러리가 있으므로 의존관계로 설정해야 한다. 메이븐에서는 pom.xml에 의존관계 등의 빌드 정보를 작성한다.

이 책에서는 카프카 환경을 구축하기 위해 컨플루언트 플랫폼을 이용했다. 따라서 빌드할 때 사용하는 라이브러리도 컨플루언트 플랫폼에 맞는 것을 이용한다. 여기에서는 그러한 라이브러리로 컨플루언트가 제공하는 리포지토리 정보를 추가한다.

앞서 만든 파일 중 pom.xml에 다음과 같이 컨플루언트 리포지토리 정보를 추가한다. 이 정보는 project 태그 안에 작성한다. 이미 repositories 태그가 pom.xml에 존재하는 경우 repository 태그와 그 내용을 기존 repositories 태그 안에 추가하면 된다.

**~/firstapp/pom.xml**

```xml
<repositories>
    <repository>
        <id>confluent</id>
        <url>https://packages.confluent.io/maven/</url>
    </repository>
</repositories>
```

다음으로 필요한 카프카 라이브러리의 의존관계를 설정한다. 리포지토리 정보와 마찬가지로 pom.xml의 project 태그 안에 다음의 내용을 추가한다. dependencies 태그가 이미 존재하는 경우는 다음의 dependency 태그와 그 내용을 기존의 dependencies 태그 안에 추가한다.

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka_2.11</artifactId>
        <version>2.0.0-cp1</version>
    </dependency>
</dependencies>
```

이 장에서는 빌드한 애플리케이션을 실행하기 쉽도록 의존관계의 라이브러리를 포함한 jar 파일(Fat JAR)을 작성한다. pom.xml의 project 태그 속에 Fat JAR 작성을 위한 플러그인 설정을 추가한다. 지금까지와 마찬가지로 build 태그 등의 기존 태그가 존재하면 그 속에 추가한다.

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>2.6</version>
            <configuration>
                <descriptorRefs>
                    <descriptorRef>jar-with-dependencies</descriptorRef>
                </descriptorRefs>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

> 직접 해본 결과, 자동으로 생성된 pluginManagement 태그 안에 넣었더니 제대로 Fat JAR가 생성되지 않았다. pluginManagement 태그 속이 아닌 build 태그 하위에 따로 plugins 태그를 만들어서 작성하도록 주의하자.

Fat JAR는 프레임워크에 따라 이용할 수 없는 경우나 사용을 권장하지 않는 경우도 있다. 그런 경우 이설정으로 하지 않고 프레임워크의 사용 방법에 따라 클래스 경로를 설정하여 실행하기 바란다.

이것으로 애플리케이션 개발 환경 준비가 끝났다.

> IntelliJ IDEA로 작업하던 중 아래의 plugin 이 인식되지 않았다.
>
> ```xml
> <reporting>
>   <plugins>
>       <plugin>
>           <artifactId>maven-project-info-reports-plugin</artifactId>
>       </plugin>
>   </plugins>
> </reporting>
> ```
>
> 이에 maven repository를 검색하여 dependency를 추가함으로써 해결하였다.
>
> ```xml
> <dependencies>
>   ...
>   <dependency>
>       <groupId>org.apache.maven.plugins</groupId>
>       <artifactId>maven-project-info-reports-plugin</artifactId>
>       <version>3.1.2</version>
>       <type>maven-plugin</type>
>   </dependency>
> </dependencies>
> ```

---

_<center>커뮤니티 버전의 카프카 라이브러리를 사용하는 애플리케이션의 빌드</center>_

지금까지의 설명에서는 사용하는 카프카 클러스터의 환경에 맞추어 컨플루언트가 제공하고 있는 카프카의 라이브러리를 사용하여 애플리케이션을 빌드했지만, 카프카 개발 커뮤니티(아파치소프트웨어재단)가 제공하는 라이브러리를 사용하여 빌드할 수도 있다. 그것을 사용하는 경우는 의존관계(dependencies 태그 안)에 다음과 같이 작성한다.

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka_2.11</artifactId>
        <version>2.0.0-cp1</version>
    </dependency>
</dependencies>
```

version 태그는 사용할 카프카의 버전에 맞게 적절하게 수정하기 바란다. 참고로 Confluent Platform 버전과 커뮤니티 버전의 카프카 버전은 동일하지 않으므로 주의해야 한다. 만약 컨플루언트가 제공하는 라이브러리를 사용하지 않는 겨우 이 항목 첫머리에서 소개한 repositories 태그에 컨플루언트 리포지토리 정보를 쓸 필요는 없다.

---

<br><br>

## 4.3 프로듀서 애플리케이션 개발

이 절에서는 카프카의 자바 API를 사용하여 프로듀서 애플리케이션을 개발한다. 여기에서는 간단한 예제로 1에서 100까지 숫자를 문자열로 변환한 것을 메시지로 보내는 애플리케이션을 만들어본다.

### 4.3.1 프로듀서 애플리케이션 소스 코드

프로듀서 애플리케이션의 소스 코드부터 작성한다. 프로젝트 디렉터리의 src/main/java/com/example/chapter4에서 FirstAppProducer.java 라는 파일을 만들고 아래와 같이 작성한다. 참고로 kafka-broker01 같은 서버명은 3장에서 구성한 여러 대의 카프카 클러스터 환경을 전제로 작성했다(독자 환경에 맞게 변경하면 된다).

> 나의 경우 IntelliJ IDEA와 git remote를 사용하여 코드를 작성했다.

**com.example.chapter4.FirstAppProducer.java**

```java
package com.example.chapter4;

import org.apache.kafka.clients.producer.*;

import java.util.Properties;

public class FirstAppProducer {

    private static String topicName = "first-app";

    public static void main(String[] args) {
        // [1] KafkaProducer Configuration
        Properties conf = new Properties();
        /*conf.setProperty("bootstrap.servers", "kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092");*/
        conf.setProperty("bootstrap.servers", "localhost:9092");
        conf.setProperty("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        conf.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        // [2] Object for producing messages to KafkaCluster
        Producer<Integer, String> producer = new KafkaProducer<>(conf);

        int key;
        String value;

        for (int i = 1; i <= 100; i++) {
            key = i;
            value = String.valueOf(i);

            // [3] Record to be produced
            ProducerRecord<Integer, String> record = new ProducerRecord<>(topicName, key, value);

            // [4] Callback for Ack after Producing
            producer.send(record, new Callback() {
                @Override
                public void onCompletion(RecordMetadata recordMetadata, Exception e) {
                    if (recordMetadata != null) {
                        // Success
                        String infoString = String.format("Success partition:%d, offset:%d", recordMetadata.partition(), recordMetadata.offset());
                        System.out.println(infoString);
                    } else {
                        // Fail
                        String infoString = String.format("Failed:%s", e.getMessage());
                        System.out.println(infoString);
                    }
                }
            });
        }

        // [5] Close KafkaProducer and exit
        producer.close();
    }
}
```

### 4.3.2 프로듀서 애플리케이션 빌드 및 실행

작성한 애플리케이션을 빌드하여 동작시켜보자. 프로젝트의 루트 디렉터리(홈 디렉터리 아래에 생성되는 firstapp 디렉터리)로 현재 디렉터리를 이동하고 다음 명령으로 애플리케이션을 빌드한다.

```bash
(dev)$ mvn package -DskipTests
```

첫 회의 빌드는 빌드에 필요한 라이브러리를 인터넷에서 다운로드하기 때문에 시간이 걸릴 수이 있다. 제대로 빌드되면 프로젝트의 루트 디렉터리의 target 디렉터리 아래 다음 2개의 jar 파일이 생성된다.

- firstapp-1.0-SNAPSHOT.jar
- firstapp-1.0-SNAPSHOT-jar-with-dependecies.jar

이 중 jar-with-dependencies라는 파일명에 들어 있는 것이 Fat JAR이다. 이 JAR 파일을 사용하여 애플리케이션을 실행한다.

프로듀서 애플리케이션 실행 전에 프로듀서 애플리케이션이 메시지를 송신하는 토픽을 미리 작성하고 다음 명령을 카프카 클라이언트에서 실행한다.(3장 참고)

```bash
(kafka-client)$ kafka-topics --zookeeper localhost:2181 \
> --create --topic first-app --partitions 3 --replication-factor 3
Created topic "first-app".
```

프로듀서 애플리케이션에서 보낸 메시지를 확인하기 위해 여기에서는 Kafka Console Consumer를 사용한다(3장 참고). 컨수머 서버에서는 Kafka Console Consumer를 다음 명령으로 실행한다.

```bash
(consumber-client)$ kafka-console-consumer --bootstrap-server kafka-broker01:9092, \
> kafka-broker02:9092,kafka-broker03:9092 --topic first-app
```

그럼 작성한 프로듀서 애플리케이션을 실행해보자. 조금 전 빌드한 jar 파일 중 firstapp-1.0-SNAPSHOT-jar-with-dependencies.jar 를 프로듀서 서버에 전송하고, 다음 명령으로 프로듀서 애플리케이션을 실행한다. 여기에서는 jar 파일은 사용자의 홈 디렉터리 아래에 있는 것으로 한다. 컨수머 서버와 프로듀서 서버가 동일한 서버인 경우 위의 절차로 Kafka Console Consumer를 시작한 것과는 다른 별도의 콘솔을 열고 다음 명령을 실행한다.

```bash
(producer-client)$ java -cp ~/firstapp-1.0-SNAPSHOT-jar-with-dependencies.jar \
> com.example.chapter3.FirstAppProducer
log4j:WARN No appenders could bd found ...
(생략)
Success partition:0, offset:0
Success partition:0, offset:1
(생략)
```

미리 실행한 Kafka Console Consumer에 제대로 결과가 나왔는지 확인한다.

```bash
(consumer-client)$ kafka-topics --zookeeper (생략)
2
3
9
16
(생략)
```

Kafka Console Consumer에 표시되어 있는 숫자는 작성한 프로듀서 애플리케이션에서 보낸 메시지다. 오류 발생 없이 숫자 메시지가 나열되어 있으면 프로듀서에서 제대로 메시지를 보낸 것이다.

이 결과처럼 Kafka Console Consumer 결과는 1에서 100까지 순서대로 나열되지 않을 수도 있다. 이것은 프로듀서에서 보낸 메시지가 지정된 토픽 중 하나의 파티션에 전송되지만, 카프카에서 메시지 순서는 동일한 파티션 안에서만 보증하고 있기 때문에 다른 파티션에 보낸 메시지 처리 순서는 취득 타이밍 등에 따라 다르기 때문이다.

프로듀서 애플리케이션 동작을 확인했으면 Kafka Console Consumer를 [Ctrl] + [C]로 종료한다. 이번에 만든 프로듀서 애플리케이션은 메시지 송신이 완료되면 정지하기 때문에 사용자에 의한 정지 작업은 필요 없다.<br><br><br>

## 4.4 프로듀서 애플리케이션의 핵심 부분

앞에서 만든 프로듀서 애플리케이션의 소스 코드에서 핵심 부분을 살펴보자.

### 4.4.1 KafkaProducer 객체 생성

카프카의 자바 API로 메시지를 송신하기 위해서는 KafkaProducer 객체를 이용한다. 예제 소스 코드에서 [1]의 KafkaProducer에 필요한 설정을 하고 [2]로 객체를 생성한다.

```java
// [1] KafkaProducer Configuration
Properties conf = new Properties();
/*conf.setProperty("bootstrap.servers", "kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092");*/
conf.setProperty("bootstrap.servers", "localhost:9092");
conf.setProperty("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
conf.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// [2] Object for producing messages to KafkaCluster
Producer<Integer, String> producer = new KafkaProducer<>(conf);
```

우선 KafkaProducer에 필요한 설정부터 살펴보자. 여기에서는 동작하는 데 필요한 최소한의 설정만 하고 있다. 그 이외의 항목은 카프카 문서를 참고하기 바란다.

**bootstrap.servers**

bootstrap.servers에서는 작성할 KafkaProducer가 접속하는 브로커의 호스트명과 포트 번호를 지정하고 있다.

3장에서 소개한 Kafka Console Producer 옵션 중 broker-list와 마찬가지로 <호스트명>:<포트 번호>의 형식으로 작성하고 여러 브로커를 지정할 때는 쉼표로 연결한다.

**key.serializer, value.serializer**

카프카에서는 모든 메시지가 직렬화된 상태로 전송된다. key.serializer와 value.serializer는 이 직렬화 처리에 이용되는 시리얼라이저 클래스를 지정한다.

key.serializer는 메시지의 Key를 직렬화하는 데 사용되고, value.serializer는 메시지의 Value를 직렬화하는 데 사용된다.

카프카에는 기본적인 시리얼라이저가 준비되어 있어 사용 가능하다. 또한 준비되어 있지 않은 데이터 형에 대해서도 직접 시리얼라이저를 만들어 사용할 수 있다.

**카프카에 있는 시리얼라이저**

| 데이터 타입 | 카프카에서 제공되고 있는 시리얼라이저                      |
| ----------- | ---------------------------------------------------------- |
| Short       | org.apache.kafka.common.serialization.ShortSerializer      |
| Integer     | org.apache.kafka.common.serialization.IntegerSerializer    |
| Long        | org.apache.kafka.common.serialization.LongSerializer       |
| Float       | org.apache.kafka.common.serialization.FloatSerializer      |
| Double      | org.apache.kafka.common.serialization.DoubleSerializer     |
| String      | org.apache.kafka.common.serialization.StringSerializer     |
| ByteArray   | org.apache.kafka.common.serialization.ByteArraySerializer  |
| ByteBuffer  | org.apache.kafka.common.serialization.ByteBufferSerializer |
| Bytes       | org.apache.kafka.common.serialization.BytesSerializer      |

여기까지의 설정을 이용하여 [2]에서 KafkaProducer의 객체를 작성한다. KafkaProducer는 프로듀서 인터페이스를 구현하기 위해 변수의 형을 Producer로 하고 있다.

KafkaProducer의 객체를 작성할 때 형 파라미터를 지정하고 있다. 이것은 각각 송신하는 메시지의 Key와 Value의 형을 나타낸다. 여기에선느 메시지의 Key를 정수형(Integer), Value를 문자열형(String)으로 하고 있다. 여기에서 지정한 형은 먼저 지정한 시리얼라이저와 대응하고 있어야 한다.

### 4.4.2 메시지 송신하기

작성한 KafkaProducer 객체를 사용하여 메시지를 송신한다. 소스 코드에서 [3]에서 ProducerRecord라는 객체를 만들고 있다.

```java
// [3] Message to be produced
ProducerRecord<Integer, String> record = new ProducerRecord<>(topicName, key, value);
```

KafkaProducer를 이용하여 메시지를 보낼 때는 송신 메시지를 이 ProducerRecord라는 객체에 저장한다. 이때 메시지의 Key, Value 외에 송신처의 토픽도 함께 등록한다. ProducerRecord에도 형 파라미터가 있으므로 KafkaProducer의 객체를 만들 때와 동일하게 지정한다.

작성한 ProducerRecord 객체는 소스 코드 [4]에서 송신된다.

```java
// [4] Callback for Ack after Producing
producer.send(record, new Callback() {
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
        if (recordMetadata != null) {
            // Success
            String infoString = String.format("Success partition:%d, offset:%d", recordMetadata.partition(), recordMetadata.offset());
            System.out.println(infoString);
        } else {
            // Fail
            String infoString = String.format("Failed:%s", e.getMessage());
            System.out.println(infoString);
        }
    }
});
```

여기에서는 ProducerRecord 객체뿐만 아니라 Callback 클래스를 구현하여 KafkaProducer에 전달하고 있다. 이 Callback 클래스에서 구현하고 있는 onCompletion 메서드에서는 송신을 완료했을 때 실행되어야 할 처리를 하고 있다.

KafkaProducer의 송신 처리는 비동기적으로 이루어지기 때문에 _send_ 메서드를 호출했을 때 발생하지 않는다. _send_ 메서드의 처리는 KafkaProducer의 송신 큐에 메시지를 넣을 뿐이다. 송신 큐에 넣은 메시지는 사용자의 애플리케이션과는 별도의 스레드에서 순차적으로 송신된다.

메시지가 송신된 경우 카프카 클러스터에서 Ack가 반환된다. Callback 클래스의 메서드는 그 Ack를 수신했을 때 처리된다.

Callback 클래스의 메서드는 메시지 송신에 성공했을 때와 실패했을 때 동일한 내용이 호출된다. 메시지 송신에 성공했을 때는 메서드 인수인 RecordMetadata가 null이 아닌 객체이며 Exception은 null이 된다. 메시지 송신에 실패했을 대는 RecordMetadata가 null이 되고, Exception은 null 이외의 객체가 된다. 따라서 예제 소스와 같이 Exception이 null인지 아닌지로 메시지 송신에 성공했을 때와 실패했을 때의 처리를 분기할 필요가 있다.

마지막으로 [5]에서 송신한 KafkaProducer를 종료한다.

```java
// [5] Close KafkaProducer
producer.close();
```

close 메서드 호출로 KafkaProducer 안의 송신 큐에 남아 있는 메시지도 송신되어 안전하게 애플리케이션을 종료할 수 있다.<br><br><br>

## 4.5 컨슈머 애플리케이션 개발

지금까지 프로듀서 애플리케이션 작성 방법을 소개했다. 다음은 메시지를 수신하는 컨슈머 애플리케이션 개발 방법을 소개한다. 여기에서는 1초마다 받은 메시지를 콘솔에 표시하는 컨슈머 애플리케이션을 만든다.

## 4.5.1 컨슈머 애플리케이션 소스 코드

컨슈머 애플리케이션의 소스 코드를 작성하자. 프로젝트 디렉터리에 src/main/java/com/example/chapter4에 FirstAppConsumer.java라는 파일을 만들고 아래와 같이 작성한다.

```java
package com.example.chapter4;

import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import java.util.*;

public class FirstAppConsumer {

    private static String topicName = "first-app";

    public static void main(String[] args) {
        // [1] KafkaConsumer Configuration
        Properties conf = new Properties();
        /*conf.setProperty("bootstrap.servers", "kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092");*/
        conf.setProperty("bootstrap.servers", "localhost:9092");
        conf.setProperty("group.id", "FirstAppConsumerApp");
        conf.setProperty("enable.auto.commit", "false");
        conf.setProperty("key.deserializer", "org.apache.kafka.common.serialization.IntegerDeserializer");
        conf.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        // [2] Object for consuming messages from KafkaCluster
        Consumer<Integer, String> consumer = new KafkaConsumer<>(conf);

        // [3] Register the subscribing Topic
        consumer.subscribe(Collections.singletonList(topicName));

        for (int count = 0; count < 300; count++) {
            // [4] Consume messages and print on console
            ConsumerRecords<Integer, String> records = consumer.poll(1);
            for (ConsumerRecord<Integer, String> record : records) {
                String msgString = String.format("key:%d, value: %s", record.key(), record.value());
                System.out.println(msgString);

                // [5] Commit Offset of completed message
                TopicPartition tp = new TopicPartition(record.topic(), record.partition());
                OffsetAndMetadata oam = new OffsetAndMetadata(record.offset() + 1);
                Map<TopicPartition, OffsetAndMetadata> commitInfo = Collections.singletonMap(tp, oam);
                consumer.commitSync(commitInfo);
            }
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        // [6] Close KafkaConsumer
        consumer.close();
    }
}
```

### 4.5.2 컨슈머 애플리케이션의 빌드 및 실행

프로듀서 애플리케이션의 경우와 마찬가지로 컨수머 애플리케이션을 빌드하고 실행한다. 프로젝트 루트 디렉터리로 현재 디렉터리를 변경하고 다음 명령으로 애플리케이션을 빌드한다.

```bash
$ mvn package -DskipTests
```

프로듀서 애플리케이션 경우처럼 2개의 jar 파일이 target 디렉터리 안에 생성된다. 빌드 전부터 이미 jar 파일이 존재하는 경우 필요에 따라 덮어 쓴다.

- firstapp-1.0-SNAPSHOT.jar
- firstapp-1.0-SNAPSHOT-jar-with-dependecies.jar

이 중 두 번째 Fat JAR를 사용하여 컨수머 애플리케이션을 실행한다. firstapp-1.0-SNAPSHOT-jar-with-dependencies.jar를 컨수머 서버에 배치하고 다음 명령으로 컨수머 애플리케이션을 실행한다.

```bash
(consumer-client)$ java -cp ~/firstapp-1.0-SNAPSHOT-jar-with-dependencies.jar \
> com.example.chapter4.FirstAppConsumer
```

다음으로 컨수머 애플리케이션의 동작 확인을 위해 먼저 작성한 프로듀서 애플리케이션을 실행한다. 프로듀서 애플리케이션에서 메시지를 보내고 앞서 실행한 컨수머 애플리케이션에서 그 메시지를 수신하여 동작을 확인한다.

프로듀서 버버에서 다음 명령을 실행하여 이전 절에서 작성한 프로듀서 애플리케이션을 실행한다. 컨수머 서버와 프로듀서 서버가 동일한 경우 컨수머 애플리케이션을 실행시킨 것과는 별도의 콘솔을 열고 실행한다.

```bash
(prducer-client)$ java -cp ~/firstapp-1.0-SNAPSHOT-jar-with-dependencies.jar \
> com.example.chapter4.FirstAppProducer
(생략)
```

실행한 컨수머 애플리케이션의 콘솔에 프로듀서 애플리케이션에서 보낸 메시지가 제대로 표시되는지 확인한다.

```bash
(consumer-client)$ java -cp ~/firstapp-1.0-SNAPSHOT-jar-with-dependencies.jar \
> com.example.chapter4.FirstAppConsumer
(생략)
key:2, value: 2
key:5, value: 5
key:6, value: 6
key:12, value: 12
(생략)
```

여기에서 작성한 컨수머 애플리케이션은 실행 5분 후 정지하도록 되어 있지만 동작을 확인하고 도중에 종료할 경우 [Ctrl] + [C]를 입력한다.<br><br><br>

## 4.6 컨수머 애플리케이션 핵심 부분

개밣한 컨수머 애플리케이션에서 핵심 부분을 살펴보자

### 4.6.1 KafkaConsumer 객체 작성

카프카의 자바 API에서 메시지 수신은 KafkaConsumer 객체를 이용한다. 예제 애플리케이션의 소스 코드 [1]에서 필요ㅛ한 설정을 하고, [2]에서 KafkaConsumer 객체를 작성한다.

```java
// [1] KafkaConsumer Configuration
Properties conf = new Properties();
/*conf.setProperty("bootstrap.servers",
"kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092");*/
conf.setProperty("bootstrap.servers", "localhost:9092");
conf.setProperty("group.id", "FirstAppConsumerApp");
conf.setProperty("enable.auto.commit", "false");
conf.setProperty("key.deserializer",
"org.apache.kafka.common.serialization.IntegerDeserializer");
conf.setProperty("value.deserializer",
"org.apache.kafka.common.serialization.StringDeserializer");
```

KafkaConsumer에 필요한 설정을 살펴보자. 프로듀서 애플리케이션과 마찬가지로 동작을 위해 필요한 최소한의 내용만 작성했다. 그 외의 항목은 카프카 문서를 참고하자.

**bootstrap.servers**

접속할 브로커의 호스트명과 포트 번호를 지정하고 있다. 3장에서 소개한 Kafka Console Consumer와 똑같이 <호스트명>:<포트 번호>의 형태로 작성하고 여러 브로커를 지정할 때는 쉼표로 연결한다.

**group.id**

작성할 KafkaConsumer가 속한 Consumer Group을 지정한다(11장 참고).

**enable.auto.commit**

오프셋 커밋을 자동으로 실행할지의 여부를 지정한다(11장 참고). 여기에선느 수동으로 오프셋을 커밋하기(Manual Offset Commit) 때문에 false로 했다.

**key.deserializer, value.deserializer**

카프카에 송신되는 모든 메시지가 직렬화된다는 것은 프로듀서 애플리케이션에서 소개했다. key.deserializer와 value.deserializer는 컨수머의 사용자 처리에 전달되기 전에 실시되는 디시리얼라이즈(역직렬화) 처리에 이용되는 역직렬화 클래스를 지정한다. 시리얼라이저와 마찬가지로 카프카에는 시리얼라이저와 짝을 이루는 기본적인 타입의 디시리얼라이저가 준비되어 있다. 여기에서 지저하는 디시리얼라이저는 프로듀서에서 지정한 시리얼라이저에 해당하는 것이어야 한다.<br><br><br>

여기까지 설정을 이용하여 예제 코드 [2]에서 KafkaConsumer의 객체를 작성한다. KafkaConsumer는 컨슈머 인터페이스 구현을 위해 변수 타입을 컨수머로 하고 있다.

```java
// [2] Object for consuming messages from KafkaCluster
Consumer<Integer, String> consumer = new KafkaConsumer<>(conf);
```

KafkaConsumer 객체를 작성할 때 타입 파라미터를 지정하고 있다. 이것은 프로듀서 애플리케이션의 KafkaProducer에 지정한 것과 마찬가지로 각각 수신하는 메시지 Key와 Value의 타입을 나타낸다. 이 타입은 먼저 지정한 디시리얼라이저와 프로듀서에서 보낸 메시지에도 대응해야한다. 여기에서는 먼저 작성한 프로듀서 애플리케이션에 맞춰 Key를 정수형(Integer)으로 지정하고, Value를 문자열형(String)으로 지정하고 있다.

### 4.6.2 메시지를 수신하기

작성한 KafkaConsumer 객체를 이용하여 메시지를 수신한다. KafkaConsumer에서는 메시지를 수신하는 토픽을 구독할 필요가 있다. 예제 코드 [3]에서 _subscribe_ 메서드를 호출함으로써 실시하고 있다. 이 경우는 _subscribe_ 메서드에 전달하는 리스트에 여러 토픽을 등록함으로써 여러 토픽을 구독할 수도 있다.

```java
// [3] Register the subscribing Topic
// Case of single topic
consumer.subscribe(Collections.singletonList(topicName));
```

```java
// Case of multiple topics
List<String> topicList = new ArrayList<>(1);
topicList.add(topicName);
consumer.subscribe(topicList);
```

토픽을 수신한 후에는 메시지를 받는다. 예제 코드 [4]에서 _poll_ 메서드를 호출하여 메시지를 얻는다.

```java
// [4] Consume messages and print on console
ConsumerRecords<Integer, String> records = consumer.poll(1);
```

이때 메시지는 ConsumerRecords라는 객체로 전달된다. 이 ConsumerRecords 객체에는 숫니된 여러 메시지의 Key, Value, 타임스탬프 등 메타 데이터가 포함되어 있다. ConsumerRecords에 포함된 여러 메시지를 for문으로 순서대로 처리해 콘솔에 출력하고 있다.

예제 코드 [5]에서 오프셋을 커밋하고 있다.

```java
// [5] Commit Offset of completed message
TopicPartition tp = new TopicPartition(record.topic(), record.partition());
OffsetAndMetadata oam = new OffsetAndMetadata(record.offset() + 1);
Map<TopicPartition, OffsetAndMetadata> commitInfo = Collections.singletonMap(tp, oam);
consumer.commitSync(commitInfo);
```

컨수머 설정에서 Manual Offset Commit을 하고 있기 때문에 애플리케이션에서 적절한 타이밍에 오프셋 커밋을 명시적으로 실행할 필요가 있다. 여기에서는 하나의 메시지 처리가 완료될 때마다 오프셋을 커밋한다. Auto Offset Commit을 하는 설정의 경우 이 코드는 필요하지 않다(11장 참고).

여기에서는 오프셋 커밋 정보가 카프카 클러스터로 기록이 완료될 때 까지 처리를 기다리는 _commitSync_ 메서드를 이용하고 있다. 비동기적으로 처리하고, 처리 완료를 기다리지 않고 다음 처리로 진행하는 *commitAsync*라는 메서드도 있다.

마지막으로 [6]에서 애플리케이션 종료 직전에 KafkaConsumer를 닫고 있다.

```java
// [6] Close KafkaConsumer
consumer.close();
```

이로 인해 처리 중인 오프셋이 완료될 때까지 기다렸다가 안전하게 애플리케이션을 종료할 수 있다.

## 4.7 정리

이 장에서는 카프카의 표준 API를 이용한 프로듀서/컨수머 애플리케이션 개발 방법을 소개했다. 카프카의 메시지 송수신에 대응하는 외부 도구는 이미 많이 있으며 그중 대부분은 여기서 소개한 카프카의 API를 사용하고 있다. 이 장의 내용은 카프카 API를 이용한 애플리케이션 개발은 물론 외부 도구를 사용하는 경우에도 그 동작 원리를 이해하는 데 도움될 것이다.
