# 3장 카프카 설치

## 3.1 이 장의 내용

이 장에서는 카프카 배포판인 컨플루언트 플랫폼을 이용한 카프카 클러스터 구축 방법을 소개한다. 카프카 클러스터는 1대 이상의 브로커로 구성되는 점을 고려하여 서버를 1대만 사용하는 경우와 여러 대를 사용하는 경우에 대한 각각의 구축 방법을 소개한다. 여기서 만든 카프카 클러스터는 4장 이후 예제로 사용한다.

## 3.1 카프카 클러스터 환경 구축하기

### 3.2.1 클러스터 구성

카프카는 1대 이상의 브로커로 이루어진 **카프카 클러스터**, 프로듀서, 컨수머와 카프카 클러스터를 실행하는 **카프카 클라이언트**로 구성된다. 카프카 클라이언트는 카프카 클러스터의 상태 관리 및 운영을 위한 다양한 조작에 필요하다. 이 장에서는 여러 대의 서버에서 카프카 클러스터를 구축하는 예로 카프카 클러스터에 3대의 서버를 이용하는 경우와 1대의 서버에 모두 설치하는 경우의 각 단계를 소개한다.<br><br>

#### <서버가 3대인 경우>

<table>
  <tr>
    <td>호스트명</td>
    <td>역할</td>
    <td>설명</td>
  </tr>
  <tr>
    <td>kafka-broker01</td>
    <td>브로커</td>
    <td rowspan="3">서버 3대에서 카프카 클러스터를 구축한다.</td>
  </tr>
  <tr>
    <td>kafka-broker02</td>
    <td>브로커</td>
  </tr>
  <tr>
    <td>kafka-broker03</td>
    <td>브로커</td>
  </tr>
  <tr>
    <td>producer-client</td>
    <td>프로듀서</td>
    <td>카프카에 메시지를 송신한다.</td>
  </tr>
  <tr>
    <td>consumer-client</td>
    <td>컨수머</td>
    <td>카프카에서 메시지를 수신한다.</td>
  </tr>
  <tr>
    <td>kafka-client</td>
    <td>카프카 클라이언트</td>
    <td>카프카 클러스터의 상태 관리 및 운영을 위한 각종 조작을 처리한다.</td>
  </tr>
</table>

#### <서버가 1대인 경우>

<table>
    <tr>
        <td>호스트명</td>
        <td>역할</td>
        <td>설명</td>
    </tr>
    <tr>
        <td>kafka-server</td>
        <td>카프카 클러스터/클라이언트</td>
        <td>브로커, 프로듀서, 컨수머, 카프카 클라이언트의 모든 역할을 담당한다.</td>
    </tr>
</table>
<br>

### 3.2.2 각 서버의 소프트웨어 구성

카프카 클러스터는 1대 이상의 브로커와 주키퍼로 구성되어 있다. 이 장에서 구축할 카프카 클러스터는 브로커와 주키퍼를 동일 서버에 함께 설치하는 구성으로 한다. 또한 프로듀서, 컨수머, 카프카 클라이언트에는 동작에 필요한 라이브러리와 도구를 설치한다. 프로듀서, 컨수머에 필요한 소프트웨어는 사용하는 미들웨어 등에 따라 달라진다.<br>
여기에서는 클러스터를 구성하는 3대 모두에 카프카 브로커와 주키퍼를 설치한다. 주키퍼는 지속적인 서비스를 위해 항상 과반수가 동작하고 있어야 한다. 이러한 환경에서 주키퍼는 홀수의 노드 수가 바람직하다.<br>
실제 사용에서 브로커와 주키퍼를 동일 서버에 함께 설치할지 여부는 시스템 요구 사항에 따라 달라진다. 필요한 서버 수를 줄이기 위해 동일 서버에 함께 설치하거나, 하둡과 같은 다른 미들웨어와 공유하기 위해 별도로 구축하는 등 각각의 사정을 고려하여 결정한다.<br>
카프카 클라이언트 서버에는 카프카 클러스터 조작에 필요한 도구를 설치한다. 프로듀서와 컨수머는 실행하는 애플리케이션에 따라 카프카와 라이브러리를 필요로 하는 경우도 있어 미리 설치하면 좋다. 이 장에서는 카프카 클러스터의 동작을 확인하고 프로듀서, 컨수머 각각의 서버에서 카프카 도구를 이용하기 위해 설치한다.

