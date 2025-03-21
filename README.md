# MultiAppStorage

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
  <img src="https://github.com/user-attachments/assets/8e7863fc-6b0f-4a0b-8133-218fcb4e5b23" width=600/>
</div>

### 주요 구성 요소
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

## ⚙️ 환경 구성

### 1️⃣ 백업 파일을 저장할 **호스트 디렉토리** 생성

```bash
mkdir -p ~/mysql-backups
```

<br>

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
      **- ~/mysql-backups:/backup**
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

⭐ 호스트 디렉토리 **`~/mysql-backups:/backup`   → 컨테이너 내부의 `/backup` 경로로 마운트 된다.**

(컨테이너가 `/backup`에 파일을 쓰면, Ubuntu 내에서도 바로 확인할 수 있다.)

<br>

### 3️⃣ `backup.sh` DB 백업 스크립트 작성

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

### 4️⃣ `run.sh` 스크립트 작성

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

### ✳ 중간 실행 확인

1. Docker Compose로 MySQL 컨테이너 실행하기
    
    ```bash
    docker-compose up
    ```
    
2. DB 백업 스크립트 실행
    
    ```bash
    ./backup.sh
    ```
    
    **✅ 실행 결과**
<div align=center>
  <img src="https://github.com/user-attachments/assets/85a3a7fb-4c19-484b-b6f7-2f7d929df36d" width=700/>
</div>

<br>

### 5️⃣ `cron`으로 백업 자동화 

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

## 🖥️ 전체 실행 확인

<img src="https://github.com/user-attachments/assets/e68f1cbc-b4e6-4268-8696-c1c2421cf87d" width=700/>

<br>

위 스크립트들을 사용하여, 매일  19, 20, 21일 자정에 각 백업이 .sql 파일 형식으로 생성된 것을 확인 가능

해당 파일들은 MySQL 컨테이너 내 /backup/ 디렉토리에 저장되었고 주기적인 데이터베이스 백업 자동화가 된 것을 확인할 수 있다.
