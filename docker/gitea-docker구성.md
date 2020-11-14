# sqlite3 사용 버전

* 개인용이라면 sqlite로도 충분하다고 생각하여 사용.
  * 20-04-12 기준 설치 성공함



## 설치 순서

```bash
mkdir -p ~/docker/gitea
# /var/services/homes/kdmin/docker/gitea

cd ~/docker/gitea
mkdir gitea
vi docker-compose.yml
```

```yaml
version: "2"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:latest
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "3333:22"
```

```bash
sudo docker-compose up -d
```

* 최초 접속 후 로그인을 누른 후 나오는 초기설정 화면에서 db는 sqlite, localhost를 사용하는 설정값은 nas의 ip로 변경했음. (dns 설정했다면 설정한 도메인)
* 확인 누른 뒤 초기화 작업이 약 10분 소요됐음.
* 초기화 완료 후 최초로 생성한 계정이 관리자 권한이 됨.



# mysql 사용 버전 (사용안함)

* (20-04-12) 시놀로지에서 다시해보니 gitea가 mysql에 접근할 권한이 없으니 어쩌니 하면서 안 됨. mysql에 직접 붙어 계정을 추가해볼까도 했지만 번거로워서 패스하고 그냥 sqlite3 를 쓰는 걸로 설치했음.

## docker-compose.yml

* ref : https://docs.gitea.io/en-us/install-with-docker/

```yaml
version: "2"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:latest
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - DB_TYPE=mysql
      - DB_HOST=db:3306
      - DB_NAME=gitea
      - DB_USER=gitea
      - DB_PASSWD=gitea
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:22"
    depends_on:
      - db

  db:
    image: mysql:5.7
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=gitea
      - MYSQL_USER=gitea
      - MYSQL_PASSWORD=gitea
      - MYSQL_DATABASE=gitea
    networks:
      - gitea
    volumes:
      - ./mysql:/var/lib/mysql
```

## config

* 20-01-13 기준 한글 UI는 잘 동작하지 않았다. 이에 따라 UI를 english로 고정한다.
* 최초 실행 후 생기는 설정 파일을 수정 후 재기동한다.

```bash
vi gitea/gitea/conf/app.ini 
```

```diff
# ...

+ [i18n]
+ LANGS = en-US
+ NAMES = English
```

* 최초 실행 후 아무 버튼이나 누르면 install 과정이 시작된다. 이 때 ROOT_URL은 IP기준으로 넣는다. (localhost 등으로 해두면 원격에서 붙을 때 Preview 기능등이 안 됨.)

## 실행

```bash
sudo docker-compose up -d
sudo docker-compose down
```

