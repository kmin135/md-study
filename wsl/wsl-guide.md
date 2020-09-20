# 개요

* wsl2를 사용하며 정리한 가이드



# 초기세팅

* alias 세팅

```bash
# wsl의 현재경로를 windows explorer에 띄움
echo "alias open='explorer.exe .'" >> ~/.bashrc
source ~/.bashrc
```





# 자원사용제한

* `%UserProfile%\.wslconfig` 추가

```bash
[wsl2]
memory=6GB
swap=0
processors=4
localhostForwarding=true
```

* 전체사용법

```bash
[wsl2]
kernel=<path>              # An absolute Windows path to a custom Linux kernel.
memory=<size>              # How much memory to assign to the WSL2 VM.
processors=<number>        # How many processors to assign to the WSL2 VM.
swap=<size>                # How much swap space to add to the WSL2 VM. 0 for no swap file.
swapFile=<path>            # An absolute Windows path to the swap vhd.
localhostForwarding=<bool> # Boolean specifying if ports bound to wildcard or localhost in the WSL2 VM should be connectable from the host via localhost:port (default true).

# <path> entries must be absolute Windows paths with escaped backslashes, for example C:\\Users\\Ben\\kernel
# <size> entries must be size followed by unit, for example 8GB or 512MB
​```

# 참고 : https://docs.microsoft.com/en-us/windows/wsl/release-notes#build-18945
```



# 명령어

```bash
# 설치된 distro 목록
wsl --list
# 상태 조회
wsl -l -v
# 특정 가상 머신 종료
wsl -t <NAME>
```



### 재시작

* powershell 관리자권한에서수행

```
Restart-Service LxssManager
```





# 서비스관리

* `systemd` ,`update-rc.d` 등의 부팅시 서비스 자동 시작같은게 안 먹힌다.
* 대신 windows에서 wsl 명령어로 직접 vm에 명령을 보낼 수 있으므로 이점을 활용한다. 



### visudo 세팅

* sudo를 비밀번호 없이 수행하도록 수정

```bash
# 에디터를 vim.basic으로 변경
sudo update-alternatives --config editor
```

```diff
- %sudo   ALL=(ALL:ALL) ALL
+ %sudo   ALL=(ALL:ALL) NOPASSWD:ALL
```



### windows 작업 스케줄러 세팅

* 작업 스케줄러로 windows 로그인시 필요한 wsl 명령들을 수행하도록 함
* 예를 들어 cron을 항상 시작시키고 싶다면 아래와 같이 한다.

```
작업 스케줄러 > 기본 작업 만들기

트리거 : "로그인할 때"
동작 : "프로그램 시작"
- 프로그램/스크립트 : wsl
- 인수 추가 : sudo service cron start
```







# 트러블슈팅

## DNS 이슈

* https://gist.github.com/coltenkrauter/608cfe02319ce60facd76373249b8ca6