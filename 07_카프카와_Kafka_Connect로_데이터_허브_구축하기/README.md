# 7장 카프카와 Kafka Connect로 데이터 허브 구축하기

## 7.1 이 장의 내용

5장에서 카프카의 실제 사례를 소개했다. 여기서는 카프카의 전형적인 사례 중 하나인 데이터 허브 아키텍처를 응용한 사례를 살펴본다. 5장에선 소개한 대로 데이터 허브 아키텍처의 기본 개념은 데이터를 여러 시스템에 전달시키는 것이다. 카프카와 Kafka Connect를 사용하여 데이터 허브를 구축하고 체험하면서 그 개념을 이해하자.

## 7.2 Kafka Connect란

Kafka Connect는 아파치 카프카에 포함된 프레임워크로 카프카와 다른 시스템과의 데이터 연계에 사용한다. 카프카에 데이터를 넣거나, 카프카에서 데이터를 추출하는 과정을 간단히 하기 위해 만들어졌다. 6장에서 설명한 카프카의 데이터 파이프라인에서 포변 Kafka Connect는 프로듀서와 컨수머 양쪽 모두를 구성할 수 있다. 카프카는 특성상 다양한 시스템과 연계하기 때문에 Kafka Connect에서는 다른 시스템과 연결하는 부분은 커넥터라는 플러그인으로 구현하는 방식을 취하고 있으며, 'Kafka Connect 본체 + 플러그인' 구성으로 동작한다. Kafka Connect에서는 카프카에 데이터를 넣는 프로듀서 쪽 커넥터를 소스라고 부르고, 카프카에 데이터를 출력하는 컨수머 쪽의 커넥터를 싱크라고 한다.

이미 많은 플러그인이 공개되어 있어 연결할 곳에 맞는 커넥터가 있으면 직접 코딩하지 않더라도 데이터 입출력을 실행할 수 있다. 컨플루언트 공식 사이트에는 공개된 커넥터 목록을 볼 수 있다. 이 사이트에서 컨플루언트 플랫폼에 포함되어 있는 커넥터나 컨플루언트에 인정받은 커넷터, 기타 공개된 커넥터까지 여러 종류의 다양한 커넥터를 찾을 수 있다.

이러한 커넥터를 사용하면 다양한 시스템을 연결할 수 있다. 단, 주의 사항이 있다. 모든 플러그인이 소스와 싱크를 구현하고 있지는 않다는 점이다. 소스나 싱크 둘 중 하나만 제공하는 플러그인도 많기 때문에 확인이 필요하다. 또한 공개 플러그인이 반드시 카프카 커뮤니티와 컨플루언트에 의해 개발된 것이 아니기 때문에 품질이나 문서의 총실도에 차이가 있을 수 있는 점은 유의해야 한다.

사용하려는 커넥터가 없을 경우 직접 커넥터를 구현할 수 있다. 이 책에서는 다루지 않지만 컨플루언트에서 제공하는 커넥터 개발 가이드를 참고하여 만들 수 있으므로 흥미 있는 사람은 도전해보길 바란다.

Kafka Connect는 데이터를 카프카에 입출력하기 위한 확장 가능한 아키텍처를 갖고 있으며 여러 서버로 클러스터를 구성할 수 있다. Kafka Connect는 브로커와 동일한 서버에 동작할 수 있어 카프카 클러스트와 Kafka Connect 클러스터를 함께 구성하는 것도 가능하다. Kafka Connect 클러스터에서 소스나 싱크의 플러그인으로 데이터를 입출력할 때는 논리적인 작업을 실행한다. 이 논리적인 작업을 **커넥터 인스턴스**(또는 단순히 커넥터)라고 부른다. 커넥터 인스턴스는 여러 태스크를 실행하는데, 이 태스크는 클러스터 상의 여러 서버에서 잘 분담해서 실행되며 실제적인 데이터의 복사를 실시한다. 이처럼 Kafka Connect는 하나의 커넥터 인스턴스가 여러 태스크를 가질 수 있는 구조라서 확장 가능한 데이터 입출력이 가능하다.

## 7.3 데이터 허브 아키텍처 응용 사례

