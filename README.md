# MultiAppStorage

## 👨‍💻 팀원

| <img src="https://github.com/riyeong0916.png" width="220" />|<img src="https://github.com/CooolRyan.png" width="220" />|<img src="https://github.com/user-attachments/assets/fb4ba63b-b913-4401-bfb6-43debbc8b260" width="220"/>|<img src="https://github.com/eundeom.png" width="220" /> |
|:-:|:-:|:-:|:-:|
|김리영<br/>[@riyeong0916](https://github.com/riyeong0916) |나원호<br/>[@CooolRyanJ](https://github.com/CooolRyan)|박정호<br/>[@Jeongho427](https://github.com/Jeongho427)|이은정<br/>[@eundeom](https://github.com/eundeom) |


## 프로젝트 목적

- Docker-Compose를 통해 다중 컨테이너 실행
- 하나의 자원을 공유해 사용하는 다중 애플리케이션 환경 구성
- 데이터베이스 장애에 대비한 백업 방안 마련




## sql 백업


```
cat ~/mysql-backups/fisa_2025-03-21_15-00-00.sql | docker exec -i mysqldb mysql -uroot -proot
```

- .sql 파일이 MySQL 컨테이너 내부 mysql 명령으로 전달되어, 원래대로 DB가 복원.
