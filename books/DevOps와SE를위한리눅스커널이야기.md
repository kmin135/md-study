# DevOps와 SE를 위한 리눅스커널이야기

* 동일하게 리눅스 기초를 다루다보니 만화로 배우는 리눅스 시스템관리 책과 중복되는 내용이 많다. 차이점을 보면 제목대로 이 책이 좀 더 실무 운영 환경에 맞게 작성되었다.



## 공통



### 커널 파라미터 확인 

* 모든 정보는 /proc/sys 이하에서 관리
* sysctl 명령을 통해 조회 가능
  * `sysctl vm.overcommit_memory` : vm.overcommit_memory 파라미터 조회
  * `sysctl -a` : 모든 파라미터 조회

### 커널 파라미터 수정

* 임시 수정
  * `sysctl -w vm.overcommit_memory=1`
* 재부팅시 적용
  * `/etc/sysctl.conf` 수정



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

* PR : 프로세스 실행 우선 순위. (여기 보이는 값은 기본PR+NI로 결정된 우선 순위값.)
* NI : 기본 PR을 얼마만큼 조정할지 결정.
* VIRT : 프로세스에 할당된 가상 메모리 크기
  * 예약의 개념이므로 이 값이 큰 것 자체는 문제가 되지 않음
* RES : VIRT에서 실제로 사용중인 물리 메모리 크기
* SHR : 다른 프로세스와 공유하는 메모리 크기 (ex `glibc 같은 공통 라이브러리`)
* S : 프로세스 상태



### VIRT와 RES

* VIRT 는 예약의 개념이며 (overcommit) 그 최대값은 커널 파라미터에 따라 존재할 수도 있고 무한대일 수도 있음
* `vm.vm.overcommit_memory` 으로 설정
  * 0 : 기본값. overcommit 최대값 = page cache + swap + slab reclaimable 영역의 합계
  * 1 : 무조건 커밋. 최대값 없음
  * 2 : 제한적 커밋. vm.overcommit_ratio 값과 swap 영역 크기를 토대로 계산됨.



### 프로세스 상태

![Image](./imgs/devops/process_status.jpg)

* D: uninterruptible sleep. I/O 대기중. 이 상태의 프로세스들은 Run Queue에서 빠져나와 Wait Queue 에 들어감
* R : 실행 중인 프로세스
* S : sleep 상태. D 와의 차이점은 요청한 리소스를 즉시 사용할 수 있는지 여부.
* T : traced or stopped 상태. strace 등으로 프로세스의 시스템콜을 추적중일 때의 상태임. 보통은 볼 일 없음
* Z : 좀비. 부모 프로세스가 죽은 자식 프로세스

![Image](./imgs/devops/process_lifecycle.jpg)

* 정상적인 프로세스의 생성과 종료는 위와 같음
* 좀비는 부모 프로세스가 죽었는데도 자식 프로세스가 남아있거나 자식 프로세스가 죽기 전에 비정상적인 동작으로 부모 프로세스가 죽는 경우가 발생함.
* 좀비 프로세스 자체는 스케줄러에 의해 선택되지 않으므로 시스템 리소스를 차지하지 않음. (CPU, 메모리 등을 사용하지 않음)
* 좀비가 문제가 되는 것은 좀비 프로세스가 점유하는 PID 때문임. PID 도 유한한 자원이므로 좀비 프로세스가 계속 생기면 PID 고갈이 발생할 수 있음
  * `sysctl kernel.pid_max` 로 최대 pid개수를 확인할 수 있음. 이 크기를 초과해서 프로세스를 생성할 수 없음.



### 프로세스 우선순위

![Image](./imgs/devops/process_pr.jpg)



* CPU마다 Run Queue가 존재하며 RQ에는 우선순위별로 프로세스가 연결되어 있음. 스케줄러는 유후 프로세스가 깨어나거나 특정 프로세스가 스케줄링을 양보하는 경우 등의 상황에서 우선순위 정보를 토대로 디스패처에 건내줌.
* 우선순위 값이 낮을수록 우선순위가 높은 것임. (PR 20보다 PR 0이 우선권을 가지는 것)
* 기본 PR 값은 20이며 여기에 NI를 더한 값이 실제 적용되는 PR 값임.
* CPU 경합이 일어나는 상황에서 우선순위가 높은수록 빨리 끝나게 됨. 물론 경합이 없는 경우 (CPU 수가 실행중인 프로세스보다 많다면 경합할 이유가 없음) 에는 우선순위가 영향을 주지 않을 수도 있음.
* 실행중인 프로세스는 `renice` 명령을 통해 우선순위 조정 가능
* PR 값이 RT (RealTime) 로 출력되는 프로세스들은 커널에서 사용하는 데몬들임. 이름 그대로 특정 시간안에 반드시 종료되어야 하는 중요 프로세스들에 부여되며 RT 스케줄러라는 별도의 스케줄러가 제어함. 당연히 다른 스케줄러보다 우선순위가 높음.