여기서부터는 Kafka Connect를 사용하여 데이터 허브 아키텍처를 구현하는 방식을 살펴본다. 데이터 허브는 여러 시스템 사이에서 데이터를 전달하기 위한 허브다. 따라서 데이터 허브를 사용할 때는 데이터를 연계하려는 여러 시스템도 함께 고려할 필요가 있다.

### 7.3.1 데이터 허브 아키텍처가 유효한 시스템

데이터 허브 아키텍처가 유효한 시스템은 무엇인지 어떤 대규모 소매점을 예로 살펴보자. 실제 매장이 전국에 있는 대형 소매점 A가 있다. A 회사는 점포의 재고 관리에 배송 관리, 포인트카드의 회원정보 관리, 판매 관리, POS 등 사업을 운영하기 위한 많은 시스템이 있다. 그뿐 아니라 인터넷 쇼핑몰도 판매 채널이 있으며, 직접 전사상거래 사이트를 운영하는 한편 대형 온라인 쇼핑몰에도 인터넷 매장을 열고 있다.

이 예에서 A 회사는 업무를 하기 위해 많은 시스템을 사용하고 있다. 이것은 A 회사에만 해당하는 이야기가 아니라 대기업과 중견기업이라면 모두 비슷한 IT 전산화를 추진하고 있을 것이다. 아마도 여러분 주변도 이미 이와 비슷한 상황이 아닐까 싶다.

A 회사에서의 데이터 전달에 대해 생각해보자. 만약 A 회사에서 다음달 출시되는 주력 상품의 판매 예측을 세워야 한다면 대응할 수 있을까? A 회사에는 실제 매장의 판매 관리 시스템이 있기 때문에 과거의 데이터도 있을 것이고, 전자상거래 사이트 또한 매출 데이터를 가지고 있을 것이다. 과거의 매출 데이터를 잘 조합해 포인트카드에서 소비자를 연관지어 분석할 수 있으면 판매예측이 가능할 것이다. 다른 예도 생각해보자. 만약 A 회사에서 주문 처리를 효율화하기 위해 자동주문을 하려고 하면 가능할까? A 회사는 과거 매출데이터와 판매예측 데이터도 있고 재고 데이터도 있다. 이것을 조합하면 자동 주문도 할 수 있을 것이다.

하지만 이것이 가능하려면 시스템 간 데이터를 잘 전달할 수 있다는 경우에 한해서다. 만약 실제 매장의 판매 관리 시스템의 데이터 형식이 각 점포와 차이가 있어 제대로 통합할 수 없다면 판매 예측은 어렵다. 만약 재고 관리 시스템이 자동 주문 시스템에 데이터를 전달하는 인터페이스가 없다면 그 이유만으로 자동 주문은 포기해야 한다. 장래에 실제 매장과 연계한 재고 정보를 전자상거래 사이트에 표시하려고 해도 이 또한 포기해야 한다. 바로 이것이 5장에서 설명한 사일로화의 폐해이다. 이렇듯 시스템이 많아지면 모든 시스템을 연결하는 것이 현실적으로 어려워진다.

새로운 자동 주문 시스템을 만들었을 때 데이터를 출력하는 쪽인 재고 관리 시스템에도 수정을 해야하는 상황이라면 연관 시스템이 다운될지도 모른다. 이러한 경우 데이터 허브 아키텍처를 사용한다면 시스템 다운은 일어나지 않는다. 왜냐하면 데이터 허브 아키텍처에서는 재고 관리 시스템은 데이터 허브에서만 데이터를 입출력하고, 판매 관리 시스템과 자동 주문 시스템도 이와 마찬가지로 데이터 허브에만 입출력을 실행하기 때문이다. 이렇게 하면 각각의 시스템 설계가 단순화되고 시스템끼리 자유롭게 데이터를 전달할 수 있다.

### 7.3.2 데이터 허브 도입 후의 모습

A 회사에 데이터 허브 아키텍처가 도입된다면 어떻게 될까? 이후에서는 데이터 절달의 몇 가지 형태를 고려해보기로 한다. 데이터 전달은 다음 3가지를 구현하는 것을 목표로 한다.

