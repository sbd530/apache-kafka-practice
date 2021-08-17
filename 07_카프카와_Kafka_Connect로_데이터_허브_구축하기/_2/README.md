## 7.6 월별 판매 예측하기

2.에서는 전자상거래 사이트의 PostgreSQL과 POS의 MariaDB를 판매 예측 시스템의 S3에 연결한다.

이번 실습을 위해서는 앞의 파일 연계보다 많은 시스템을 준비해야 한다.

### 7.6.1 전자상거래 사이트 준비

전자상거래 사이트용 데이터베이스를 준비한다. 서버를 하나 구축하고 거기에 PostgreSQL 9.4를 설치한다. 여기에서는 서버명을 ec-data-server로 한다.

먼저 PostgreSQL을 설치한다. 설치 절차와 저장소 URl 등은 변경될 수도 있으므로 최신 정보는 공식 사이트를 참조하길 바란다. 명령의 출력 결과는 생략한다. 다음과 같이 설치에 성공했는지 확인한다.

```bash
(ec-data-server)$ sudo yum install https://download.postgresql.org/pub/repos/\
> yum/9.4/redhat/rhel-7-x86_64/pgdg-centos94-9.4-3.noarch.rpm
(ec-data-server)$ sudo yum install postgresql94 postgresql94-server
```

데이터베이스 클러스터를 초기화한다.

```bash
(ec-data-server)$ sudo /usr/pgsql-9.4/bin/postgresql94-setup initdb
Initializing database ... OK
```

카프카 클러스터에서 PostgreSQL에 연결할 수 있도록 설정한다.

```bash
(ec-data-server)$ sudo vim /var/lib/pgsql/9.4/data/postgresql.conf
  (다음과 같이 설정한다.)
listen_addresses = '*'

(ec-data-server)$ sudo vim /var/lib/pgsql/9.4/data/pg_hba.conf
  (맨 끝부분에, 접속원인 카프카 클러스터의 IP주소를 포함하는 범위를 추가한다.)
host all all 10.0.2.0/24 md5
```

durltjdml tjfwjddms 10.0.2.0/24의 범위에 있는 서버 액세스를 허용하고 암호를 요구하는 설정이다. 그러므로 각자 자신의 환경에 맞게 설정하도록 한다. 설정을 다했다면 PostgreSQL을 실행한다.

```bash
(ec-data-server)$ sudo systemctl start postgresql-9.4
```

PostgreSQL 서버의 설치와 초기 설정이 완료되었으므로 다음은 전자상거래 사이트 역할을 위해 필요한 데이터를 넣는다.

우선 postgres 사용자의 권한으로 데이터베이스를 만든다. 데이터베이스의 이름은 ec로 한다.

```bash
(postgres@ec-data-server)$ createdb ec
```

터미널형 프론트 엔드의 psql을 실행한다.

```bash
(postgres@ec-data-server)$ psql ec
psql (9.4.18)
Type "help" for help.

ec=@
```

데이터를 저장할 테이블을 만든다. 만들 테이블은 전자상거래 사이트의 매출을 기록하는 테이블로 ec_uriage라는 이름으로 만든다. 이 테이블의 데이터를 데이터 허버에서 처리할 것이 아니기 때문에 어떤 스키마도 좋지만 칼럼으로 일련 번호/판매 시간/ID/상품 ID/수량/단가 등을 넣도록 한다.

```sql
CREATE TABLE ec_uriage (
    seq        bigint PRIMARY KEY,
    sales_time timestamp,
    sales_id   varchar(80),
    item_id    varchar(80),
    amount     int,
    unit_price int
);
```

이 DDL을 조금 전에 실행한 psql에서 실행한다. 실행했다면 만들어진 테이블을 확인한다.

```bash
ec=@ \d

          List of relations
 Schema |   Name    | Type  |  Owner
 public | ec_uriage | table | postgres
(1 row)
```

테이블이 만들어졌으므로 사용자를 만들고 이 테이블에 권한을 설정하자. 이것도 psql에서 실행한다. 사용자는 Kafka Connect로 부터 접속하는 전용 사용자 connectuser를 준비해둔다.

```sql
CREATE USER connectuser with password 'connectpass';
GRANT ALL ON ec_uriage TO connectuser;
```

이어서 초기 데이터를 넣어보자.

