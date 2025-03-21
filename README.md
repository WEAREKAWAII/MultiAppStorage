# 🗂️ MultiAppStorage

<br>

## 👨‍💻 팀원

| <img src="https://github.com/riyeong0916.png" width="220" />|<img src="https://github.com/CooolRyan.png" width="220" />|<img src="https://github.com/user-attachments/assets/b6694343-dcfb-4c12-88b5-dc9a68fe9f94" width="220"/>|<img src="https://github.com/eundeom.png" width="220" /> |
|:-:|:-:|:-:|:-:|
|김리영<br/>[@riyeong0916](https://github.com/riyeong0916) |나원호<br/>[@CooolRyanJ](https://github.com/CooolRyan)|박정호<br/>[@Jeongho427](https://github.com/Jeongho427)|이은정<br/>[@eundeom](https://github.com/eundeom) |



<br>

## 🧭 프로젝트 목적

- **Docker-Compose**를 통해 다중 컨테이너 실행
- 다수의 애플리케이션이 하나의 데이터베이스 공유 환경 구성
- **컨테이너 볼륨 마운트**를 통해 호스트에 데이터베이스 장애에 대비한 백업 방안 마련

<br>

## 🔧 아키텍쳐
<div align=center>
  <img src="https://github.com/user-attachments/assets/669e2e38-333b-4612-83c2-3f6026aeb29d" width=700/>
</div>

### 🧩 주요 구성 요소
#### 1️⃣ MySQL 컨테이너 (mysqldb)
- 공통 데이터 저장소 역할
- 앱 간 데이터 공유 지원
- 사용자 정의 네트워크 및 볼륨 적용

#### 2️⃣ Spring Boot 애플리케이션 2개 (app1, app2)
- 각각 독립된 컨테이너로 실행
- 동일한 DB에 연결하여 공동 작업 가능
- 포트 분리 (8080, 8081)

#### 3️⃣ Docker 네트워크
- 컨테이너 간 통신을 위한 사용자 정의 브리지 네트워크
- app1, app2, mysqldb가 동일 네트워크에 연결되어 DB 접근 가능

#### 4️⃣ 백업 연동
- mysqldump를 활용한 자동 백업 스크립트 제공
- DB 데이터를 .sql 파일로 추출하여 호스트 디렉토리에 동기화
- cron을 통한 정기 백업

<br>

## ⚙️ 환경 설정

### 1️⃣ `Dockerfile` 생성
```
# Base Image 설정
FROM openjdk:17-slim

# curl 설치 (slim 이미지에는 기본적으로 없음)
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# 작업 디렉토리 설정
WORKDIR /app

# 애플리케이션 JAR 파일 복사
COPY ../setp06_speingDataJPA-0.0.1-SNAPSHOT.jar app.jar

# 환경 변수 설정 (포트 변경이 용이하도록)
ENV SERVER_PORT=8080

# 헬스 체크 설정 (curl을 사용하여 애플리케이션이 정상 작동하는지 확인)
HEALTHCHECK --interval=10s --timeout=30s --retries=3 CMD curl -f http://localhost:${SERVER_PORT}/one || exit 1

# 애플리케이션 실행 (exec 방식 사용)
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 2️⃣ `docker-compose.yaml` 생성

```bash
version: "3.8"

services:
  db:
    container_name: mysqldb
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    networks:
      - spring-mysql-net
    volumes:
      - mysql_data:/var/lib/mysql
      - ~/mysql-backups:/backup
    healthcheck:
      test: ['CMD-SHELL', 'mysqladmin ping -h 127.0.0.1 -u root --password=$${MYSQL_ROOT_PASSWORD} || exit 1']
      interval: 10s
      timeout: 2s
      retries: 100

  app1:
    container_name: springbootapp1
    build:
      dockerfile: ./Dockerfile
    ports:
      - "8080:8080"
    environment:
      MYSQL_HOST: db
      MYSQL_PORT: 3306
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

  app2:
    container_name: springbootapp2
    build:
      dockerfile: ./Dockerfile
    ports:
      - "8081:8080"
    environment:
      MYSQL_HOST: db
      MYSQL_PORT: 3306
      MYSQL_DATABASE: fisa
      MYSQL_USER: user01
      MYSQL_PASSWORD: user01
    depends_on:
      db:
        condition: service_healthy
    networks:
      - spring-mysql-net

networks:
  spring-mysql-net:
    driver: bridge

volumes:
  mysql_data:
```
-  호스트 디렉토리 **`~/mysql-backups:/backup`   → 컨테이너 내부의 `/backup` 경로로 마운트 된다.**
(컨테이너가 `/backup`에 파일을 쓰면, Ubuntu 내에서도 바로 확인할 수 있다.)
- DB, App에 **동일한 네트워크** 추가 및 `yaml` 파일에 추가
  ```
  docker network create spring-mysql-net
  ```

<br>

### 3️⃣ `run.sh` 스크립트 작성

**Docker Compose**를 이용하여 **애플리케이션 빌드하고 실행**하는 `run.sh` 스크립트 작성

```bash
#!/bin/bash

echo "빌드 및 컨테이너 실행 시작"

# 이미지 빌드 및 컨테이너 실행
docker-compose up --build -d

# 로그 확인
docker-compose logs -f
```

- `docker-compose up --build -d` : Docker Compose 파일을 기반으로 컨테이너를 실행
    - `-d` : 백그라운드 모드에서 실행하여 터미널을 차지하지 않고 컨테이너가 실행되도록 함
- `docker-compose logs -f` : 현재 실행 중인 Docker Compose 애플리케이션의 로그 출력
    - `-f` : 로그를 실시간으로 추적 (follow). 로그가 업데이트 될 때 마다 실시간으로 로그를 화면에 출력함

<br>

### 4️⃣ 백업 파일을 저장할 **호스트 디렉토리** 생성

```bash
mkdir -p ~/mysql-backups
```

### 5️⃣ `backup.sh` DB 백업 스크립트 작성

해당 파일은 **MySQL 컨테이너**에서 DB를 **`mysqldump`**를 이용해 **백업**하는 스크립트

MySQL 컨테이너 내에서 `mysqldump` 명령어를 실행하고 결과를 호스트 OS의 `/backup` 디렉토리에 `.sql` 파일로 저장함

```bash
#!/bin/bash

docker exec mysqldb sh -c \
  'mysqldump -uroot -proot fisa > /backup/fisa_$(date +%F_%H-%M-%S).sql'

echo "[✔] Backup completed! Check ~/mysql-backups/"
```

- `docker exec mysqldb sh -c \
  'mysqldump -uroot -proot fisa > /backup/fisa_$(date +%F_%H-%M-%S).sql'`
: MySQL 데이터베이스 fisa의 **백업을 생성**하는 명령어를 실행함

<br>


### 6️⃣ `cron`으로 백업 자동화 

1. crontab에 등록
    
    ```bash
    crontab -e
    ```
    
2. 맨 아래에 추가
    
    ```bash
    0 0 * * * /home/ubuntu/backup.sh >> /home/ubuntu/backup.log 2>&1
    ```

<img src="https://github.com/user-attachments/assets/65de6ae5-ce9c-4338-bc66-08fc6f85fe57" width=700/>

    
( 매일 자정에 자동 백업 + backup.log에 로그 기록 예시 )
    

<br>


## 💾 sql 백업


```
cat ~/mysql-backups/fisa_2025-03-21_15-00-00.sql | docker exec -i mysqldb mysql -uroot -proot
```

- `.sql` 파일이 MySQL 컨테이너 내부 mysql 명령으로 전달되어, 원래대로 DB가 복원

<br>

## 실행 확인

### #️⃣run.sh 파일 실행

- run.sh 파일을 실행하여 docker-compose, Dockerfile 실행

![image](https://github.com/user-attachments/assets/210a0ca1-f02c-4f68-81f6-a4f350c50fea)

### #️⃣컨테이너 생성 및 확인

![image](https://github.com/user-attachments/assets/5d6b29a9-9dcf-4f19-a8f2-e2354f2399a9)


### #️⃣ 포트 확인 후 curl로 접속 확인

- 실행시킨 컨테이너가 잘 실행되고 있는지 확인

<img src="https://github.com/user-attachments/assets/606d27ad-cbf8-4f96-825d-14135d3faf9b" width=600/>

- 실행 중인 컨테이너 포트 확인 후, curl로 접속하여 앱이 잘 실행되는지 확인

<img src="https://github.com/user-attachments/assets/7bfd56c3-969c-431f-96b8-61f3f2ad0854" width=600/>

<img src="https://github.com/user-attachments/assets/dc7f8125-ee18-47df-a05a-564e06f7b1e4" width=600/>

<br>

## 🖥️ 생성 리소스 확인 

### DB 백업 확인 

<img src="https://github.com/user-attachments/assets/e68f1cbc-b4e6-4268-8696-c1c2421cf87d" width=700/>

<br>

위 스크립트들을 사용하여, 매일  19, 20, 21일 자정에 각 백업이 .sql 파일 형식으로 생성된 것을 확인 가능

해당 파일들은 MySQL 컨테이너 내 /backup/ 디렉토리에 저장되었고 주기적인 데이터베이스 백업 자동화가 된 것을 확인할 수 있다.

### 🗂️ 파일별 의미

| 파일/폴더 | 설명 |
| --- | --- |
| `auto.cnf` | MySQL 서버의 UUID를 저장하는 파일 (클러스터링 등에서 사용됨) |
| `binlog.000001`, `binlog.000002`, `binlog.index` | **Binary Log (binlog)**: 트랜잭션 및 변경 사항 기록 |
| `fisa/` | 네가 만든 **`fisa` 데이터베이스**의 실제 데이터 파일이 저장되는 폴더 |
| `mysql/` | MySQL 시스템 데이터베이스 (사용자 계정, 권한 정보 포함) |
| `sys/`, `performance_schema/` | MySQL 내부 시스템 테이블 |
| `ibdata1` | **InnoDB 시스템 테이블스페이스** (트랜잭션 데이터, 테이블 메타데이터 저장) |
| `ib_buffer_pool` | InnoDB 버퍼 풀 정보 (빠른 데이터 로드를 위해 사용) |
| `ibtmp1` | 임시 테이블 저장 공간 |
| `undo_001`, `undo_002` | **Undo 로그 파일** (트랜잭션 롤백에 사용) |
| `server-cert.pem`, `server-key.pem`, `ca.pem`, `ca-key.pem`, `client-cert.pem`, `client-key.pem`, `public_key.pem`, `private_key.pem` | MySQL 보안 인증서 및 키 파일 |

<br>

### *️⃣ **가장 중요한 폴더**

#### 1. `fisa/`

- `fisa`라는 폴더 안에 실제 테이블 데이터 파일(`.ibd`)이 들어 있음.

```bash
ls /var/lib/docker/volumes/08practice_mysql_data/_data/fisa/
```

#### 2. `binlog.*`

- Binary Log (이진 로그) 파일
- MySQL의 **모든 데이터 변경 사항을 기록**하는 로그
- DB 복원할 때 유용함

#### 3. `ibdata1`

- InnoDB 테이블스페이스 파일
- 모든 트랜잭션과 메타데이터를 저장하는 중요한 파일

<br>

### *️⃣ **이 정보를 통해 알 수 있는 것**

1. **MySQL 볼륨(`mysql_data`)이 정상적으로 마운트됨**
    
    → `fisa/` 폴더가 존재하는 걸 보면, DB가 정상적으로 저장되고 있음.
    
2. **MySQL 데이터가 컨테이너 재시작 후에도 유지됨**
    
    → 볼륨을 사용하고 있기 때문에 컨테이너를 삭제하고 다시 실행해도 데이터가 남아 있음.
    
3. **백업할 때 중요한 파일**
    - `fisa/` → 개별 데이터 테이블
    - `binlog.*` → 변경 로그
    - `ibdata1`, `ibtmp1`, `undo_*` → 트랜잭션 정보

<br>

### #️⃣ Docker volume 목록 확인

![image](https://github.com/user-attachments/assets/25b7c0d3-962b-4d02-88a5-0ba37bcf7571)


### #️⃣ Docker volume 정보 상세보기

```bash
docker volume inspect [volume_name]
```
<img src="https://github.com/user-attachments/assets/e41dd800-0d78-4355-98ab-2452d000416a" width=600/>


- /var/lib/docker/volumes/08practice_myql_data/_data에 볼륨 마운트 있는 디렉터리 확인

<img src="https://github.com/user-attachments/assets/a71b7ea8-e291-4c92-8a84-b27af69165db" width=600/>

<br>

### #️⃣ Docker network 확인
<img src="https://github.com/user-attachments/assets/91486e08-716c-4101-b7e6-abb00bd64dee" width=600/>


- 명령어를 통해 08practice_spring-mysql-net network 상에 존재하는 컨테이너 목록 확인
- springbootapp1, springbootapp2, mysqldb 확인 가능

```bash
ubuntu@myserver1:~/08.practice$ docker network inspect 75853b5d03f9
[
    {
        "Name": "08practice_spring-mysql-net",
        "Id": "75853b5d03f9d91fca711f88602f814dc6051d26404e3752a34e7550b9b07f1a",
        "Created": "2025-03-21T12:31:53.549942424+09:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv4": true,
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "532a20296f34dee552ec720732fb16eaffe3b21470c1bdf9187e317b0e30b9d8": {
                "Name": "springbootapp1",
                "EndpointID": "b3399b07bcdb6cd9584beec389967c105cb272513469b4dc6a51fa66ffa083bf",
                "MacAddress": "7e:64:28:be:34:9f",
                "IPv4Address": "172.19.0.4/16",
                "IPv6Address": ""
            },
            "efbcb07a374fa6d1b9ab28f08dd0db0397b90444d5188f0121b9867c79cea5d4": {
                "Name": "springbootapp2",
                "EndpointID": "c35870e21dc417f2ca2de7fade001a29e0418af4d42075b8476f31ecbcd6a5a6",
                "MacAddress": "6a:ea:9b:61:24:0b",
                "IPv4Address": "172.19.0.3/16",
                "IPv6Address": ""
            },
            "f2b3ad1d40838d6bae27ae5123f09b3d20303e08ff79ed4ac1dd6691676c85ad": {
                "Name": "mysqldb",
                "EndpointID": "42e73e54db3718a7bfd9dbd275e5c596de7dd3bb4e384443245d197e2f81cfe3",
                "MacAddress": "de:9c:e4:ee:07:73",
                "IPv4Address": "172.19.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "com.docker.compose.network": "spring-mysql-net",
            "com.docker.compose.project": "08practice",
            "com.docker.compose.version": "2.29.6"
        }
    }
]

```

<br>