### 3.2.3 카프카 패키지와 배포판

카프카는 개발사인 아파치소프트웨어재단이 배포하고 있는 커뮤니티 버전의 패키지 외에도 미국 컨플루언트, 미국 클라우데라, 미국 호튼웍스가 배포하고 있는 배포판에 포함된 패키지로도 사용할 수 있다.
<br>
패키지/배포판|개발사|URL
---|---|---
커뮤니티 버전|아파치소프트웨어재단|http://kafka.apache.org
Confluent Platform|컨플루언트|http://www.confluent.io
Cloudera's Distribution of Apache Kafka(CDK)|클라우데라|http://www.cloudera.com
Hortonworks Data Platform(HDP)|호튼웍스|http://hortonworks.com

<br>
각 사의 배포판은 커뮤니티 에디션이 출시된 이후에 자체적으로 개발한 도구를 추가하거나, 배포판에 포함된 다른 도구와의 연계를 위한 커스터마이징을 하고 있다. 따라서 카프카를 스파크같은 다른 미들웨어와 함께 사용할 경우 이용 상황에 따라 선택해야 한다. 배폴판별로 출시 주기의 차이로 인해 커뮤니티 에디션 최신 버전이 포함될 때까지 시간이 경과할 수도 있다.<br>
이 책에서는 미국 컨플루언트가 배포하고 있는 컨플루언트 플랫폼의 OSS 버전으로 카프카 클러스터를 구축한다.

## 3.3 카프카 구축

여기서는 여러 대의 서버에서 카프카 클러스터를 구축하는 경우와 1대의 서버로 구축하는 경우를 구분하여 소개한다. 여러 대의 서버로 구축할 경우 절 제목에서 '공통'과 '여러 대의 경우'를, 1대만으로 구축하는 경우는 '공통'만을 실행하면 된다.

### 3.3.1 OS설치 - 공통

카프카와 컨플루언트 플랫폼은 리눅스와 맥OS에서 동작한다. 윈도우에서는 지원되지 않는다. 이 책에서는 CentOS 7의 64비트 버전을 사용한다. 설치 방법은 환경에 따라 다르기 때문에 이 책에서는 설명을 생략한다.<br>
이 책에서는 일반 사용자가 사용하는 것을 가정하므로 OS 설치 후에 필요에 따라 사용자를 설정하면 된다. 일부 루트 권한이 필요한 작업에서는 sudo 명렬으로 실행하기 때문에 적절한 사용자에게 권한을 부여해야 한다. 또한 여러 대의 서버에서 환경을 구축하려면 서버 간 서로의 호스트명으로 이름을 확인하여 통신할 수 있도록 /etc/hosts 에 네트워크 설정도 완료한다.

> 나의 경우 Windows 10의 개발환경이며, 추후 확장성을 고려하여 WSL을 사용하지 않고 오라클 VirtualBox를 통해 CentOS 7을 설치하였다.

### 3.3.2 JDK 설치 - 공통

카프카 실행에는 JDK가 필요하므로 각 서버에 설치한다. JDK에는 오라클이 제공하는 JDK와 오픈소스인 OpenJDK 등이 있다.<br>
자바와 JDK는 여러 버전이 존재하는데 이 책을 쓰는 시점에서 카프카가 지원하고 있는 것은 버전8뿐이다. 이 책을 쓰는 시점에서 최신 버전인 Oracle JDK의 Java SE 8u181을 이용한다. 다음 웹사이트에서 Orcacle JDK를 다운로드한다.<br><br>
http://www.oracle.com/technetwork/java/javase/downloads/index.html
<br><br>
해당 버전의 Linux x64 rpm 패키지를 선택하여 다운로드한다. 여기에서는 다운로드한 파일을 각 서버의 /tmp에 배치한다. 다운로드한 rpm 패키지를 다음 명령어로 설치한다.

```bash
$ sudo rpm -ivh /tmp/jdk-8u221-linux-x64.rpm
```