적절한 매출 데이터를 몇 건 넣어둔다. 확인을 위한 것이므로 데이터 자체의 내용은 다소 부자연스러워도 문제는 없다. 이러한 INSERT문을 psql에서 실행한다(명령의 출력은 생략).

```sql
INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
    VALUES (1, '2018-10-05 11:11:11', 'ECSALES00001', 'ITEM001', 2, 300);
INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
    VALUES (2, '2018-10-01 11:11:11', 'ECSALES00001', 'ITEM002', 1, 5800);
INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
    VALUES (3, '2018-10-02 12:12:12', 'ECSALES00002', 'ITEM001', 4, 298);
INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
    VALUES (4, '2018-10-02 12:12:12', 'ECSALES00002', 'ITEM003', 1, 2500);
INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
    VALUES (5, '2018-10-02 12:12:12', 'ECSALES00002', 'ITEM004', 1, 198);
INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
    VALUES (6, '2018-10-02 12:12:12', 'ECSALES00002', 'ITEM005', 1, 273);
```

투입한 데이터를 확인해두자.

```bash
ec=@ select * from ec_uriage;
  (생략)
(6 rows)
```

이제 전자상거래 사이트가 완성됐다.

### 7.6.2 POS 준비

POS용 데이터베이스도 전자상거래 사이트와 동일하게 RDBMS 서버를 준비한다. POS에는 MariaDB를 사용한다. 실제로 다양한 시스템을 도입하다 보면 다른 RDBMS를 도입한 시스템도 많이 존재한다. 혹여 그러한 상태라고 해도 다행히 Kafka Connect에서는 조그만 문제로 보일 것이다.

1대의 서버를 구축하고 그 위 MariaDB를 설치한다. 여기서는 서버 이름을 pos-data-server로 한다.

먼저 MariaDB를 설치한다. PostgreSQl 설치 때와 마찬가지로 절차나 저장소 URL 등은 변경될 수 있으므로 최신 정보는 공식 사이트를 참조한다. 명령의 출력 결과는 생략한다. 설치가 성공했는지 확인하자.

```bash
(pos-data-server)$ sudo yum install mariadb mariadb-server
```

MariaDB를 실행한다.

```bash
(pos-data-server)$ sudo systemctl start mariadb
```

mysql 명령어 프롬프트를 실행한다.

```bash
(pos-data-server)$ mysql -u root
mysql -u root
Welcome to the MariaDB monitor.
  (생략)
MaridDB [(none)]>
```

post라는 이름의 데이터베이스를 만들고 전환한다.

```bash
MaridDB [(none)]> CREATE DATABASE pos;
Query OK, 1 row affected (0.00 sec)

MaridDB [(none)]> USE pos;
Database changed
```

테이블 pos_uriage를 만든다. POS는 매출을 기록하는 테이블이 하나만 있으면 충분하다. 전자상거래를 고려해서 동일한 칼럼 구성으로 다음과 같은 DDL을 작성한다.

```sql
CREATE TABLE pos_uriage (
    seq        bigint PRIMARY KEY,
    sales_time timestamp,
    sales_id   varchar(80),
    shop_id    varchar(80),
    item_id    varchar(80),
    amount     int,
    unit_price int
);
```

이것을 앞서 실행한 mysql 명령어 프롬프트상에서 실행해 테이블을 확인한다.

```bash
MaridDB [(none)]> show tables;
+---------------+
| Tables_in_pos |
+---------------+
|  pos_uriage   |
+---------------+
1 row in set (0.01 sec)
```

Kafka Connect에서 접속하기 위한 사용자 connectuser를 만들고 권한을 설정한다.

```sql
CREATE USER 'connectuser' IDENTIFIED BY 'connectpass';
GRANT ALL PRIVILEGES ON pos_uriage TO connectuser;
```

앞서와 마찬가지로 적절한 매출 데이터를 몇 건 넣어둔다. 이쪽도 데이터가 다소 부자연스러워도 문제없다.

