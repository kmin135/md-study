# 개요

* wsl2 + docker 환경의 gitlab 구성 및 관리 방법 가이드
* 참고 : https://github.com/sameersbn/docker-gitlab



## 201118 노트

* wsl에 구성해서 써보다가 그냥 폐기
* 그럭저럭 잘 돌긴 하는데 종종 pc가 절전모드에서 복구될 때 docker-engine이 내려가있는 상황이 있었고 또 docker-engine 업데이트 후 먹통이 되기도 했다.
* 또 개인용으로는 어차피 오버스펙이기도 하다.
* 대체제는 시놀로지 도커 위에 gitea로 결정



# backups

* 기본으로 7일치 백업이 자동생성됨 (`GITLAB_BACKUP_*` 파라미터들로 스케줄링 정책이 결정됨)
* host쪽 경로가 `gitlab` 이라면 `gitlab/backups/` 에 백업 tar 파일이 생성되는 형식
* 이 파일들을 cron 으로 백업대상 디렉토리에 복사하는 형태로 주기적인 백업 가능



### 즉시 backup

```bash
docker-compose rm -sf gitlab
docker-compose run --rm gitlab app:rake gitlab:backup:create
docker-compose up -d 
```



### backup파일 외부 이중백업

* cron 활용

```bash
crontab -e

# 매일 0시에 백업. 경로는 알맞게 조정해서 사용
* 0 * * * cp ~/docker/gitlab/gitlab/backups/*.tar /mnt/d/Backups/gitlab/
# ...
```



# restore

* 우선 동일한 docker-compose.yml 을 활용하여 새로운 환경에서 gitlab을 올림
* 새로운 gitlab이 모두 올라온 것을 확인한 뒤 복구 대상 backup tar파일을 신규 환경의 `gitlab/backups/` 에 복사함
* 아래의 순서로 진행함

```bash
docker-compose run --rm gitlab app:rake gitlab:backup:restore

# 아래와 같이 복사해둔 tar파일이 list 되면 정상임
#> 1600562005_2020_09_20_13.3.4_gitlab_backup.tar (created at 20 Sep, 2020 - 09:33:25 KST)
# 해당 파일명을 그대로 적어주면 복구가 시작되며, 안내에 따라 진행하면 복구가 완료됨
```



# upgrade

* TODO
* 실제로 해본적은 없음. 당연하지만 백업 후 작업 필수. 한다면 restore 과정을 통해 새로운 환경에 구성한 뒤 upgrade해야할 것.
* https://github.com/sameersbn/docker-gitlab#upgrading



# docker-compose.yml

* 20-09-20 기준 최초 사용한 docker-compose 파일
* volumes 는 상대경로로 지정함