> 나의 경우 OpenJDK를 통해 자바를 설치하였다. 또한 구글신의 힘을 빌려 네트워크나 기타 설정을 마쳤다.
> <br>
>
> ```bash
> $ sudo yum install java-1.8.0-openjdk-devel.x86_64
> ```

(이후 환경변수 설정 생략)<br>

### 3.3.3 컨플루언트 플랫폼 리포지터리 등록 - 공통

컨플루언트 플랫폼의 Yum 리포지터리를 등록한다. 집필 시점 최신 버전인 5.0.0을 사용한다. 버전에 다라 일부 절차가 다를 수 있다.

먼저 Yum 리포지터리 사용을 위해 컨플루언트가 제공하는 공개키를 등록한다.

```bash
$ sudo rpm --import https://packages.confluent.io/rpm/5.0/archive.key
```

다음으로 /etc/yum.repos.d/confluent.repo라는 파일을 만들고 다음과 같이 작성한다.<br>
(파일 작성은 ROOT 권한으로 해야 한다.)

```shell
[Confluent.dist]
name=Confluent repository (dist)
baseurl=https://packages.confluent.io/rpm/5.0/7
gpgcheck=1
gpgcheck=https://packages.confluent.io/rpm/5.0/archive.key
enabled=1

[Confluent]
name=Confluent repository
baseurl=https://packages.confluent.io/rpm/5.0
gpgcheck=1
gpgcheck=https://packages.confluent.io/rpm/5.0/archive.key
enabled=1
```

여기까지 Yum 리포지터리 등록을 완료했다. 마지막으로 기존의 캐시를 삭제한다.

```bash
$ yum clean all
```

Yum으로 사용 가능한 패키지 목록에 컨플루언트 플랫폼이 포함되어 있는지 여부를 확인한다.

```bash
$ yum list | grep confluent
  (생략)
confluent-kafka-2.11.noarch    2.0.0-1    Confluent
  (생략)
```

### 3.3.4 카프카 설치 - 공통

등록한 컨플루언트 플랫폼 리포지터리에서 카프카를 실행하기 위해 필요한 패키지를 설치한다. 다음 명령으로 confluent-platform-oss-2.11이라는 패키지를 설치한다.

```bash
$ sudo yum install confluent-platform-oss-2.11
```

컨플루언트 플랫폼은 잘게 구분된 여러 패키지가 존재하는데, 이 confluent-platform-oss-2.11은 그 중 컨플루언트 플랫폼 OSS 버전의 패키지를 설치하기 위한 것이다. 이것을 설치함으로써 필요한 패키지가 모두 설치된다. 이 패키지에 대한 자세한내용은 컨플루언트 플랫폼 문서를 참고하자.

### 3.3.5 브로커의 데이터 디렉터리 설정 - 공통

카프카 패키지를 설치한 다음 브로커의 데이터 디렉터리를 설정한다.

/etc/kafka/server.properties 를 열고 다음과 같이 수정한다.

```shell
(생략)
log.dirs=/var/lib/kafka/data
(생략)
```

여기에서 설정한 log.dirs는 브로커에서 사용하는 데이터 디렉터리를 설정하는 항목이다. 컨플루언트 플랫폼의 기본값은 /var/lib/kafka로 설정되어 있는데 Oracle JDK를 사용하는 경우 이를 변경해야 한다. 여기에서는 /var/lib/kafka/data를 이용한다.

여기에서는 하나의 디렉터리를 지정하고 있지만 log.dirs에는 여러 디렉터리를 지정할 수도 있다. 여러 디렉터리를 지정하려면 디렉터리의 경로를 쉼표(,)로 이어서 작성한다.

다음으로 브로커가 사용하는 데이터 디렉터리를 만든다. 여기에서는 /var/lib/kafka/data를 이용하므로 이 디렉터리를 만든다. 나중에 언급할 컨플루언트 플랫폼에 포함되어 있는 스크립트로 브로커를 시작하는 절차에서는 브로커를 시작하는 사용자가 cp-kafka이므로 디렉터리의 소유자도 여기에 맞게 변경한다.

```bash
$ sudo mkdir /var/lib/kafka/data
$ sudo chown cp-kafka:confluent /var/lib/kafka/data
```

