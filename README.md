# Model
## kafka-cdc-20250612 (cdc 설정을 통한 )
www.msaez.io/#/storming/C7pO0ZuWtXXxIKenocD9EMPYrxw2/6170b502533a967cfa212d4417243d1e

- 주문 데이터 실시간 동기화 확인 (CDC 성공)
- 주문 생성: http POST :8081/orders ... 명령으로 Order 서비스에 새로운 주문 생성 (예: order/3).
- 동기화 확인: http GET :8082/syncOrders로 Marketing 서비스에서 동기화된 주문 목록 조회.
- 결과: order/3이 기존 order/1, order/2와 함께 조회됨.
→ CDC를 통한 실시간 데이터 동기화 성공!

![스크린샷 2025-06-12 141817](https://github.com/user-attachments/assets/06cca263-0450-42a2-b83b-40fa5e679bc2)
![스크린샷 2025-06-12 161647](https://github.com/user-attachments/assets/c0f74180-d9a5-406b-9ab9-69b8c617e023)
![스크린샷 2025-06-12 162407](https://github.com/user-attachments/assets/b73da7d6-2b0f-476e-96b4-c0799499436f)
![스크린샷 2025-06-12 162602](https://github.com/user-attachments/assets/1eaaf63b-9f72-4078-82fc-ac2251d238fb)
![스크린샷 2025-06-12 163218](https://github.com/user-attachments/assets/7af6ef2e-144e-49f8-a51d-b384596e827f)

---
## 터미널 작성 참고용
1. sdk install java
- 처음 실행하면 mysql 3306포트가 실행되었음
---
2. Kafka connect를 위한 JDBC드라이브 다운 (2181포트 주키퍼 실행)
```
git clone https://github.com/acmexii/kafka-connect.git
cd kafka-connect
```
---
3. Kafka를 수동설치 후 주키퍼 실행
```
curl "https://archive.apache.org/dist/kafka/2.7.1/kafka_2.13-2.7.1.tgz" -o ./kafka-2.7.1.tgz
tar xvfz kafka-2.7.1.tgz

cd kafka_2.13-2.7.1/
bin/zookeeper-server-start.sh config/zookeeper.properties &
```
---
4. kafka 데몬실행 (9092포트 실행)
```
cd kafka-connect
cd kafka_2.13-2.7.1/
bin/kafka-server-start.sh config/server.properties &
```
---
5. JDBC 커넥터 실행
- $PWD는 현재 카프카 홈이라는 내폴더라는걸 저장한다. 저장경로, 윈도우 환경변수랑 비슷한거다?
```
- cd ~
- cd /workspace/kafka-cdc-20250612/kafka-connect/kafka_2.13-2.7.1/
- export kafka_home=$PWD
- echo "export kafka_home=$PWD" >> ~/.bashrc
- source ~/.bashrc
- echo $kafka_home
- mkdir -p connectors
- cd connectors

- cp ../../confluentinc-kafka-connect-jdbc-10.2.5.zip ./
- unzip confluentinc-kafka-connect-jdbc-10.2.5.zip

- plugin.path=/workspace/kafka-cdc-20250612/kafka-connect/kafka_2.13-2.7.1/connectors
```
---
6. VI가 아니라 VS-CODE를 이용해서 connect-distributed.properties 맨아래 코드 추가
---
7. kafka connect 서버 실행 (소스 커넥터 실행)
```
cd $kafka_home
bin/connect-distributed.sh config/connect-distributed.properties 
$kafka_home/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```
---
8. 소스커넥터 설치 터미널에 추가
```
curl -i -X POST -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/ \
    -d '{
    "name": "mysql-source-connector",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
        "connection.url": "jdbc:mysql://localhost:3306/my-database?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC&characterEncoding=UTF-8",
        "connection.user":"root",
        "connection.password":"1234",
        "mode":"incrementing",
        "useSSL":"false",
        "incrementing.column.name" : "id",
        "table.whitelist" : "ORDER_TABLE",
        "topic.prefix" : "SYNC_",
        "tasks.max" : "1"
    }
}'
```
```
http :8083/connectors / mysql DB확인
```
---
9. 오더 실행
```
Order의 application.yml을 열어 default profile의 datasource를 확인 및 수정함
cd order
mvn spring-boot:run
```
---
10. SYNC_ORDER_TABLE 토픽 추가 목록 확인
```
http POST :8081/orders productId=1 qty=10 customerId=1000 price=10000
http POST :8081/orders productId=2 qty=20 customerId=2000 price=20000
$kafka_home/bin/kafka-console-consumer.sh --bootstrap-server 127.0.0.1:9092 --topic SYNC_ORDER_TABLE --from-beginning
```
---
11. 싱크 커넥터 설치
```
curl -i -X POST -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/ \
    -d '{
    "name": "mysql-sink-connector",
    "config": {
        "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
        "connection.url": "jdbc:mysql://localhost:3306/my-database?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC&characterEncoding=UTF-8",
        "connection.user":"root",
        "connection.password":"1234",
        "useSSL":"false",        
        "auto.create":"true",       
        "auto.evolve":"true",       
        "delete.enabled":"false",
        "tasks.max":"1",
        "topics":"SYNC_ORDER_TABLE"
    }
}'
```
---
```
12. my sql clien로 복제된 테이블 확인
cd mysql
docker-compose exec -it master-server bash
mysql --user=root --password=1234
use my-database;
show tables;
```
---
13. marketing 서비스 수정 및 실행 (8082포트)
```
  datasource:
    url: jdbc:mysql://${_DATASOURCE_ADDRESS:localhost:3306}/${_DATASOURCE_TABLESPACE:my-database}?serverTimezone=UTC&characterEncoding=UTF-8
    username: ${_DATASOURCE_USERNAME:root}
    password: ${_DATASOURCE_PASSWORD:1234}
    driverClassName: com.mysql.cj.jdbc.Driver 
```
```
cd marketing
mvn spring-boot:run
```
---
13. 동기화 데이터 조회 
```
http GET :8082/syncOrders

http POST :8081/orders productId=1 qty=10 customerId=1000 price=10000
http GET :8082/syncOrders
```
---
## Before Running Services
### Make sure there is a Kafka server running
```
cd kafka
docker-compose up
```
- Check the Kafka messages:
```
cd kafka
docker-compose exec -it kafka /bin/bash
cd /bin
./kafka-console-consumer --bootstrap-server localhost:9092 --topic 
```

## Run the backend micro-services
See the README.md files inside the each microservices directory:

- order
- marketing


## Run API Gateway (Spring Gateway)
```
cd gateway
mvn spring-boot:run
```

## Test by API
- order
```
 http :8088/orders id="id" customerId="customerId" qty="qty" price="price" productId="productId" 
```
- marketing
```
 http :8088/syncOrders id="id" customerId="customerId" qty="qty" price="price" productId="productId" 
 http :8088/suggestions id="id" productId="productId" suggestProductId-1="suggestProductId-1" suggestProductId-2="suggestProductId-2" suggestProductId-3="suggestProductId-3" 
```


## Run the frontend
```
cd frontend
npm i
npm run serve
```

## Test by UI
Open a browser to localhost:8088

## Required Utilities

- httpie (alternative for curl / POSTMAN) and network utils
```
sudo apt-get update
sudo apt-get install net-tools
sudo apt install iputils-ping
pip install httpie
```

- kubernetes utilities (kubectl)
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

- aws cli (aws)
```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

- eksctl 
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