1. 전자상거래 사이트에 실제 매장의 재고 정보를 표시하기<br>
   재고 관리 시스템의 데이터를 전자상거래 사이트에 전달한다.
2. 매월 판매 예측하기<br>
   실제 매장의 POS 시스템 데이터와 전사상거래 사이트의 매출 데이터를 판매 예측 시스템에 전달한다.
3. 자동 주문하기<br>
   판매 예측 데이터와 재고 관리 시스템의 데이터를 자동 주문 시스템에 전달한다.

하나하나가 매우 단순하기 때문에 대략적으로 상상이 가능할 것이다. 데이터 허브를 이용하는 방식에서는 각 시스템의 데이터를 데이터를 개별적으로 연계하는 것이 아니라 그 사이에 데이터 허브를 끼워 넣는 식으로 구현한다. 이렇게 함으로써 3가지 기능을 간단히 구현할 수 있다.

이제 각각을 개별적으로 살펴보자. 이 장에서는 3개의 기능중 1.(재고 정보 표시)과 2.(월 판매 예측)를 설명한다. 세 번째 자동 주문은 1.과 2.에서 배운 지식을 응용하여 여러분이 구현해보기를 바란다. Kafka Connect로 시스템끼리 연결하는 것은 대부분 데이터 저장소 사이에서 연계하게 된다. 따라서 3개 기능을 실현하는 데이터 허브의 데이터 저장소를 정리하면 다음과 같다.(그림 생략)

1.에서는 재고 관리 시스템과 전자상거래 사이트를 연결한다. 처음에 기본을 확실히 익히는 것이 중요하다. Kafka Connect에서는 Hello World 같은 것은 없지만 기본을 이해하는 데 좋은 로컬 파일 연계용 커넥터가 있다. 이 커넥터는 Kafka Connect 의 로컬 파일 업데이트를 감시하여 카프카에 데이터를 넣거나, 카프카의 데이터를 Kafka Connect의 로컬 파일에 기록하는 커넥터다. 로컬 파일을 사용하기 때문에 Kafka Connect의 클러스터만 있으면 동작하므로 간편하게 사용할 수 있다. 재고 관리 시스템은 순차적으로 새로운 데이터를 파일에 추가해가며 전자상거래 사이트는 업데이트 데이터를 파일 형식으로 받는 식으로 시스템이 완성된다. 이 예제로 Kafka Connect의 기본을 파악하고 동작을 이해하는 것을 목표로 한다.

2.에서는 POS, 전자상거래 사이트, 판매 예측 시스템을 연결한다. 기본을 마쳤으니 실제로 있을 법한 경우를 체험해보자. 많은 시스템의 데이터 저장소 계층은 RDBMS인 경우가 많다. 여기서는 전사상거래 사이트나 POS가 RDBMS에 데이터를 저장하고 있다고 가정한다. 여러 시스템을 도입하고 있는 회사는 시스템마다 사용하는 RDBMS가 각각 다른 경우가 많다. 이번에도 전자상거래 사이트와 POS를 서로 다른 제품으로 시도해보자. 여기에서는 PostgreSQL과 MariaDB를 사용한다. 판매 예측 시스템은 최근 흔히 볼 수 있는 구성으로 클라우드 서비스로 구현한다고 가정해보자. 이 경우에는 클라우드 서비스의 스토리지에 데이터를 두는 것이 좋을 것이다. 여기에서는 AWS의 S3으로 가정한다. 이 예제를 통해 실제 Kafka Connect의 사용법을 파악할 수 있을 것이다.

3.은 독자가 생각할 차례다. 컨플루언트의 공식 사이트에 공개된 Kafka Connect 목록에서 생각하고 있는 시스템에서 사용할 만한 커넥터를 찾아 구현해보기를 바란다.

참고로 데이터 허브 아키텍처는 Kafka Connect를 사용하지 않고도 프로듀서나 컨수머를 개별적으로 구현할 수도 있다. 실제로 이번과 같은 사례에서는 어떠한 방식으로든지 구현할 수 있을 것이다. Kafka Connect를 사용하는 경우 이미 커넥터가 공개되어 있는 연결 시스템이라면 코딩 없이 비교적 간단하게 시스템끼리 연계할 수 있다는 점이 큰 장점이다. 단, 실제 구현의 난이도/성능 요구 사항/시맨틱스(어느정도의 메시지 손실과 중복을 허용할지) 등을 종합적으로 판단하여 결정한다.

