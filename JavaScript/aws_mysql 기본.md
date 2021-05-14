# 데이터 베이스 핵심 요약



[aws 사이트]

인스턴스 시작 > Ubuntu Server 18.04 LTS (HVM), SSD Volume Type 선택 > 두번째 t2.micro

\> 6.보안 그룹 구성 > 포트번호 

3306 - mysql 0.0.0.0/0

27017 - monggodb 0.0.0.0/0

// 80 - http? 안해도 됌

\> 시작 후 키 ~.pem 생성



[터미널 이동]

$ cd 위치 잡음 

$ mkdir .ssh

$ mv ~/Downloads/fds_test.pem ~/Documents/aws/.ssh // pem키를 .ssh폴더 안으로 옮김

$ cd .ssh

$ ls -al // 권한 보기

// -rw-r--r--@ 1 lee_haeun staff 1692 4 27 19:09 fds_test.pem >> 나만 접근 가능하게 권한 바꿔야 함

$ chmod 400 fds_test.pem 

// -r--------@ 1 lee_haeun staff 1692 4 27 19:09 fds_test.pem >> 이걸로 바뀜



인스턴스 접속 키워드 

$ ssh -i ~/Documents/aws/.ssh/fds_test.pem [ubuntu@52.79.113.78](mailto:ubuntu@52.79.113.78) // ssh -i (띄어쓰기) ~/pem 비번 파일경로 (띄) ubuntu@aws 퍼블릭 ip 복붙

yes



// 여기서 부터 앞에 **ubuntu@ip-172-31-40-129**:**~**$ 이게 붙음 서버로 들어감

// sudo 란 관리자 권한



apt-get 업데이트

$ sudo apt-get update -y // apt-get 이란 npm같은 관리자 같은 것. -y 는 yes 자동으로 해줌

$ sudo apt-get upgrade -y 



MySQL Server 설치
 $ sudo apt-get install -y mysql-server mysql-client 



MySQL 패스워드 설정 

$ sudo mysql 

mysql> SELECT user,authentication_string,plugin,host FROM mysql.user; 

mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'rada'; 

mysql> FLUSH PRIVILEGES;
 mysql> SELECT user,authentication_string,plugin,host FROM mysql.user; 

mysql> exit 



비번은 haha 임!!!

\c << mysql에 잘못 입력 엔터했을때 입력했던거 클리어 하는 방법이다.



패스워드 설정이 끝났다. 이제 앞으로 mysql에 접근하려면 sudo mtsql이 아니라 mysql -u root -p으로 접근후 비번 확인 해야 한다.

$ mysql -u root -p
 Enter password: haha



\# 외부접속 설정 // 외부에서 접속할때 127.0.0.1 인 내 아이피만 접속가능한게 아니라 모두 접근 가능하게 0.0.0.0 으로 바꿔준다.
 $ sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf 



vim 열림 /bind로 글씨 찾을 수 있음/ i를 눌러서 글씨를 수정해준후 esc :wq 로 저장종료 한다.

\# bind-address를 127.0.0.1을 0.0.0.0 으로 변경

bind-address = 0.0.0.0



\# 외부접속 패스워드 설정
 mysql> grant all privileges on *.* to root@'%' identified by haha’; 



// 부차적인 명령어들

\# 서버 시작 종료 상태 확인
 $ sudo systemctl start mysql.service 

$ sudo systemctl stop mysql.service
 $ sudo systemctl restart mysql.service 

$ sudo systemctl status mysql.service 

\# 설정 후 서버 재시작으로 설정 내용 적용 

$ sudo systemctl restart mysql.service 

\# password 변경 : rada로 패스워드를 변경하는 경우
 mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'rada'; 



##  

## 샘플 데이터 다운 후 mysql에 추가해주기



샘플 데이터들 https://drive.google.com/drive/folders/17_3hH4raJiwdh12j__-dVkorvW3ualdn 에서 다운 받음. 



$ scp -i ~/Documents/aws/.ssh/fds_test.pem ~/Downloads/drive-download-20200427T063115Z-001/* ubuntu@52.79.113.78:~/

scp는 파일 이동 명령어. 

scp (띄) 키 비번경로.pem (띄) 이동시킬 원본 데이터들 파일 컴퓨터 폴더 경로:~이동될 서버 경로



// 이후 인스턴스 서버에서 ls 해보면 데이터 파일들이 이동되어 있다.

**ubuntu@ip-172-31-40-129**:**~**$ ls

employees.sql pyenv.sh sakila-data.sql sakila-schema.sql world.sql zigbang.py



$ mysql -u root -p 이후 비번 haha치고 mysql시작



mysql> create database world;
 mysql> create database sakila;
 mysql> create database employees; 

mysql> quit 



// 생성 삭제 참고

mysql> create database 이름 // 데이터 베이스 생성

mysql> show databases; // 전체 데이터 베이스 확인. 복수형 s 주의!

mysql> drop database 이름; // 만든 데이터 삭제



\# 데이터 베이스에 데이터 추가
 $ mysql -u root -p world < world.sql 

$ mysql -u root -p sakila < sakila-schema.sql 

$ mysql -u root -p sakila < sakila-data.sql
 $ mysql -u root -p employees < employees.sql 

방금전에 mysql안에 만든 데이터베이스 안에다가 다운로드 받아서 :~/에 옮겨온 데이터들을 집어넣는 과정이다.

이게 되면 workbrench에서 볼 때 데이터 베이스 안에 데이터가 들어와있고 안했으면 데이터 베이스 껍데기만 남아있다.





// workbrench 열기

\+ 버튼을 누른 후 이름을 정하고 hostname 의 ip에 52.79.113.78(aws 퍼블릭 ip)를 넣는다. 생성

// 모델 생성하기

File - New Model 선택
 더블 클릭으로 이름 설정, add diagram더블클릭으로 물리적 모델 열기



테이블 클릭 후 테이블 생성, 이름 설정 , 키 이름들 설정 … 여러 테이블 만든후 테이블끼리 관계선 이어주기

모델 다 작성했으면 상단 메뉴 Database > forward engineer 클릭 > fds(맨처음 만들때 폴더?)선택 > 컨티뉴 계속하다가 코드 50번줄 쯤 VISABLE어찌고 두개 지워줌.. 버전문제 때문..



생성 후 Database > reverse engineer 클릭 > 모델 적용할 데이터 베이스 클릭 > 컨티뉴 이후 끝



// workbrench 쿼리 작성 예시

use world; #데이터베이스 선택



select * # * 모든 컬럼 데이터

from country; # country table



select code, name, continent, population

from country;



select *

from city

WHERE countrycode = "KOR";

c shift eneter