이것으로 브로커의 데이터 디렉터리 작성과 설정이 완료됐다. 서버 1대만으로 카프카 환경을 구축하는 경우 지금까지 과정으로 모든 작업이 완료된다.

### 3.3.6 여러대 에서 동작하기 위한 설정 - 여러 대의 경우

컨플루언트 플랫폼의 패키지에는 설치 후 최소한의 설정이 포함되어 있다. 서버 1대로 동작시킨다면 앞 절의 과정만으로 실행할 수 있으나 주키퍼와 카프카 클러스터를 여러 대의 클러스터로 동작시키기 위해서는 몇 가지 추가 설정이 필요하다.

우선 주키퍼 추가 설정을 한다. 주키퍼 설정 파일인 /etc/kafka/zookeeper.properties에 다음의 내용을 추가한다.

```shell
initLimit=10
syncLimit=5
server.1=kafka-broker01:2888:3888
server.2=kafka-broker02:2888:3888
server.3=kafka-broker03:2888:3888
```

initLimit와 syncLimit는 주키퍼 클러스터의 초기 접속 및 동기 타임아웃 값을 설정하고 있다. 이 타임아수은 tickTime이라는 파라미터를 단위로 계산된다. tickTime의 기본값은 3000밀리초. 이때 initLimit=10은 30초(30,000ms), syncLimit=5는 15초(15,000ms)를 의미한다.

server.1=kafka-broker01:2888:3888 부분은 클러스터를 구성하는 서버군의 정보를 담고 있다. 이곳은 다음의 형식으로 클러스터를 구성하는 모든 서버만큼 작성한다.

```shell
server.<myid>=<서버 호스트명>:<서버 통신용 포트1>:<서버 통신용 포트2>
```

myid란 주키퍼 클러스터 내에서 각 서버에 고유하게 부여되는 서버 번호다. 여기에서는 kafka-broker01이 1, kafka-broker02가 2, kafka-broker03이 3이 된다.

각 서버에 할당된 myid는 각 서버의 /var/lib/zookeeper/myid에 작성해야 한다. myid는 서버마다 다르기 때문에 실행하는 명령도 서버마다 다르다. 컨플루언트 플랫폼이 제공하는 실행 방법으로는 주키퍼와 카프카의 실행자가 cp-kafka이므로 파일도 이 사용자의 소유가 되도록 작성한다.

kafka-borker01(myid=1)에서는 다음 명령을 실행한다.

```bash
(kafka-broker01)$ echo 1 | sudo -u cp-kafka tee -a /var/lib/zookeeper/myid
```

kafka-borker02(myid=2)에서는 다음 명령을 실행한다.

```bash
(kafka-broker02)$ echo 2 | sudo -u cp-kafka tee -a /var/lib/zookeeper/myid
```

kafka-borker03(myid=3)에서는 다음 명령을 실행한다.

```bash
(kafka-broker03)$ echo 3 | sudo -u cp-kafka tee -a /var/lib/zookeeper/myid
```

다음으로 브로커를 추가 설정한다. /etc/kafka/server.properties를 열고 다음과 같이 수정한다.

```shell
(생략)
# 이미 작성되어 있는 것을 수정
broker.id=<서버별로 정한 Broker ID>
# 새롭게 작성
broker.id.generation.enable=false
(생략)
# 이미 작성되어 있는 것을 수정
zookeeper.connect=kafka-broker01:2181,kafka-broker02:2181,kafka-broker03:2181
(생략)
```

앞서 언급했듯이 log.dirs 설정을 변경한 상태인데, 여기에서는 그 파일에 추가로 여러 대의 서버에서 실행하기 위한 설정을 한다.

broker.id는 브로커 ID를 설정하기 위한 항목이다. 브로커도 주키퍼의 myid와 마찬가지로 브로커마다 고유 ID를 부여해야한다. 이 브로커마다 부여되는 고유 ID를 브로커 ID라고 부르며 정수값으로 설정한다. 이 브로커 ID는 주키퍼의 myid와 동일할 필요는 없지만 여기에서는 kafka-broker01을 1,kafka-broker02을 2, kafka-broker03을 3으로 설정한다.