그럼 Kafka Connect를 사용한 데이터 허브를 구축해보자.

## 7.4 환경 구성

데이터 허브를 구현하기 위해서는 준비가 필요하다. 당연한 이야기지만 데이터 허브는 데이터 허브 단독으로 존재해봐야 아무런 의미가 없기 때문에 접속할 시스템이 존재해야 한다. 필요한 준비 환경을 나열하고 전체 그림을 그려보자.

우선적으로 필요한 것은 카프카 클러스터와 Kafka Connect다. 카프카 클러스터는 3장에서 구축한 것이 있으면 그대로 사용할 수 있다. 만약에 없으면 3장의 절차에 따라 클러스터를 구축하면 된다. Kafka Connect는 카프카 클러스터의 브로커와 함께 설치해 실행한다.

다음으로 필요한 것은 연계 시스템이다. 1.에서는 재고관리 시스템과 전자상거래 사이트가 등장했다. 2.에서는 POS, 전자상거래 사이트, 판매 예측 시스템이 등장했다. 이러한 시스템을 준비해야 한다. 그렇다고 해도 이 예제를 위해 일부러 웹사이트나 애플리케이션, 배치 등을 만드는 일은 상당히 힘든 작업이다. 데이터 허브가 동작하는 방식을 체험하려면 연계에 필요한 데이터 저장소만 준비하면 된다. 데이터는 수작업을 통한 입력으로도 충분하다.

준비가 필요한 카프카 클러스터와 연계 시스템용 데이터 저장소를 아래와 같이 정리했다.

| 준비물              | 구성제품       | 서버명               | 비고                                   |
| ------------------- | -------------- | -------------------- | -------------------------------------- |
| 카프카 클러스터     | Kafka          | kafka-broker01,02,03 | 3대 구성                               |
| 1.재고관리 시스템   | 로컬파일시스템 | kafka-broker01       | 카프카 클러스터의 로컬 파일시스템 사용 |
| 2.전자상거래 사이트 | 로컬파일시스템 | kafka-broker01       | 카프카 클러스터의 로컬 파일시스템 사용 |
| 2.전자상거래 사이트 | PostgreSQL     | ec-data-server       |
| 2.POS               | MariaDB        | pos-data-server      |
| 2.판매예측 시스템   | S3             | 없음                 | AWS 사용                               |

준비가 필요한 것은 그 외에 더 있다. 이들을 연결하는 커넥터도 검토한다. Kafka Connect는 연결 시스템에 맞춰 커넥터 플러그인이 필요하다. 사용할 커넥터 플러그인을 선택해두자.

1.에서 파일 연계를 위해 필요한 커넥터는 다음과 같다.

- FileStream Connectors (Source)
- FileStream Connectors (Sink)

한편 2.의 경우 POS와 전자상거래 사이트로부터는 RDBMS의 데이터를 받고, 판매 예측 시스템에는 S3에 전달하기 때문에 사용하는 커넥터는 다음과 같다.

- JDBC Connector (Source)
- S3 Connector (Sink)

여기까지 이번에 사용할 환경이다. 그럼 순서대로 만들어보자.

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
> {
>   "name" : "load-zaiko-data",
>   "config" : {
>     "connector.class" : "org.apache.kafka.connect.file.FileStreamSourceConnector",
>     "file" : "/zaiko/latest.txt",
>     "topic" : "zaiko-data"
>   }
> }
> ' | curl -X POST -d @- http://localhost:8083/connectors \
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
> {
>   "name" : "sink-zaiko-data",
>   "config" : {
>     "connector.class" : "org.apache.kafka.connect.file.FileStreamSinkConnector",
>     "file" : "/ec/zaiko-latest.txt",
>     "topics" : "zaiko-data"
>   }
> }
> ' | curl -X POST -d @- http://localhost:8083/connectors \
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