```sql
INSERT INTO ec_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
    VALUES (1, '2018-10-05 11:11:11', 'POSSALES00001', 'SHOP001', 'ITEM001', 2, 300);
INSERT INTO ec_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
    VALUES (2, '2018-10-05 11:11:11', 'POSSALES00001', 'SHOP001', 'ITEM001', 2, 198);
INSERT INTO ec_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
    VALUES (3, '2018-10-05 11:11:11', 'POSSALES00001', 'SHOP002', 'ITEM002', 2, 327);
INSERT INTO ec_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
    VALUES (4, '2018-10-05 11:11:11', 'POSSALES00002', 'SHOP002', 'ITEM002', 2, 300);
INSERT INTO ec_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
    VALUES (5, '2018-10-05 11:11:11', 'POSSALES00003', 'SHOP003', 'ITEM003', 2, 512);
INSERT INTO ec_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
    VALUES (6, '2018-10-05 11:11:11', 'POSSALES00002', 'SHOP003', 'ITEM001', 2, 512);
```

투입된 데이터를 확인한다.

```bash
MaridDB [(none)]> SELECT * FROM pos_uriage;
  (결과 생략)
6 rows in set (0.00 sec)
```

이제 POS 시스템도 완성됐다.

그런데 앞서 언급했듯이 컨플루언트 플랫폼에서 설치된 카프카에는 MariaDB 드라이버가 포함되어 있지 않다. 따라서 수동으로 드라이버를 설치해야 한다. 드라이버는 MariaDB에 접속하는 모든 서버에서 필요하다. 즉, kafka-broker01, kafka-broker02,kafka-broker03 서버에서 필요하다.

MariaDB Connector/J의 jar 파일을 다운로드한다. 다운로드했다면 jar 파일을 JDBC Connector가 인식할 수 있는 위치에 둔다. 다음과 같이 하면 좋은 것이다.

```bash
$ sudo cp mariadb-java-client-*.jar /usr/share/java/kafka-connect-jdbc
```

이제 POS의 준비도 완료됐다.

### 7.6.3 판매 예측 시스템 준비

판매 예측 시스템은 S3 경유로 연계한다. 여기서는 S3에 버킷만 만들어두면 된다. 이후는 이러한 버킷이 만들어졌다는 가정하의 작업이므로 적적히 상황에 맞추어 준비한다.

- 지역 : ap-northeast-1
- 버킷 이름 : datahub-sales

Kafka Connect의 커넥터가 S3에 엑세스하기 위해서는 인증이 필요하다. 작업을 위해 AWS계정의 액세스 키를 생성해두자. 액세스 키를 커넥터에 전달하는 방법은 여러 가지가 있는데 여기에서는 ~/.aws/credentials 파일에 기술한다. 이미 설정되어 있는 경우는 이 단계를 건너뛴다. 이 설정은 S3에 액세스하는 모든 서버에 필요하다. 즉, kafka-broker01, kafka-broker02, kafka-broker03 서버에 필요하다

```bash
$ mkdir ~/.aws
$ touch ~/.aws/credentials
$ chmod 600 ~/.aws/credentials
$ vim ~/.aws/credentials
```

[default] 부분을 다음과 같이 작성한다.

```shell
[default]
aws_access_key_id = your_access_key_id
aws_secret_access_key = your_secret_access_key
```

이제 모든 시스템의 준비가 끝났다. 이제부터 Kafka Connect로 모든 시스템을 연결해보자.

### 7.6.4 Kafka Connect 실행

이번에는 많은 시스템이 등장하기 때문에 모든 시스템을 IP 주소로 쓰면 어떤 서버인지 알기 어려워 일일이 파악하는 것이 힘들다. 따라서 이 장에서 등장하는 서버는 IP 주소를 사용하지 않고 서버명으로 기술한다. 등장하는 모든 서버의 /etc/host에 서버명이 있는 것을 전제로 설명한다. 다른 환경을 사용하는 사람은 적절히 상황에 맞추어 바꾸면 된다.

```shell
XX.XX.XX.XX kafka-broker01
XX.XX.XX.XX kafka-broker02
XX.XX.XX.XX kafka-broker03
XX.XX.XX.XX ec-data-server
XX.XX.XX.XX pos-data-server
```

먼저 Kafka Connect를 실행한다. 방금 전 Kafka Connect 자제를 중지시켰기 때문에 다시 시작한다. 절차는 아까와 동일하지만 다른 점이 딱 하나 있다. 이번에는 Kafka Connect를 여러 서버에서 실행한다는 점이다.

우선 실행을 위한 설정 파일을 작성해야 한다. 카프카 클러스터의 모든 서버(kafka-broker01,02,03)에 설정 파일을 만들어야 한다. 여러 대에서 Kafka Connect를 사용하는 경우 bootstrap.severs도 함께 두는 것이 좋다.

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