브로커 ID는 사용자가 명시적으로 지정하지 않고 카프카에서 자동으로 부여할 수도 있다. 브로커 ID를 자동으로 부여할 경우 broker.id를 설정하지 않고 broker.id.generation.enable에 true를 설정한다. 위의 설정에서는 broker.id.generation.enable을 false로 설정하여 이 기능을 무효로 하고 수동으로 브로커 ID를 설정하고 있다.

zookeeper.connect는 브로커가 주키퍼에 접속할 때의 접속 정보를 설정한다. <주키퍼 호스트명>:<접속에 사용할 포트 번호>의 형태로 작성한다. 접속하는 서버가 여럿일 경우 앞서 언급한 바와 같이 쉽표로 연결하여 지저한다.

이것으로 여러 대의 서버를 이용할 때 카프카의 설치 및 설정을 완료했다.

## 3.4 카프카 실행과 동작 확인

구축한 카프카 클러스터를 실행한다. 주키퍼와 브로커 중에 주키퍼를 먼저 실행한 후에 브로커를 실행해야 한다.

우선 주키퍼를 실행하기 위해 다음 명령어를 입력한다. 여러 대의 서버에서 카프카 클러스터를 구축하고 있는 경우 주키퍼가 설치된 모든 머신에서 실행한다. 여러 대의 서버에서 명령을 실행할 때 주키퍼 서버 간 명령 실행 순서는 상관없다.

```bash
$ sudo systemctl start confluent-zookeeper
```

다음으로 브로커를 실행하기 위해 다음 명령을 입력한다. 여러 대의 서버에서 카프카 클러스터를 구축하고 있는 경우는 브로커가 설치된 모든 머신에서 실행한다. 마찬가지로 브로커 간 명령 실행 순서는 상관없다.

```bash
$ sudo systemctl start confluent-kafka
```

주키퍼의 로그는 /var/log/kafka/zookeeper.out, 브로커의 로그는 /var/log/kafka/server.log에 출력된다. 제대로 실행되지 않으면 이 로그를 확인하여 원인을 파악할 수 있다.

## 3.4.2 카프카 클러스터 동작 확인

마지막으로 실행시킨 카프카 클러스트가 동작하는지 확인한다.

여기에서는 카프카에 들어 있는 도구 Kafka Console Producer와 Kafka Console Consumer를 이용하여 실제로 메시지를 전송하고, 카프카 클러스터가 제대로 메시지를 송수신하는지 여부를 확인할 것이다.

우선 동작 확인을 위한 메시지를 송수신하기 위한 토픽을 작성한다. 카프카 클라이언트에서 다음의 명령을 실행하자. 여기에서는 first-test라는 이름으로 토픽을 작성한다. 주키퍼의 접속 정보가 다르기 때문에 구성한 서버 수에 따라 명령이 다를 수 있다. 다음은 서버 3대로 클러스터를 작성한 경우의 명령이다.<br><br>

_서버 3대로 클러스터를 구축한 경우_

```bash
(kafka-client)$ kafka-topics --zookeeper kafka-broker01:2181,kafka-broker02:2181,kafka-broker03:2181 \
> --create --topic first-test --partitions 3 --replication-factor 3

Created topic "first-test".
```

> 나의 경우 서버 1대로 진행했기 때문에 아래와 같이 명령어를 작성했다.
>
> ```bash
> (kafka-client)$ kafka-topics --zookeeper localhost:2181 \
> > --create --topic first-test --partitions 3 --replication-factor 1
>
> Created topic "first-test".
> ```

명령의 의미를 살펴보자.

**--zookeeper**

카프카 클러스터를 관리하고 있는 주키퍼로의 정속 정보를 지정한다. 카프카의 설정 파일과 마찬가지로 <호스트명>:<접속 포트 번호> 형태로 지정하고 여러 개인 경우는 쉼표로 연결하여 지정한다.

서버 1대로 카프카 환경을 구축한 경우에는 다음과 같이 지정한다.

```bash
--zookeeper kafka-server:2181
```

**--create**

토픽을 작성한다. --create 외에 토픽의 목록을 확인하는 --list, 토픽을 삭제하는 --delete 등이 있다.

**--topic**

작성하는 토픽 이름을 지정한다. 여기에서는 토픽 이름으로 first-test를 지정하고 있다. 토픽 이름에는 밑줄(\_)과 마침표(.)를 사용하지 않는다.

