---
title: "about docker-mysql"
date: 2022-01-13 08:52:00 -0400
categories: Tools
---

도커 설치
우분투 shell에 접속하여 도커를 설치한다.

# 최신 버전으로 패키지 업데이트
sudo apt-get update

# 도커 다운을 위해 필요한 패키지 설치
sudo apt-get install apt-transport-https // 패키지 관리자가 https를 통해 데이터 및 패키지에 접근할 수 있도록 해준다.
sudo apt-get install ca-certificates // certificate authority에서 발행되는 디지털 서명. SSL 인증서의 PEM 파일이 포함되어 있어 SSL 기반 앱이 SSL 연결이 되어있는지 확인할 수 있다.
sudo apt-get install curl // 특정 웹사이트에서 데이터를 다운로드 받을 때 사용한다.
sudo apt-get install software-properties-common // *PPA를 추가하거나 제거할 때 사용한다.

# curl 명령어로 도커의 공식 GPG 키를 추가
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add

# 도커의 공식 GPG 키가 추가된 것을 확인
sudo apt-key fingerprint 0EBFCD88

# 도커의 저장소를 추가, 등록
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# 최신 버전으로 패키지 업데이트
sudo apt-get update

# 도커 설치
sudo apt-get install -y docker-ce

# 도커 실행해보기
sudo usermod -aG docker ubuntu

# 도커 compose 설치
sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 권한 설정 해주기
sudo chmod +x /usr/local/bin/docker-compose

# 링크 파일 생성
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose



도커가 설치되었다면, docker-compose.yml을 이용하여 mysql을 설치해보겠다.

docker라는 디렉토리를 만들어서 진행해보자

mkdir docker
cd docker
vi docker-compose.yml
docker-compose.yml 파일에 아래와 같이 적어준다.

version: '3' # docker compose 버전 
services:
  local-db:
    image: library/mysql:5.7
    container_name: subway_local # 컨테이너 이름
    restart: always
    ports:
      - 13306:3306 # 로컬의 13306 포트를 컨테이너의 3306포트로 연결
    environment:
      MYSQL_USER: air
      MYSQL_PASSWORD: air
      MYSQL_ROOT_PASSWORD: root
      TZ: Asia/Seoul
    volumes:
      - ./db/mysql/data:/var/lib/mysql
      - ./db/mysql/init:/docker-entrypoint-initdb.d
아래의 명령어를 실행하면 도커 mysql이 설치된다.

sudo docker-compose up -d
Mysql 설정
컨테이너에 접속해서 mysql 설정을 해보자.

docker exec -it [컨테이너 이름] bash # 컨테이너에 접속.
mysql -u root -p # Mysql 접속 
root 유저를 사용하기 보단 다른 유저를 사용하는 것이 좋다. 우리는 아까 docker-compose.yml에 MYSQL_USER 값을 줬기 때문에 air라는 유저가 생성되있을 것이다.

유저를 확인해보고 싶다면 아래의 명령어로 확인해 볼 수 있다.

mysql> use mysql;
mysql> select user, host from user;


참고로 host의 %는 모든 ip에서 접근할 수 있다는 뜻이다.

air에게 모든 db 및 테이블에 접근 권한을 부여해주자.

mysql> grant all privileges on *.* to 'air'@'%'; # 권한 부여
mysql> flush privileges; # 권한 적용
그리고 사용할 DB(spring boot의 applicaiton.yml 파일에 설정해 놓은 DB이름)를 생성해준다.(utf8 설정도 해줘야 한글과 관련 문제가 생기지 않는다.)

mysql> CREATE DATABASE [db_name] DEFAULT CHARACTER SET utf8 COLLATE utf8_general_ci;
잘 돌아가면 성공! 👏👏👏

MySQL 대소문자 관련 이슈
똑같이 따라 했는데 빈 생성 오류 같은 이상한 오류가 뜰 수 있다.


이는 우분투 mysql의 대소문자 구분 기본 설정이 대소문자 구분함으로 되어있기 때문이다. schema.sql 쿼리문을 바꿔주던가 mysql 설정을 변경해줘야한다.

mysql 설정을 변경해서 이 에러를 해결해보겠다.

먼저 mysql 컨테이너에 접속한다.

docker exec -it [컨테이너 이름] bash # 컨테이너에 접속.
그리고 아주 높은 확률로 docker 컨테이너에 vi가 설치되어있지 않을 것이다. 환경설정 파일을 수정해야하므로 vim을 설치해주자.

root@[컨테이너id]:/# apt-get update
root@[컨테이너id]:/# apt-get install vim
이제 설정파일에 lower_case_table_names의 속성 값을 줘야한다. 기본 값은 0이다. 0이 대소문자를 구분한다는 뜻이고, 이 값을 1로 정해준다.

root@[컨테이너id]:/# vi /etc/mysql/mysql.conf.d/mysqld.cnf //설정 파일 접속
lower_case_table_names = 1을 추가해주자.


이렇게 해 준 다음, 이전 실행시 남아있는 쓰레기 값들이 있어서 이것까지 지워주자.

root@[컨테이너id]:/# rm -rf /var/lib/mysql/[DB_NAME]
그 후 mysql 재시작 해주기!

root@[컨테이너id]:/# service mysql restart