```bash
$ cp /etc/kafka/connect-distributed.properties \
> connect-distributed-2.properties
$ vim connect-distributed-2.properties
  (중간 생략)
# bootstrap.servers=kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092
bootstrap.servers=localhost:9092
group.id=connect-cluster-datahub-2
  (이후 생략)
```

이 설정 파일을 사용하여 Kafka Connect를 시작해보자. 이것도 카프카 클러스터의 모든 서버에서 실행한다.

```bash
$ connect-distributed ./connect-distributed-2.properties
```

Kafka Connect를 실행했다. 계속해서 전자상거래 사이트의 매출 데이터를 로드하는 커넥터를 실행한다. 이때 카프카 클러스터 중 하나에서 실행하길 바란다. 또는 클러스터 외부의 카프카 클라이언트에서 실행해도 상관없다.

```bash
(kafka-client)$ echo '
{
  "name" : "load-ecsales-data",
  "config" : {
    "connector.class" : "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url" : "jdbc:postgresql://ec-data-server/ec",
    "connection.user" : "connectuser",
    "connection.password" : connectpass",
    "mode" : "incrementing",
    "incrementing.column.name" : "seq",
    "table.whitelist" : "ec_uriage",
    "topic.prefix" : "ecsales_",
    "tasks.max" : "3"
  }
}
' | curl -X POST -d @- http://localhost:8083/connectors \
# http://kafka-broker01:8083/connectors
> --header "content-Type:application/json"

{"name":"load-ecsales-data","config":{"connector.class" : "io.confluent.connect.jdbc.JdbcSourceConnector","connection.url" : "jdbc:postgresql://ec-data-server/ec","connection.user" : "connectuser","connection.password":"connectpass","mode":"incrementing","incrementing.column.name":"seq","table.whitelist":"ec_uriage","topic.prefix":"ecsales_","tasks.max":"3","name":"load-ecsales-data"},"tasks":[],"type":null}
```

설정 내용은 다음과 같다.

- connection.url, connection.user, connection.password
  \- JDBC에 접속하기 위한 정보를 지정한다.
- mode, incrementing.column.name
  \- 실행하고 있는 동안 커넥터는 JDBC를 통해 RDBMS를 폴링한다. 그리고 변경이 있으면 그 내용을 읽어 카프카에 전달한다. 변경 사항을 감지하는 방법이 몇 가지 있지만 여기에서는 incrementing를 지정하고 있다. 이것은 단순히 증가하는 칼럼의 값을 보고 갱신 유무를 판별하는데 변경 사항을 감지하고 싶은 칼럼은 incrementing.column.name에 지정한다. 그래서 이번에는 seq 칼럼의 값이 단순히 증가하도록 값을 입력해야 한다. mode는 그 밖에도 bulk, timestamp+incrementing 등을 지정할 수 있으므로 관심이 있다면 문서를 살펴보길 바란다.
- table.whitelist
  \- 로드할 대상의 테이블을 지정한다. table.blacklist 등으로 테이블을 제외할 수도 있는데 이 둘은 서로 배타적이다.
- topic.prefix
  \- 카프카에 데이터를 넣을 때 토픽명을 결정할 접두어를 지정한다. 테이블명에 접두어가 붙은 이름이 토픽명이 된다.
- tasks.max
  \- 이 커넥터에서 만들어지는 태스크 수의 최대 수를 지정한다.

앞서와 마찬가지로 POS의 매출 데이터를 로드하는 커넥터를 실행한다.

```bash
(kafka-client)$ echo '
{
  "name" : "load-possales-data",
  "config" : {
    "connector.class" : "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url" : "jdbc:mysql://pos-data-server/pos",
    "connection.user" : "connectuser",
    "connection.password" : connectpass",
    "mode" : "incrementing",
    "incrementing.column.name" : "seq",
    "table.whitelist" : "pos_uriage",
    "topic.prefix" : "possales_",
    "tasks.max" : "3"
  }
}
' | curl -X POST -d @- http://localhost:8083/connectors \
# http://kafka-broker01:8083/connectors
> --header "content-Type:application/json"

{"name":"load-possales-data","config":{"connector.class":"io.confluent.connect.jdbc.JdbcSourceConnector","connection.url":"jdbc:mysql://pos-data-server/pos","connection.user":"connectuser","connection.password":"connectpass","mode":"incrementing","incrementing.column.name":"seq","table.whitelist":"pos_uriage","topic.prefix":"ecsales_","tasks.max":"3","name":"load-possales-data"},"tasks":[],"type":null}
```