**--partitions**

작성하는 토픽의 파티션 수를 지정한다. 여기에서는 파티션 수를 3으로 하고 있다. 파티션 수를 결정하는 것은 11.5절에서 설명한다.

**--replication-factor**

작성하는 토픽의 레플리카의 수 (Replication-Factor)를 지정한다. 3대의 서버를 사용하는 경우 3을 지정한다.

Replication Factor는 카프카의 클러스트의 브로커 수 이하여야 한다. 예를 들어 서버가 1대인 카프카 클러스터에서 Replication-Factor를 3으로 지정하면 오류가 발생한다. 따라서 서버 1대로 카프카 환경을 구축한 경우는 다음과 같이 지정한다.

```bash
--replication-factor 1
```

토픽을 만든 다음 토픽이 제대로 작성되었는지 확인하자. 다음 명렬으로 지저한 토픽의 정보를 확인할 수 있다. 이것도 토픽을 만들 때와 마찬가지로 주키퍼 접속 정보가 다를 경우 명령도 달라진다.

다음 명령은 여러 대의 서버에서 클러스트를 구축한 경우의 예다.

```bash
(kafka-client)$ kafka-topics --zookeeper kafka-broker01:2181,kafka-broker02:2181,kafka-broker03:2181 \
> --describe --topic first-test

Topic:first-test PartitionCount:3 ReplicationFactor: 3 Configs:
Topic: first-test Partition: 0 Leader: 1 Replicas: 1,2,3 Isr: 1,2,3
Topic: first-test Partition: 1 Leader: 2 Replicas: 2,3,1 Isr: 2,3,1
Topic: first-test Partition: 0 Leader: 1 Replicas: 3,1,2 Isr: 3,1,2
```

```bash
# 서버 1대인 경우
(kafka-client)$ kafka-topics --zookeeper localhost:2181 --describe --topic first-test
```

토픽을 만들 때 지정한 옵션 --create가 --describe로 바뀌어 있다. 이것은 지정된 토픽에 대한 상세 정보를 표시하는 옵션이다.

작성할 때 토픽이 존재하고 지정된 파티션 수와 Replication-Factor가 있으면 토픽이 제대로 작성된 것이다.

표시된 출력의 각 항목의 의미는 다음과 같다.

**Topic, PartitionCount, ReplicationFactor**

지정한 토픽의 이름, 파티션 수, Replication-Factor가 표시되어 있다.

**Leader**

각 파티션의 현재 Leader Replica가 어떤 브로커에 존재하고 있는지 표시한다. 여기에 표시되는 번호는 각 브로커에 설정한 브로커 ID다.

**Replicas**

각 파티션의 레플리카(복제본)를 보유하고 있는 브로커의 목록이 표시된다.

**Isr**

In-Sync Replicas의 약자로 복제본 중 Leader Replica와 올바르게 동기가 실행된 복제본을 보유하고 잇는 브로커 목록이 표시된다. 장애가 발생하고 있는 브로커를 보유하고 있거나, 특정 이유로 Leader Replica의 동기화가 실행되지 않은 레플리카는 In-Sync Replicas에 포함되지 않는다. Leader Replica 자신은 In-Sync Replicas에 포함된다.
<br><br><br>
다음으로 프로듀서 서버에서 메시지를 보내기 위한 Kafka Console Producer를 실행한다.

Kafka Console Producer는 카프카에 들어 있는 도구로 콘솔에 입력한 데이터를 메시지로 카프카에 보낸다.

다음 명령으로 Kafka Console Producer를 실행한다.

```bash
(producer-client)$ kafka-console-producer --broker-list kafka-broker01:2181,kafka-broker02:2181,kafka-broker03:2181 \
> --topic first-test
>
```

```bash
# 서버 1대
(producer-client)$ kafka-console-producer --broker-list localhost:9092 --topic first-test
>
```

지정한 옵션의 의미는 다음과 같다.

**--broker-list**

메시지를 보내는 카프카 클러스터의 브로커 호스트명과 포트 번호를 지저한다. <호스트명>:<포트 번호>의 형태로 지저하며 여러 개가 있을 경우는 쉼표로 연결한다. 카프카가 통신에 사용하는 포트의 기본값은 9092다. 서버 1대의 호나경에선느 적절하게 수행해야 한다.