```yaml
# https://github.com/sameersbn/docker-gitlab
version: '2.3'

services:
  redis:
    restart: always
    image: redis:5.0.9
    command:
    - --loglevel warning
    volumes:
    - ./redis:/var/lib/redis

  postgresql:
    restart: always
    image: sameersbn/postgresql:11-20200524
    volumes:
    - ./db:/var/lib/postgresql
    environment:
    - DB_USER=gitlab
    - DB_PASS=password
    - DB_NAME=gitlabhq_production
    - DB_EXTENSION=pg_trgm,btree_gist

  gitlab:
    restart: always
    image: sameersbn/gitlab:13.3.4
    depends_on:
    - redis
    - postgresql
    ports:
    - "10080:80"
    - "10022:22"
    volumes:
    - ./gitlab:/home/git/data
    healthcheck:
      test: ["CMD", "/usr/local/sbin/healthcheck"]
      interval: 5m
      timeout: 10s
      retries: 3
      start_period: 5m
    environment:
    - DEBUG=false

    - DB_ADAPTER=postgresql
    - DB_HOST=postgresql
    - DB_PORT=5432
    - DB_USER=gitlab
    - DB_PASS=password
    - DB_NAME=gitlabhq_production

    - REDIS_HOST=redis
    - REDIS_PORT=6379

    - TZ=Asia/Seoul
    - GITLAB_TIMEZONE=Seoul

    - GITLAB_HTTPS=false
    - SSL_SELF_SIGNED=false

    - GITLAB_HOST=pc.kmin.me
    - GITLAB_PORT=10080
    - GITLAB_SSH_PORT=10022
    - GITLAB_RELATIVE_URL_ROOT=
    - GITLAB_SECRETS_DB_KEY_BASE=long-and-random-alphanumeric-string
    - GITLAB_SECRETS_SECRET_KEY_BASE=long-and-random-alphanumeric-string
    - GITLAB_SECRETS_OTP_KEY_BASE=long-and-random-alphanumeric-string

    - GITLAB_ROOT_PASSWORD=
    - GITLAB_ROOT_EMAIL=

    - GITLAB_NOTIFY_ON_BROKEN_BUILDS=true
    - GITLAB_NOTIFY_PUSHER=false

    - GITLAB_EMAIL=notifications@example.com
    - GITLAB_EMAIL_REPLY_TO=noreply@example.com
    - GITLAB_INCOMING_EMAIL_ADDRESS=reply@example.com

    - GITLAB_BACKUP_SCHEDULE=daily
    - GITLAB_BACKUP_TIME=01:00

    - SMTP_ENABLED=false
    - SMTP_DOMAIN=www.example.com
    - SMTP_HOST=smtp.gmail.com
    - SMTP_PORT=587
    - SMTP_USER=mailer@example.com
    - SMTP_PASS=password
    - SMTP_STARTTLS=true
    - SMTP_AUTHENTICATION=login

    - IMAP_ENABLED=false
    - IMAP_HOST=imap.gmail.com
    - IMAP_PORT=993
    - IMAP_USER=mailer@example.com
    - IMAP_PASS=password
    - IMAP_SSL=true
    - IMAP_STARTTLS=false

    - OAUTH_ENABLED=false
    - OAUTH_AUTO_SIGN_IN_WITH_PROVIDER=
    - OAUTH_ALLOW_SSO=
    - OAUTH_BLOCK_AUTO_CREATED_USERS=true
    - OAUTH_AUTO_LINK_LDAP_USER=false
    - OAUTH_AUTO_LINK_SAML_USER=false
    - OAUTH_EXTERNAL_PROVIDERS=

    - OAUTH_CAS3_LABEL=cas3
    - OAUTH_CAS3_SERVER=
    - OAUTH_CAS3_DISABLE_SSL_VERIFICATION=false
    - OAUTH_CAS3_LOGIN_URL=/cas/login
    - OAUTH_CAS3_VALIDATE_URL=/cas/p3/serviceValidate
    - OAUTH_CAS3_LOGOUT_URL=/cas/logout

    - OAUTH_GOOGLE_API_KEY=
    - OAUTH_GOOGLE_APP_SECRET=
    - OAUTH_GOOGLE_RESTRICT_DOMAIN=

    - OAUTH_FACEBOOK_API_KEY=
    - OAUTH_FACEBOOK_APP_SECRET=

    - OAUTH_TWITTER_API_KEY=
    - OAUTH_TWITTER_APP_SECRET=

    - OAUTH_GITHUB_API_KEY=
    - OAUTH_GITHUB_APP_SECRET=
    - OAUTH_GITHUB_URL=
    - OAUTH_GITHUB_VERIFY_SSL=

    - OAUTH_GITLAB_API_KEY=
    - OAUTH_GITLAB_APP_SECRET=

    - OAUTH_BITBUCKET_API_KEY=
    - OAUTH_BITBUCKET_APP_SECRET=

    - OAUTH_SAML_ASSERTION_CONSUMER_SERVICE_URL=
    - OAUTH_SAML_IDP_CERT_FINGERPRINT=
    - OAUTH_SAML_IDP_SSO_TARGET_URL=
    - OAUTH_SAML_ISSUER=
    - OAUTH_SAML_LABEL="Our SAML Provider"
    - OAUTH_SAML_NAME_IDENTIFIER_FORMAT=urn:oasis:names:tc:SAML:2.0:nameid-format:transient
    - OAUTH_SAML_GROUPS_ATTRIBUTE=
    - OAUTH_SAML_EXTERNAL_GROUPS=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_EMAIL=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_NAME=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_USERNAME=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_FIRST_NAME=
    - OAUTH_SAML_ATTRIBUTE_STATEMENTS_LAST_NAME=

    - OAUTH_CROWD_SERVER_URL=
    - OAUTH_CROWD_APP_NAME=
    - OAUTH_CROWD_APP_PASSWORD=

    - OAUTH_AUTH0_CLIENT_ID=
    - OAUTH_AUTH0_CLIENT_SECRET=
    - OAUTH_AUTH0_DOMAIN=
    - OAUTH_AUTH0_SCOPE=

    - OAUTH_AZURE_API_KEY=
    - OAUTH_AZURE_API_SECRET=
    - OAUTH_AZURE_TENANT_ID=
```

