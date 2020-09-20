# 개요

* wsl2를 사용하며 정리한 가이드



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



# 재시작

* powershell 관리자권한에서수행

```
Restart-Service LxssManager
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



# 서비스관리

* `systemd` ,`update-rc.d` 등의 부팅시 서비스 자동 재시작같은게 안 먹힌다.
* 대신 windows에서 wsl 명령어로 직접 vm에 명령을 보낼 수 있으므로 이점을 활용한다. (sudo 권한이 필요하다면 visudo에 등록)







# 트러블슈팅

## DNS 이슈

* https://gist.github.com/coltenkrauter/608cfe02319ce60facd76373249b8ca6