실행 중인 커넥터 목록을 살펴보자.

```bash
# (kafka-client)$ curl http://kafka-broker01:8083/connectors
(kafka-client)$ curl http://localhost:8083/connectors
["load-ecsales-data","load-possales-data"]
```

투입한 커넥터가 둘 다 동작하고 있기 때문에 데이터가 제대로 카프카에 로드되어 있는지 kafka-console-consumer를 사용하여 확인해보자.

**load-ecsales-data**

```bash
(consumer-client)$ kafka-console-consumer \
# --bootstrap-server kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092
> --bootstrap-server localhost:9092 \
> --topic ecsales_ec_uriage --from-beginning
  (JSON 데이터 생략)
Processed a total of 6 messages
```

**load-possales-data**

```bash
(consumer-client)$ kafka-console-consumer \
# --bootstrap-server kafka-broker01:9092,kafka-broker02:9092,kafka-broker03:9092
> --bootstrap-server localhost:9092 \
> --topic possales_pos_uriage --from-beginning
  (JSON 데이터 생략)
Processed a total of 6 messages
```

전자상거래 사이트와 POS, 양쪽 모두의 데이터가 카프카에 로드되어 있음을 알 수 있다.

그럼 계속해서 싱크 쪽의 커넥터도 실행해보자. 판매 예측 시스템의 S3로 출력하는 커넥터는 다음과 같이 실행한다.

```bash
(kafka-client)$ echo '
{
  "name" : "sink-sales-data",
  "config" : {
    "connector.class" : "io.confluent.connect.s3.S3SinkConnector",
    "s3.bucket.name" : "datahub-sales",
    "s3.region" : "ap-northeast-1",
    "storage.class" : "io.confluent.connect.s3.storage.S3Storage",
    "format.class" : "io.confluent.connect.s3.format.json.JsonFormat",
    "flush.size" : 3,
    "topics" : "possales_pos_uriage,ecsales_ec_uriage",
    "tasks.max" : "3"
  }
}
' | curl -X POST -d @- http://localhost:8083/connectors \
# http://kafka-broker01:8083/connectors
> --header "content-Type:application/json"

{"name":"sink-sales-data","config":{"connector.class" : "io.confluent.connect.s3.S3SinkConnector","s3.bucket.name":"datahub-sales","s3.region":"ap-northeast-1","storage.class":"io.confluent.connect.s3.storage.S3Storage","format.class":"io.confluent.connect.s3.format.json.JsonFormat","flush.size":3,"topics":"possales_pos_uriage,ecsales_ec_uriage","tasks.max":"3","name":"sink-sales-data"},"tasks":[],"type":null}
```

설정 내용은 다음과 같다.

- s3.bucket.name, s3.region
  \- 데이터를 출력하는 S3의 버킷 정보를 지정한다. 사전에 준비한 S3의 버킷명과 지역을 설정하자.
- storage.class
  \- S3용 스토리지 인터페이스를 지정한다. 위의 예와 같이 하는 게 좋다.
- format.class
  \- 데이터를 출력할 때 포맷 클래스를 지정해야 한다. 여기에서는 사람이 읽을 수 있도록 JSON 형식으로 출력한다.
- flush.size
  \- 파일에 커밋할 때의 레코드 수를 지정한다.

커넥터가 실행되었는지 실행 중인 커넥터 목록을 보고 확인해보자.

```bash
# (kafka-client)$ curl http://kafka-broker01:8083/connectors
(kafka-client)$ curl http://localhost:8083/connectors
["sink-sales-data","load-ecsales-data","load-possales-data"]
```

싱크 커넥터도 동작을 시작한 것 같다. 그럼 S3을 확인해보자. AWS CLI나 Web UI로 지정한 버킷을 확인하면 이와 같은 파일이 생성되어 있음을 확인할 수 있다.

```
datahub-sales/
    └─ topics/
        ├─ ecsales_ec_uriage
        │   └─ partition=0
        │       ├─ ecsales_ec_uriage+0+0000000000.json
        │       └─ ecsales_ec_uriage+0+0000000003.json
        └─ possales_pos_uriage
            └─ partition=0
                ├─ possales_ec_uriage+0+0000000000.json
                └─ possales_ec_uriage+0+0000000003.json
```

