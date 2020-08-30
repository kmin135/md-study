# DevOps와 SE를 위한 리눅스커널이야기

* 동일하게 리눅스 기초를 다루다보니 만화로 배우는 리눅스 시스템관리 책과 중복되는 내용이 많다. 차이점을 보면 제목대로 이 책이 좀 더 실무 운영 환경에 맞게 작성되었다.



## 1장 시스템 구성 정보 확인하기

* 시스템의 문제 분석을 위해서는 현재 시스템의 구성 정보를 볼 줄 알아야함.



### 커널 정보 확인

* `uname -a` : 커널 버전 확인



### 하드웨어 정보 확인

* dmidecode 명령을 통해 CPU, 메모리, 시스템 등의 주요 하드웨어 정보 확인 가능. 주로 쓰는 건 bios, system, processor, memory



### CPU

* `dmidecode -t bios` : bios 정보
* `dmidecode -t system` : 제조사, 모델명 등
* `dmidecode -t processor` : cpu 정보 (소켓, 코어)
  * 소켓 : 물리적인 cpu 개수
  * 코어 : 물리적인 cpu 안의 컴퓨팅 코어 개수
* `cat /proc/cpuinfo` : cpu 정보를 보는 두 번째 방법
* `lscpu` : cpu 정보를 보는 세 번째 방법. NUMA 정보를 함께 보여줌.



### 메모리

* `dmidecode -t memory`
* memory 키워드는 크게 Physical Memory Array와 Memory Device 의 두 영역으로 나눌 수 있음
  * Physical Memory Array : 하나의 CPU 소켓에 함께 할당된 물리 메모리 그룹. NUMA라는 개념을 통해 각각의 CPU가 사용할 수 있는 로컬 메모리 개념.
  * Memory Device : 실제로 시스템에 설치된 메모리
* `dmidecode -t memory | grep -i size` 로 보면 실제로 설치된 메모리들의 크기를 확인할 수 있으며 이의 합계는 `free` 명령과 일치함. (다를 경우 메모리 인식에 문제가 있는 것)



### 디스크

* `df -h` : 파티션 (ex: `/, /data1, ...`), 디스크(ex: `/dev/sda, ...`) 등의 정보. 디스크 방식인 sda, hda, vda 등은 컨트롤러 방식에 따라 구분됨.
  * sda : SATA, SAS, SCSI 방식
  * hda : IDE 방식
  * vda : 가상 하이퍼바이저 기반의 디스크
* `smartctl -a /dev/sda` : 디스크의 물리적인 정보 (제조사, 시리얼번호, 펌웨어 정보, ...)
  * 서버의 경우 대부분 RAID 컨트롤러가 달려 있어 바로 정확한 정보를 알 수 없는 경우가 많음. (Product에 LOGICAL VOLUME 등으로 표기됨) 이 경우 RAID 컨트롤러 드라이버 정보까지 명시하면 정보를 볼 수 있음.
  * RAID 컨트롤러 드라이버는 제조사에 따라 다름 : cciss, megaraid, hpt, areca 등 (시스템에서 사용중인 RAID 컨트롤러 드라이버 정보는 lsmod 로 확인)
  * `smartctl -a /dev/sda -d cciss,0` : HP사의 cciss RAID 컨트롤러로 0번째 디스크 베이의 정보를 출력하라



### 네트워크

* `lspci | grep -i ether` : 네트워크 카드 정보
* `ethtool eth0` : 네트워크 연결 상태
  * 네트워크 카드가 지원 가능한 최대 속도
  * 현재 연결되어 있는 속도
  * 네트워크 연결 정상여부
* `ethtool -g eth0` : Ring Buffer 크기 확인
  * Ring Buffer : 네트워크 카드의 버퍼 공간. 모든 패킷은 이 버퍼 공간에 복사된후 커널에 복사되므로 RB 크기가 너무 작으면 네트워크 성능 저하가 있을 수 있음. maximums, current 값을 같도록 세팅하는 것이 좋음.
* `ethtool -G eth0 rx 18811` : Current RB 값 수정
* `ethtool -k eth0` : 네트워크 카드의 성능 최적화 옵션 확인
* `ethtool -K eth0` : 네트워크 카드의 성능 최적화 옵션 수정
* `ethtool -i eth0` : 네트워크 카드의 커널 모듈 정보 표시 (드라이버명, 버전 등)



## 2장 top을 통해 살펴보는 프로세스 정보들

* top 기본 기본 3초 간격으로 시스템 상태 출력
* `top -b -n 1 ` : 1번만 출력하고 종료



### top 명령어 보는법

```
top - 02:11:47 up 36 min,  1 user,  load average: 0.00, 0.00, 0.00
```

* 구동 후 지난 시간 (uptime)
* 로그인중인 사용자 수
* 시스템 Load Average

```
Tasks: 110 total,   1 running, 109 sleeping,   0 stopped,   0 zombie
```

* 구동 중인 프로세스 개수

````
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   1923.2 total,    125.0 free,   1421.0 used,    377.2 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.    346.8 avail Mem
```

* CPU, Memory, swap 사용량

```
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
   1809 root      20   0       0      0      0 I   0.3   0.0   0:00.35 kworker     
      1 root      20   0  168676  12692   8292 S   0.0   0.6   0:05.11 systemd
      2 root      20   0       0      0      0 S   0.0   0.0   0:00.00 kthreadd
      3 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 rcu_gp
```

