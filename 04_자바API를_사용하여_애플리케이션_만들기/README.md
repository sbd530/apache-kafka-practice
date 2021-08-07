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
        // KafkaProducer Configuration
        Properties conf = new Properties();
        /*conf.setProperty("bootstrap.servers", "kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092");*/
        conf.setProperty("bootstrap.servers", "localhost:9092");
        conf.setProperty("key.serializer", "org.apache.kafka.common.serialization.IntegerSerializer");
        conf.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        // Object for producing messages to KafkaCluster
        Producer<Integer, String> producer = new KafkaProducer<>(conf);

        int key;
        String value;

        for (int i = 1; i <= 100; i++) {
            key = i;
            value = String.valueOf(i);

            // Record to be produced
            ProducerRecord<Integer, String> record = new ProducerRecord<>(topicName, key, value);

            // Callback for Ack after Producing
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

        // Close KafkaProducer and exit
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