**--topic**

메시지 송신처가 되는 토픽을 지정한다.

<br><br>

그리고 컨수머 서버에서 메시지를 받기 위한 Kafka Console Consumer를 실행한다. Kafka Console Consumer는 Kafka Console Producer와 마찬가지로 카프카에 속한 도구로 받은 메시지를 콘솔에 표시한다.

컨수머 서버에서 다음 명령을 실행하여 Kafka Console Consumer를 실행한다. 1대의 서버로 구축한 경우는 Kafka Console Producer를 앞서 실행한 콘솔이 아닌 별도의 콘솔을 열어 실행한다.

```bash
(consumer-client)$ kafka-console-consumer --bootstrap-server kafka-broker01:2181,kafka-broker02:2181,kafka-broker03:2181 \
> --topic first-test

# 서버 1대
(consumer-client)$ kafka-console-consumer --bootstrap-server localhost:9092 \
> --topic first-test
```

지정한 옵션의 의미는 다음과 같다.

**--bootstrap-server**

메시지를 수신하는 카프카 클러스터의 브로커 호스트명과 포트 번호를 지정한다. 지정 방법은 Kafka Console Producer의 --broker-list와 동일하다.

**--topic**

메시지를 수신하는 토픽을 지정한다.
<br><br>

그럼 실제로 메시지를 보내서 동작을 확인해보자. 실행한 Kafka Console Producer로 송신 메시지를 입력한다.

```bash
(producer-client)$ kafka-console-producer --broker-list kafka-broker01:2181,kafka-broker02:2181,kafka-broker03:2181 \
> --topic first-test
> Hello Kafka!
# 문자열 입력후 엔터
```

송신 메시지가 실행한 Kafka Console Comsumer에 표시되는지 확인한다.

```bash
(consumer-client)$ kafka-console-consumer --bootstrap-server kafka-broker01:2181,kafka-broker02:2181,kafka-broker03:2181 \
> --topic first-test
Hello Kafka!
```

카프카 클러스터를 경유하여 메시지를 서로 교환하는 것을 확인할 수 있엇다. 여러 메시지를 Kafka Console Producer에서 보내더라도 동일하게 Kafka Console Consumer에서 받아서 표시할 것이다. 확인을 했으면 Kafka Console Producer, Kafka Console Consumer 모두 [Ctrl] + [C] 로 종료한다.

### 3.4.3 카프카 클러스터 종료

동작이 확인되면 마지막으로 실행한 카프카 클러스터를 종료한다. 종료할 때는 실행한 것과 반대로 브로커부터 종료하고, 그 다음 주키퍼를 종료한다.

우선 브로커를 종료하자. 종료는 실행과 마찬가지로 systemctl 명령으로 실행한다. 여러 대의 서버에서 브로커를 구축하는 경우 브로커 전체에서 실행한다. 여러 대의 브로커에서 실행할 때 브로커끼리 명령 실행 순서는 상관없다.

```bash
$ sudo systemctl stop confluent-kafka
```

다음으로 주키퍼를 종료한다. 마찬가지로 systemctl을 이용하여 다음 명령을 실행한다. 여러대의 서버에서 주키퍼를 구축한 경우 모든 주키퍼에서 실행한다. 이것도 여러 대의 서버에서 실행 시 서버 간 명령 실행 순서는 상관없다.

```bash
$ sudo sytemctl stop confluent-zookeeper
```

실행했던 카프카 클러스터가 종료됐다. 다시 클러스터를 실행할 때는 이 장에서 설명한 카프카 클러스터 실행 순서를 반복하면 된다. 이 책의 예제에서는 카프카 클러스터가 이미 실행하고 있는 것을 전제로 하고 있기 때문에 예제 실습에 앞서 실행하기 바란다.

## 3.5 정리

컨플루언트 플랫폼을 이용한 카프카 설치 방법과 동작 확인 방법을 소개했다. 다음 장에서는 이 환경에서 카프카의 사용법을 소개한다.

~~처음부터 모든 환경을 준비하느라, 카프카 설치와 동작 확인을 완료하는데 10시간이 조금 넘게 걸렸다.~~
