# docker-compose.yml

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

# config

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

# 실행

```bash
sudo docker-compose up -d
sudo docker-compose down
```