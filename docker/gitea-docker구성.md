# gitea sqlite3 사용 버전

* 개인용이라면 sqlite로도 충분하다고 생각하여 사용.
  * 20-04-12 기준 설치 성공함
* 공식 가이드 : https://docs.gitea.io/en-us/install-with-docker

## 설치 순서

```bash
mkdir -p ~/docker/gitea
cd ~/docker/gitea
mkdir gitea
vi docker-compose.yml
```
```yaml
# 21-08-28 기준
version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:1.15.0
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
# 최초 구동
sudo docker-compose up -d
```

* 최초 접속 후 로그인을 누른 후 나오는 초기설정 화면에서 db는 sqlite, localhost를 사용하는 설정값은 nas의 ip로 변경했음. (dns 설정했다면 설정한 도메인)
* 확인 누른 뒤 초기화 작업이 약 10분 소요됐음.
* 초기화 완료 후 최초로 생성한 계정이 관리자 권한이 됨.

## config

* 초기설정 화면에서 db는 sqlite, ROOT_URL 등의 설정값은 nas의 IP를 설정했음.
    * localhost 등으로 해두면 원격에서 붙을 때 Preview 기능 등이 안 됨.
    * 추후 `~/docker/gitea/gitea/gitea/conf/app.ini` 에서 변경 가능.
* 20-01-13 에는 한글 UI가 잘 안나왔었는데 이 경우 설정파일에서 UI를 영어로 고정한 적이 있음.
    * 21-08-28에 확인한 `1.15.0` 버전에서는 한글도 잘 됨.
```bash
vi ~/docker/gitea/gitea/gitea/conf/app.ini
```
```bash
# 설정들...
[i18n]
LANGS = en-US
NAMES = English
```
```bash
# 재기동
sudo docker-compose restart
```

## 구동 명령

```bash
# 시작
sudo docker-compose up -d
# 재시작
sudo docker-compose restart
# 종료
sudo docker-compose down
```

## 업그레이드

* 공식 가이드 복붙. 안전하게 하려면 데이터 폴더를 백업해두고 할 것.
```bash
# Edit `docker-compose.yml` to update the version, if you have one specified
# Pull new images
docker-compose pull
# Start a new container, automatically removes old one
docker-compose up -d
```

# mysql 사용 버전 (사용안함)

* (20-04-12) 시놀로지에서 해보니 gitea가 mysql에 접근할 권한이 없으니 어쩌니 하면서 안 됨. mysql에 직접 붙어 계정을 추가해볼까도 했지만 번거로워서 패스하고 그냥 sqlite3 를 쓰는 걸로 설치했음.
* 관심있으면 공식가이드 참고