환경이나 데이터에 따라 조금씩 다를 수 있겠지만 비슷한 구조로 출력되었을 것이다. JSON의 내용도 살펴보자.

```json
ecsales_ec_uriage+0+0000000000.json

{"seq":"1","sales_time":1538737871000,"sales_id":"ECSALES00001","item_id":"ITEM001","amount":2,"unit_price":300}
{"seq":"2","sales_time":1538737871000,"sales_id":"ECSALES00001","item_id":"ITEM002","amount":1,"unit_price":5900}
{"seq":"3","sales_time":1538737871000,"sales_id":"ECSALES00002","item_id":"ITEM001","amount":4,"unit_price":298}
```

전자상거래 사이트의 매출 데이터가 들어 있음을 알 수 있다. POS 데이터도 확인해보자. POS 데이터가 들어 있을 것이다. 이것으로 전자상거래 사이트와 POS 데이터가 S3에 연결되어 있음을 확인했다.

자그럼 1.의 경우와 마찬가지로 커넥터가 실행 중인 동안에는 데이터 소스에 변경이 있으면 그것이 S3에까지 전달될 것이다. 이것도 확인해보자. 방금 전자상거래 사이트에서 매출이 있어 한 줄이 추가되었다고 하자.

```bash
(postgres)$ psql ec
psql (9.4.18)
Type "help" for help.

ec=@ INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
VALUES (7, '2018-10-02 13:13:13', 'ECSALES00003', 'ITEM001', 1, 300);
```

S3에 변경이 있는지 확인해보자. 이 책과 도일하게 먼저 초기 데이터를 6행 작성한 사람은 S3에 아무런 변화도 없을 거라 생각한다. 이것은 flush.size의 설정 때문이다. 이 값을 3으로 설정하여 싱크를 실행했기 때문에 3행 데이터가 모이지 않으면 파일에 출력되지 않는다. 시험 삼아 두 줄을 더 추가하자. 한 행씩 추가하여 S3의 모습을 확인해보자.

```sql
INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
  VALUES (8, '2018-10-02 14:14:14', 'ECSALES00004', 'ITEM001', 1, 300);
INSERT INTO ec_uriage(seq, sales_time, sales_id, item_id, amount, unit_price)
  VALUES (9, '2018-10-02 14:14:14', 'ECSALES00004', 'ITEM002', 1, 5800);
```

S3을 확인하면 새로운 파일이 생성됐다.

똑같이 POS 쪽도 확인해보자. POS에도 매출이 있었다.

```bash
(pos-data-server)$ mysql -u root
mysql -u root
  (생략)
MariaDB [(none)]> use pos;
  (생략)
Database changed
MariaDB [(none)]> INSERT INTO pos_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
VALUES (7, '2018-10-12 13:13:13', 'POSSALES00004', 'SHOP001', 'ITEM001', 2, 300);
MariaDB [(none)]> INSERT INTO pos_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
VALUES (8, '2018-10-12 13:13:13', 'POSSALES00004', 'SHOP001', 'ITEM001', 2, 300);
MariaDB [(none)]> INSERT INTO pos_uriage(seq, sales_time, sales_id, shop_id, item_id, amount, unit_price)
VALUES (9, '2018-10-12 13:13:13', 'POSSALES00004', 'SHOP001', 'ITEM001', 2, 198);
```

한 행씩 추가하여 어느 타이밍에 S3에 파일이 생성되는지를 확인해보는 것이 좋을 것이다. 3행마다 새로운 파일이 생성되는 것을 확인할 수 있다.

이것으로 데이터 허브가 동작하고 있는 동안 RDBMS상 데이터 변경이 항상 S3에 반영됨을 알 수 있다.

이것으로 대략적인 데이터 허브의 동작을 확인할 수 있었다. 뒷정리를 하고 작업을 끝내자. 실행 중인 커넷터를 제거한다.

```bash
# http://kafka-broker01:8083/...
(kafka-client)$ curl -X DELETE http://localhost:8083/connectors/load-ecsales-data
(kafka-client)$ curl -X DELETE http://localhost:8083/connectors/load-possales-data
(kafka-client)$ curl -X DELETE http://localhost:8083/connectors/sink-sales-data
```

Kafka Connect를 정짛한다. 실행한 connect-distributed를 [Ctrl] + [C]로 종료한다.
