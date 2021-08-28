# 개요

* wsl 개발환경 구축 관련된 스크립트들

## mariadb

```bash
sudo apt-get update
# client만 설치
sudo apt-get install mariadb-client
```

## jenkins


* wsl에 jenkins 구축
    * https://www.jenkins.io/doc/book/installing/linux/#long-term-support-release
    * https://dev.to/davidkou/install-jenkins-in-windows-subsystem-for-linux-wsl2-209

### 설치

```bash
sudo apt-get update
sudo apt-get install openjdk-11-jre-headless

wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
sudo apt-get update
sudo apt-get install jenkins
```

### 주요파일

* `/var/lib/jenkins/secrets/initialAdminPassword`
  * 초기 구동시 물어보는 비번
* `/etc/default/jenkins`
  * 경로, 실행포트 (기본 8080) 등의 주요 설정
* `/var/lib/jenkins/`
  * 기본 jenkins home 경로. job 등이 저장됨.

#### 명령

```bash
sudo service jenkins start
sudo service jenkins restart
sudo service jenkins stop
```

#### 접속

* `http://localhost:8080` 또는 `http://[wsl 의 ip]:8080` 으로 접속
  * `127.0.0.1` 이나 host ip로 안 되는거보면 wsl이 `localhost`에 한해서 자동 포워딩 해주는 기능이 있는 것 같음

### docker-in-docker 방식

* jenkins는 도커에도 구축가능하긴함. (docker-in-docker 방식) 
* 이 방식으로 nas에 올릴까 했는데 램 사용량이 700MB로 무거워서 그냥 wsl에 구축하기로함.
    * 관심있으면 다음 링크 참고
    * https://www.section.io/engineering-education/building-a-java-application-with-jenkins-in-docker/