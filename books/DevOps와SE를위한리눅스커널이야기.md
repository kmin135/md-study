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

```
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



### top 명령어 tip

* c를 누르면 자세한 프로세스 실행경로가 나옴
* `Shift + M` : 메모리 내림차순 정렬
* `Shift + P` : CPU 사용량 내림차순 정렬



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



## 3장 Load Average와 시스템 부하

### Load Average의 정의

*  1, 5, 15분 동안 실행 상태 또는 I/O 대기 중인 프로세스 수의 평균값
*  Core 수는 고려되지 않으므로 같은 Load Average 라도 Core 수에 따라 해석해야함.



### Load Average 보기

* `uptime` 명령
* strace로 확인해보면 단순히 `/proc/loadavg` 파일을 출력하는 것



### 한계점과 vmstat

* 시스템 부하의 기본척도이지만 이것만으로는 CPU에 부하가 걸린건지 I/O 병목에 따른 부하인지 확인이 어려움
* `vmstat` 으로 실행중인 프로세스의 개수와 (`r 열`) I/O 대기중인 프로세스의 개수를 (`b열`) 구분해서 볼 수 있음.
  * `vmstat 1` : 1초마다 갱신하여 출력



### 파이썬 예제

* cpu bound 

```python
#!/usr/bin/python
test = 0
while True:
    test = test +1
```

* cpu bound (multithread)

```python
#!/usr/bin/python
import threading

def infinite():
    test = 0
    while True:
        test = test + 1

threads = []
for i in range(10):
    thread = threading.Thread(target=infinite, args=[])
    thread.start()

for thread in threads:
    thread.join()
```

* I/O bound

```python
#!/usr/bin/python
while True:
    f = open('./io-test', 'w')
    f.write('TEST')
    f.close()
```

* /proc/sched_debug 파일의 nr_running, runnable tasks 항목에서는 각 CPU에 할당된 프로세스 수와 프로세스의 PID 등의 정보를 확인할 수 있다.



## 4장 free 명령이 숨기고 있는 것들

* Mem 행의 열 설명
  * total : 전체 메모리 크기
  * used : 사용중인 메모리 크기
  * free : 시스템에서 사용하고 있지 않은 메모리 크기
  * shared : 프로세스 사이에 공유하는 메모리 크기
  * buff/cache : 커널에 시스템의 I/O 성능 향상을 위해 사용하는 버퍼 캐시 및 페이지 캐시 영역의 크기. 메모리가 부족할 경우 커널이 자동 반환함.
  * available : free와 buff/cache 메모리에서 실제 사용가능한 메모리 크기. 해지 가능한 buff/cache 영역의 크기가 포함되어있음



### buffers와 cached 영역

* Buffer Cache : 파일 시스템의 메타 데이터 등을 저장하고 있는 캐시
* Page Cache : 파일의 내용을 저장하고 있는 캐시



### /proc/meminfo

* free에 보이는 정보는 이 파일의 일부
* Active 영역은 비교적 최근 참조된 메모리 영역을 의미
* Inactive 영역은 비교적 참조된지 오래되어 swap 영역으로 이동될 수 있는 영역의 의미함
* Dirty 는 I/O 쓰기 요청이 발생했을 때 실제 블록 디바이스의 블록에 씌어줘야할 영역



### slab

* slab은 커널이 직접 사용하는 영역. dentry cache, inode cache 등이 포함됨.
  * dentry cache : 디렉터리의 계층 관계를 저장해 두는 캐시. (ls 만 해도 늘어난다.)
  * inode cache : 파일의 inode 정보를 저장해두는 캐시
  * 파일에 자주 접근하고 디렉토리의 생성/삭제가 빈번한 프로그램이 있다면 slab 메모리가 높아질 수 있다.
* SReclaimable : slab 영역 중 재사용될 수 있는 영역. 메모리 부족시 해제될 수 있는 영역
* SUnreclaim : slab 영역 중 재사용될 수 없는 영역. 해제불가.
* `slabtop` 명령으로 사용 중인 slab 현황 확인 가능
* 메모리 누수가 발생할 때 free 명령어로 확인한 메모리 사용량과 프로세스들이 사용중인 메모리의 합계에 큰 차이가 발생한다면 커널이 사용하는 slab 영역도 의심해봐야한다.



## 5장 swap, 메모리 증설의 포인트

* 메모리 사용량이 부족할 경우 시스템이 안정적으로 운영될 수 있도록 비상용으로 디스크의 일부분을 메모리처럼 사용할 수 있도록 확보해둔 영역
* `free` 명령어로 확인 가능
* 
* swap 영역을 사용한다는 것 자체가 시스템 메모리가 부족할 수 있다는 의미임. 사용하는 것이 확인되면 이어서 어떤 프로세스가 swap을 사용하는지를 살펴봐야함



### 프로세스별 swap 영역 확인

* `/proc/<pid>/smaps` 파일은 프로세스별 메모리 정보를 담고 있고 이 파일에서는 프로세스가 사용하는 메모리 영역 별로 swap 영역을 사용한느지 확인할 수 있음
* `/proc/<pid>/status` 파일은 프로세스의 전체 swap 영역 정보를 볼 수 있음. (`VmSwap` 정보)
* `smem` 이라는 유틸리티로 각 프로세스별 메모리 사용 현황을 확인할 수 있는데 여기서 swap 정보도 나오기 때문에 프로세스별 swap 사용량을 파악하기 용이함
  * 이것도 `/proc/<pid>` 를 활용하는 것

### 관련 커널 파라미터

* vm.swappiness : 메모리를 재할당할 때 swap을 사용하게 할지 페이지 캐시를 해제하게 할지의 비율을 조절
* vm.vfs_cache_pressure : 메모리(캐시)를 재할당한다고 결정했을 때 PageCache를 더 많이 해제할지 아니면 디렉토리나 inode 캐시를 더 많이 해제할지를 결정

### 메모리 증설의 포인트

* swap 사용량이 있다고 항상 메모리를 증설하는 것이 답은 아니다. 메모리 누수가 있는 경우는 swap 사용시점을 뒤로 늦출뿐 근본 해결은 아니기 때문.
* 메모리 누수가 있는 경우는 `pmap` 명령어, `/proc/<pid>/smaps` 등으로 메모리 누수 포인트를 잡아낸뒤 gdb 등의 도구를 사용해 메모리 덤프를 떠서 메모리 누수를 근본해결해야한다.
* 누수가 아닌 특정한 이벤트로 인해 메모리 사용량이 급격히 늘어난 경우에는 swap은 일종의 방어책이 된다. 1회성 이벤트가 아니라 지속적으로 발생할 것으로 판단된다면 메모리 증설을 검토한다.



### case study - 메모리 누수 잡기

```c
// 1초에 1MB씩 계속 할당하고 해제하지 않는 프로그램
#include <stdio.h>
#include <stdlib.h>
#include <sys/time.h>

#define MEGABYTE 1024*1024

int main() {
        struct timeval tv;
        char *current_data;

        while (1) {
                gettimeofday(&tv, NULL);
                current_data = (char *) malloc(MEGABYTE);
                sprintf(current_data, "%d", tv.tv_usec);
                printf("current_data = %s\n", current_data);
                sleep(1);
        }

        exit(0);
}
```

* pid가 6290 이라고 가정하면 원인분석과정은 아래와 같다.

* 메모리 덤프 과정

  * `ps aux | grep -i 6290` 으로 RSS 증가 확인

  * `pmap 6290` 으로 특정 메모리 영역의 증가 확인 (테스트에서는 `[ anon ]` 이라고 표기된 부분 중 한 곳이 계속 증가하는 것을 확인)

  * cat `/proc/6290/smaps | grep -B5 -i size` 로 확인해보면 아래 처럼 사용량이 지속적으로 증가하는 메모리 영역을 확인할 수 있음

    ```
    ...
    7f32f3423000-7f32f5342000 rw-p 00000000 00:00 0
    Size:              31868 kB
    KernelPageSize:        4 kB
    ...
    ```

  * `gdb -p 6290` 으로 디버깅을 시작하고

  * gdb 상에서 `dump memory /home/kwon/test/memory_dump 0x7f32f3423000 0x7f32f5342000` 의 형식으로 메모리 덤프를 뜨고

  * strings 명령어로 데이터를 살펴보면 `strings memory_dump` 

    ```
    ...
    405365
    405125
    404895
    404669
    404427
    404192
    403917
    ...
    ```

  * 위와 같이 어떤 데이터가 메모리에 쓰이고 있는지 살펴볼 수 있다. 이를 통해 메모리 누수 포인트를 특정짓고 근본원인을 해결할 수 있다.



## 6장 NUMA, 메모리 관리의 새로운 세계

* NUMA (Non-Uniform Memory Access) 는 멀티프로세스 환경에서의 불균형 메모리 접근 아키텍처이다.
* 기존 UMA는 모든 프로세스가 공용 BUS를 이용해 메모리 접근하므로 0번 소켓의 CPU가 메모리에 접근하는 동안 1번 소켓에 있는 CPU는 메모리에 접근할 수가 없었음
* NUMA 에서는 각 CPU가 자신의 로컬 메모리를 가지므로 로컬 메모리로의 접근은 복수의 CPU가 동시에 접근할 수 있음. 반면 자신의 로컬 메모리를 초과하는 양은 다른 메모리에 접근하게 되는데 이는 로컬 메모리대비 느리므로 NUMA 에서는 최대한 로컬 메모리를 이용하도록 설정하는 것이 포인트임.
* CPU와 로컬 메모리를 합쳐 노드라고 부르며 노드 내에서의 메모리 접근은 로컬 액세스, 다른 노드의 메모리에 접근하는 것을 리모터 액세스라고 부름



### 관련 명령어

* `numactl --show` : NUMA 정책확인
* `numactl -H`
* `numastat` : NUMA 환경에서 현재 시스템에 할당된 메모리의 상태 확인



### 성능 튜닝 포인트

* bind, preferred, interleave 등의 NUMA 정책 결정
* numad 를 통한 프로세스들의 메모리 할당 최적화
* vm.zone_reclaim_mode 파라미터 조절을 통한 메모리 부족에 대한 대응 조절



## 7장 TIME_WAIT 소켓이 서비스에 미치는 영향

### TCP 통신 과정

* TCP 연결 맺기 : 3 way handshake
  * 연결을 맺고자 하는 측에서 먼저 SYN을 보냄

![99087C405C18E3CD28](./imgs/DevOps와SE를위한리눅스커널이야기/99087C405C18E3CD28.png)



* TCP 연결 끊기 : 4way handshake
  * 연결을 끊고자 하는 측에서 먼저 FIN을 보냄
  * 먼저 연결을 끊는 쪽을 active closer, 반대측을 passive closer 라고 함.

![99229C485C1D90C038](imgs/DevOps와SE를위한리눅스커널이야기/99229C485C1D90C038.png)



* 리눅스에서 80포트 tcp dump 만들기

  * `tcpdump -A -vvv -nn port 80 -w server_dump.pcap`

  * pcap은 wireshark 에서 바로 열어볼 수 있음

  * `curl google.com` 로 80 포트로 패킷을 만들어보고 열어보자 (참고로 curl은 내가 먼저 연결을 끊는다 (내PC가 active closer))

  * `telnet google.com 80` 으로 열고 아래 명령으로 해보면 구글서버가 먼저 끊는다. (구글서버가 active closer)

    ```
    GET / HTTP/1.1
    Host:google.com
    
    ```

    



### TIME_WAIT 소켓의 문제점

* 4 way handshake 과정 중 active closer 측에 TIME_WAIT 소켓이 생성됨. 상황에 따라 클라이언트 측에 생길 수도 있고 서버 측에 생길 수도 있음

* TIME_WAIT 소켓 확인하기
  * `netstat -anpo | grep -i time_wait`

* TIME_WAIT 소켓이 많아질 때의 문제점
  1. 로컬 포트 고갈에 따른 애플리케이션 타임아웃 발생
     - `net.ipv4.ip_local_port_range` 파라미터로 외부와 통신하기 위해 필요한 로컬 포트의 범위를 지정하는데 이 범위의 값을 모두 사용하게 되면 로컬포트 할당이 불가능하게 되므로 외부와 통신을 못하게 되고 결과적으로 애플리케이션의 타임아웃으로 이어지게된다.
  2. 잦은 TCP 연결 맺기/끊기로 인해 서비스의 응답 속도 저하가 발생할 수 있음



### 클라이언트에서의 TIME_WAIT

* HTTP 기반 서비스는 대부분 서버가 먼저 연결을 끊어서 서버에만 TIME_WAIT가 생긴다고 생각할 수 있으나 어디까지나 연결을 먼저 끊는 active closer 측에 생기는 것이므로 서버/클라이언트 여부는 중요하지 않음
* TIME_WAIT 생성 테스트

```bash
curl http://google.com
netstat -anpo | grep 80
```

```
위의 curl을 많이 해보면 아래와 같이 TIME_WAIT 소켓이 늘어나게 된다.
기본적으로 1분간 유지되는데 이 시간동안 net.ipv4.ip_local_port_range 범위 내의 포트를 모두 사용하게 되면 더 이상 소켓을 만들 수 없게된다.

client:local_port1 google.com:80
client:local_port2 google.com:80
...
client:local_portN google.com:80
```



#### 해결방법1 : net.ipv4.tcp_tw_reuse

* 나갈 때 사용하는 로컬 포트에서 TIME_WAIT 상태의 소켓을 즉시 재사용할 수 있도록 함
* 1로 설정하면 키는 것
* net.ipv4.tcp_timestamps 값이 1이어야함.



#### 해결방법2 : ConnectionPool

* 커넥션풀을 사용해 무분별하게 소켓이 생성되는 것을 막아 근본적으로 TIME_WAIT 상태의 소켓이 만들어지는 것을 줄이는 방법. 소켓 생성/삭제가 줄어들기 때문에 서비스의 응답 속도 향상도 기대할 수 있음.



### 서버 입장에서의 TIME_WAIT 소켓

* nginx 에서 keepalive_timeout을 0으로 설정하고 올리고 클라이언트에서 curl을 날려보면 서버쪽이 먼저 연결을 끊기 때문에 서버쪽에 TIME_WAIT 소켓이 생긴다.



#### 하면 안 되는 해결책 : net.ipv4.tcp_tw_recycle

* 클라이언트측 해결책으로 쓰던 net.ipv4.tcp_tw_reuse 와 달리 서버 입장에서 TIME_WAIT 소켓을 빠르게 재사용할 수 있도록 해줌
* 가장 마지막에 해당 소켓으로부터 들어온 timestamp 를 TIME_WAIT 소켓 정리에 사용한다. 즉 같은 클라이언트 IP에서는 timestamp가 증가할 것을 전제로 만들어져있다. 하지만 NAT IP 등의 환경에서는 동일 IP일지라도 서로 다른 클라이언트일 수 있고 이 경우 timestamp가 더 작은 값이 나중에 들어오는 것도 가능한데 이러면 tcp_tw_recycle이 켜진 경우 패킷을 드랍시켜버린다. 그러므로 실제 운영환경에서 tcp_tw_recylce 옵션은 절대로 사용해서는 안 된다.
* (내가 확인) ubuntu20.04 기준으로 해당 파라미터는 아예 없었다. 이런 문제로 deprecated 된걸까..
* 1로 설정하면 키는 것



#### 해결책 : keepalive 사용하기

* keepalive란 한 번 맺은 세션을 요청히 끝나도 일정시간 유지해주는 기능
* keepalive 기능을 통해 생성된 소켓이 오랫동안 유지되도록 함으로써 TIME_WAIT 소켓을 줄일 수 있다. 불필요한 TCP 연결 맺기/끊기 과정이 줄어드므로 응답 속도 향상도 기대할 수 있다.
* ex) nginx에서 keepalive를 30초로 설정하고 telnet으로 요청테스트를 해보면 마지막 요청으로부터 30초 후 끊긴다. 즉, 최초 연결 후 30초가 아니라 마지막 통신 후 30초.



### TIME_WAIT 의 존재이유

* time_wait 소켓은 여러 문제를 발생시키는 원인처럼 보이기도 하지만 정상적인 TCP 연결 해제를 위해 반드시 필요함.
* 예를 들면 TCP 연결 종료 과정에서 패킷이 유실되더라도 TIME_WAIT 가 있기 때문에 종료 과정중의 ACK이 소실되더라도 재전송된 FIN 등을 처리하여 소켓이 정상적으로 종료될 수 있음. 만약 TIME_WAIT 시간이 매우 짧다면 패킷 유실로인해 불완전하게 종료된 소켓이 발생할 수 있음.



### Case Study - nginx upstream에서 발생하는 TIME_WAIT

* nginx - play framework 로 구성된 환경 (웹서버 -> 앱서버)이라면 nginx -> play 구간에 keepalive를 적용하는 것이 적절하다.
* 첫 번째로 로컬 포트 고갈문제가 발생할 수 있기 때문이다. 다만 이 문제는 tw_resue 커널 파라미터를 사용해도 해결가능하다.
* 두 번째로 잦은 TCP 연결/해제로 인한 성능 저하가 발생하기 때문이다. keepalive를 적용하여 성능을 향상할 수 있다.



## 8장 TCP Keepalive를 이용한 세션 유지

### TCP Keepalive란?

* 통신할 때마다 매번 3-way handshake 를 진행하는 것은 낭비이므로 통신이 계속되는 상황이라면 처음 만든 세션을 계속 사용하도록 하는 기능
* 동작원리1 : 3-way handshake로 연결을 맺고 최초 의도했던 요청/응답을 주고받은 뒤 바로 끊지 않고 keepalive 를 위한 패킷을 하나 보내고 받은 측은 ack을 보내어 세션을 유지한다. (서버와 클라이언트 중 어느쪽이든 이 기능을 사용하면 세션은 유지됨)
* 동작확인 : netstat 으로 확인했을 때 마지막 열에 keepalive timer가 표시된다. 

```bash
# 아래는 ssh 연결로 확인한것. sshd 데몬이 사용하는 소켓에 keepalive 옵션이 켜져있음을 알 수 있다.

kwon@DESKTOP-0U32D8C:~/md-study$ sudo netstat -anpo | grep 2222
tcp        0      0 172.20.2.11:33684       192.168.1.194:2222      ESTABLISHED 2026/ssh             keepalive (7168.51/0/0)
```

* 동작원리2 : keepalive timer가 다 되면 연결이 살아있는지 확인하는 작은 패킷을 보낸다.
* 사용방법1 : 소켓을 생성할 때 소켓 옵션을 설정해야됨. setsockopt() 라는 함수를 사용하며 옵션 중에 SO_KEEPALIVE 를 사용하면 keepalive를 사용하는 것
* 사용방법2 : 소켓을 직접 만들 때 그렇다는거고 보통 애플리케이션이 제공하는 tcp keepalive 옵션을 켜면된다.
* 여담으로 google 등에 웹요청을 날려도 Keep-alive 관련 헤더는 응답으로 돌아오지 않는데 이는 HTTP/1.1 부터는 기본 동작이 keep-alive라 그렇다. (생략 가능)



### TCP Keepalive 파라미터

* 커널 파라미터는 3개
* net.ipv4.tcp_keepalive_time : keepalive 소켓의 유지 시간. 애플리케이션에서 세팅할 경우에는 그 값이 우선이고 그런 설정이 없을 때 커널 파라미터 값이 사용됨 
* net.ipv4.tcp_keepalive_probes : keepalive 패킷을 보낼 최대 전송 횟수. keepalive 패킷 전달 자체가 실패할 수 있으므로 그에 대비한 값임.
* net.ipv4.tcp_keepalive_intvl : probes와 함께 keepalive 패킷 전달이 실패할 경우의 재전송을 위한 값. 최초 keepalive_time 초 동안 기다린 후 keepalive 확인 패킷을 보내고 최초 패킷에 대한 응답이 돌아오지 않으면 keepalive_intvl 간격으로 keepalive_probes 번의 패킷을 더 보냄. 그 후에도 응답이 안 오면 연결을 끊음.
* 두 종단 간의 연결을 끊기 위해서는 FIN 패킷이 필요함. 하지만 실제 운영환경에서는 FIN을 주고 받지 못하고 끊어지는 상황이 있을 수 있음. 예를 들면 서버가 연결되어 있는 스위치에 장애가 생기면 두 종단 간 연결이 끊어진 것이지만 FIN을 전달할 방법이 없어 계속해서 연결된 것처럼 남아있게 됨. 이 때 keepalive를 사용한 소켓의 경우 재전송 절차를 밟은뒤에 그래도 응답이 없으면 커널이 해당 소켓을 정리함.



### TCP Keepalive와 좀비 커넥션

* keepalive의 첫번째 기능은 두 종단 간의 연결을 유지함으로써 불필요한 tcp handshake를 줄여 전체적인 서비스 품질을 높이는 것.
* 더 중요한 두번째 기능은 좀비 커넥션 (잘못된 커넥션 유지) 이라 부르는 소켓을 방지하는 기능
* 애플리케이션레벨에서 직접 keepalive를 관리할 수도 있겠으나 커널에서 제공하는 기능만으로도 좀비 커넥션을 관리할 수 있음



### TCP Keepalive와 HTTP Keepalive

* apache나 nginx 에도 keepalive timeout 옵션이 있는데 차이점은 무엇일까
* 최대 유지 시간1 : 만약 둘다 60초로 설정한다면
  * TCP Keepalive : 60초 간격으로 연결이 유지되는지 확인하고 응답을 받았으면 계쏙 연결을 유지함
  * HTTP Keepalive : 60초 동안 소켓을 유지하고 60초가 지난 후에도 요청이 없다면 연결을 끊음
* 최대 유지 시간2 : TCP Keepalive는 30초, Apache Keepalive는 60초
  * netstat 으로 확인해보면 keepalive 타이머는 30초로 설정된다. 2번 타이머가 돌아 60초가 되면 apache 설정에 따라 서버가 먼저 클라이언트와의 연결을 종료한다.
* 최대 유지 시간3 : TCP Keepalive는 120초, Apache Keepalive는 60초
  * keepalive 타이머는 120초로 설정되지만 60초가 되면 apache가 먼저 소켓을 정리해버린다.
* 결국 keepalive 패킷 전송은 tcp keepalive 옵션을 따르지만 실제 종료 타이밍은 애플리케이션이 정한 설정값을 따르게 됨을 알 수 있다.



### Case Study - MQ 서버와 로드 밸런서

* L4 뒤에 MQ서버 두 대가 있는 상황을 가정해보자
* 그냥 운영하면 MQ서버에는 사용하지 않는 소켓이 많이 쌓여있고 (established) 정작 클라이언트에는 서버에 열려있는 소켓이 존재하지 않기도 한다. 또 간헐적으로 클라이언트가 MQ서버로 접근하다가 타임아웃이 나기도한다.
* 문제의 원인은 L4의 Idle timeout. 로드 밸런서는 클라이언트와 서버의 소켓 연결이 맺어지면 해당 세션을 세션 테이블에 기억함. 이를 이용해 클라이언트가 다음 요청을 보냈을 때도 같은 서버로 패킷을 보낼 수 있는 것. 물론 이 세션 테이블은 무한한 공간이 아니므로 일정한 기준으로 비워줄 필요가 있고 이 때 Idle timeout을 사용해 해당 시간이 지날때까지 패킷이 흐르지 않은 세션을 세션 테이블에서 지워짐. 근데 지웠다는걸 누군가에게 알리지는 않으므로 클라이언트와 서버는 이를 알아차릴 수 없음.
* Idle timeout이 지나 세션이 제거되어도 클라이언트를 이를 모름. 이 상태에서 클라이언트가 패킷을 보내면 로드 밸런서가 MQ1, MQ2 어디로 보낼지 알 수 없음. 원래 맺어진 서버가 MQ1이었고 운이 좋아 MQ1으로 가면 문제가 없음. 근데 MQ2로 갈 경우 MQ2 입장에서는 TCP Handshake를 맺지 않은 요청 패킷이 들어오므로 비정상 요청으로 보고 RST 패킷을 보냄. 클라이언트는 RST 패킷을 받았으니 TCP Handshake를 맺고 다시 요청을 보냄. 이 때 소요되는 시간이 애플리케이션에서 설정한 타임아웃 임계치를 넘어가면 Timeout Exception 을 경험하게 됨. 거기다 MQ1 입장에서는 클라이언트의 상황을 알 수 없으므로 MQ1에는 좀비 커넥션이 쌓이게 됨.
* 해결방법 : 로드밸런서의 Idle timeout이 120초라면 tcp_keepalive_time을 60초, tcp_keepalive_probs를 3, tcp_keepalive_intvl를 10초로 설정하면 120초 이내에 미사용 소켓을 서버에서 정리할 수 있으므로 서버 측에 좀비 커넥션이 생기지 않게됨
  * 이 때 RabbitMQ 의 TCP Keepalive 옵션을 켜야함. 켜지 않으면 소켓 자체가 SO_KEEPALIVE 옵션이 적용되지 않으므로 아무 의미가 없음.



> DSR (Direct Server Return) : 로드 밸런서 환경에서 서버의 응답 패킷이 로드 밸런서를 거치지 않고 클라이언트에게 직접 전달되는 구조
>
> Inline 구조 : 서버로의 요청과 서버에서의 응답이 모두 로드 밸런서를 거치는 구조



## 9장 TCP 재전송과 타임아웃

* TCP는 자신이 보낸 데이터 대해 상대방이 받았다는 응답 패킷을 받아야 통신이 정상적으로 이루어졌다고 판단함. 응답 패킷을 받지 못하면 패킷이 유실되었다고 보고 재전송을 시도함.
* 애플리케이션 입장에서는 이러한 재전송이 실패하여 TCP 타임아웃을 겪을 수 있음.
* 재전송은 유실이 발생가능한 TCP의 특성상 반드시 필요한 과정이며 재전송 및 타임아웃등의 설정을 통해 서비스 품질을 향상시킬 수 있음



### TCP 재전송과 RTO

* 전달한 패킷에 대한 ACK이 일정시간 동안 오지 않으면 재전송을 해야하는데 여기서 ACK을 얼마나 기다릴지를 RTO(Retransmission Timeout) 이라함.
* 일반적인 RTO는 RTT(RoundTripTime, 두 종간 간 패킷 전송에 필요한 시간)을 기준으로 설정됨. RTT가 1초라면 적어도 1초보다는 긴 시간을 RTO로 설정해야 패킷 유실을 판단하고 재전송을 시도할 수 있음.
* Init RTO는 두 종단 간 최초 연결을 맺는 TCP Handshake 가 발생할 때의 첫 번째 SYN 패킷에 대한 RTO 를 의미함. 첫 번째 패킷을 보낼 때는 당연히 RTT를 알 방법이 없으므로 OS에서 미리 설정해둔 시간을 사용하고 이를 Init RTO라 함. 리눅스라면 보통 1초.

```bash
# 현재 연결된 소켓 상세 정보를 볼 수 있음.
ss -i

# rto, rtt 값을 확인할 수 있음
# rto는 210이고 rtt는 7.553이고 편차가 11.831 이라는 의미
# 즉 210ms 동안 ACK을 못 받으면 재전송을 시도함
tcp ESTAB 0 0 172.20.2.11:45324 192.168.1.194:2222
         cubic wscale:7,7 rto:210 rtt:7.553/11.831 ato:40 mss:1448 pmtu:1500 #... 이하 생략
```

* 재전송을 반복할 때는 RTO 값을 2배씩 늘리며 보내게됨. 200ms, 400ms, 800ms, ...
* 그렇다면 무한히 보낼 수는 없으니 횟수를 지정하여 그 이내에서 시도하고 그래도 실패하면 오류로 보고 연결을 포기하게됨.



### 재전송을 결정하는 커널 파라미터

```bash
net.ipv4.tcp_syn_retries = 6
net.ipv4.tcp_synack_retries = 5
net.ipv4.tcp_orphan_retries = 0
net.ipv4.tcp_retries1 = 3
net.ipv4.tcp_retries2 = 15
```

* 위의 파라미터들은 애플리케이션마다 독자적으로 설정할 수도 있겠지만 커널 레벨에서 일괄적으로 컨트롤하고 싶을 때 도움이될 수 있다.
* net.ipv4.tcp_syn_retries : 최초 소켓 연결시의 SYN 을 재전송할 횟수
* net.ipv4.tcp_synack_retries : 최초 소켓 연결시의 SYN+ACK 을 재전송할 횟수. 참고로 SYN+ACK을 보내고나면 SYN_RECV 상태가 되는데 DDoS 공격등의 상황에서는 최초 SYN만 왕창 오는 경우도 있으므로 이 횟수도 적당한 수준으로 줄여놔야 SYN_RECV 소켓이 쌓여 리소스 고갈이 발생하는 경우를 막을 수 있다.
* net.ipv4.tcp_orphan_retries : orphan socket 상태의 소켓들에 대한 재전송 횟수. orphan socker 이란 TCP 연결 끊기 과정에서 끊는 측에서 FIN을 보낸 뒤 FIN_WAIT1 상태가 되는데 이때부터 해당 소켓은 프로세스와의 바인딩이 해제되고 커널에 귀속됨. 이처럼 특정 프로세스에 할당되지 않고 커널에 귀속되어 정리되기를 기다리는 소켓 중 FIN_WAIT1 소켓을 orphan socket 이라 함.
  * 참고로 위의 예처럼 0이라고 재전송을 안 하는게 아니라 커널마다 다를 수 있음. 책에서 소개한 커널 2.6.32 에서는 0으로 설정해도 RTO가 120초보다 작으면 8번 시도하게 되어있음. 현실적으로 RTO가 120초보다 큰 경우는 거의 없으므로 사실상 거의 8번으로 동작하게 된다는 의미.
  * 너무 작으면 FIN이 유실된 상태에서 FIN_WAIT1 소켓이 너무 빨리 정리될 수 있으므로 상대편의 패킷이 정상적으로 닫히지 않게 될 수 있다. 최소한 TIME_WAIT가 유지되는 시간인 60초 정도가 될 수 있도록 7정도의 값을 설정하는 것을 권장.
* net.ipv4.tcp_retries1, net.ipv4.tcp_retries2 : TCP 재전송 횟수의 임계치를 정함. 1은 IP 레이어에 네트워크가 잘못되었는지 확인하도록 사인을 보내는 기준. 2는 더 이상 통신을 할 수 없다고 판단하는 기준. 각각 soft threshold, hard threshold 라고 이해할 수도 있음. 결국 두 번째 값을 설정한 횟수만큼을 넘겨야 실제 연결이 끊김.



### RTO_MIN 값 변경하기

* 커널에서는 RTO_MAX를 120초, RTO_MIN을 200ms로 정의해두었음.
* 내부의 빠른 통신에서 RTT가 아무리 작더라도 RTO가 200ms 이하로 내려갈 수 없다는 의미임.
  * 위에서 `ss` 로 살펴본 예만해도 RTT가 7.553 인데 RTO가 210 임. 정상케이스라면 26번은 패킷을 주고 받을 시간이므로 낭비일 수 있음.
* `ip route` 명령을 통해 변경 가능함.

```bash
ip route

# 결과는 아래와 같음. 모든 패킷은 eth0 네트워크 디바이스의 172.20.0.1 게이트웨이를 통해 나간다라는 의미.
default via 172.20.0.1 dev eth0
172.20.0.0/20 dev eth0 proto kernel scope link src 172.20.2.11

ip route change default via 172.20.2.11 dev eth0 rto_min 100ms
# wsl2 에서 해봤을 때는 wsl2 네트워크가 먹통이되서 재시작해야 해결됐음. 재시작하면 설정값도 날라가서 물론 적용 안 됨.
```

* 외부 통신이라면 접속 환경이 다양하므로 건들지 않는게 좋고 내부 통신이고 속도가 매우 빠를때는 낮추는 것도 고려는 할 수 있음. (내의견 : 근데 굳이 그럴 이유가 있나?)



### 애플리케이션 타임아웃

* TCP 재전송을 애플리케이션에서 체감하게 되는 것은 타임아웃.
* 애플리케이션에서 TCP 연결 관련된 타임아웃에는 크게 Connection Timeout, Read Timeout이 존재함.
* Connection Timeout : TCP Handshake 과정에서 재전송이 일어날 경우 발생하며 3s 이상 설정하는 것을 권장.
  * 최초 SYN 재전송 1초, 상대측의 SYN+ACK 재전송 1초를 합친 2초보다 큰 값으로 하여 1번의 재전송은 발생할 수 있도록하여 불필요한 타임아웃 에러를 줄이기 위함.
* Read Timeout : 맺어져 있는 세션을 통해 데이터를 요청하는 과정에서 발생하며 300ms 이상 설정하는 것을 권장.
  * 마찬가지로 한 번의 재전송은 이루어질 수 있게 설정하는 것이 좋음. RTO_MIN이 200ms 이므로 이보다는 큰 300ms 정도로 잡는 것. 물론 연결환경에 따라 더 크게 잡아야될 수도 있음. 이는 `ss` 등으로 모니터링하며 설정하면 될 것.



## 10장 dirty page가 I/O에 끼치는 영향

* 디스크 I/O가 발생할 때 매번 디스크에서 I/O가 발생하는게 아니고 성능을 높이기 위해 메모리에 일시적으로 캐싱해두고 I/O가 발생함.
* 쓰기 작업의 경우 메모리상에서만 쓰기가 발생하고 디스크에는 아직 반영되지 않은 페이지를 dirty page라 함.
* dirty page 는 언젠가는 디스크에도 쓰여야하는데 이를 page writeback, dirty page 동기화라고 함. 
* dirty page가 생길때마다 디스크에 동기화를 하려면 많은 쓰기 I/O로 인해 성능이 저하되므로 어느 정도 모아서 동기화할 필요가 있음. 어떤 조건으로 동기화를 하느냐에 따라 I/O 성능에 영향을 미치므로 I/O가 많이 발생하는 서버의 경우 dirty page 관련 성능 튜닝을 검토할 수 있음



### dirty page 관련 커널 파라미터

* vm.dirty_background_ratio = 10
  * dirty page의 크기가 전체 메모리의 10%가 되면 백그라운드로 동기화를 진행한다.
  * dirty_ratio보다 큰 값을 가질 수 없다. 크게 설정해도 커널에서 dirty_ratio보다 작은 값으로 낮추어 사용한다.
* vm.dirty_ratio = 20
  * dirty page의 크기가 전체 메모리의 20%가 되면 프로세스의 I/O 작업을 모두 멈추게 하고 동기화를 진행한다. dirty page에 대한 일종의 hard limit 개념. 
  * dirty_background_ratio 값에 의해 백그라운드로 동기화를 해도 프로세스들이 쓰는 속도가 더 빠르면 dirty page가 계속 많아질 수 있는데 dirty_ratio 값에 도달하게 되면 프로세스들의 쓰기 작업 자체를 멈춰버리고 dirty page 동기화를 우선처리한다. 이 경우 많은 성능 저하가 발생한다.
  * 최소값은 5 (커널마다 다를 수 있음)
* vm.dirty_background_bytes = 0
  * ratio와 달리 dirty page가 지정한 bytes 값에 도달하면 동기화를 진행한다.
* vm.dirty_bytes = 0
  * dirty_ratio와 동일한 역할을 한다. 비율이 아닌 bytes 단위로 동작한다.
  * 참고로 dirty_background_bytes, dirty_bytes 을 사용하는 경우 ratio 관련 설정은 무시된다. 즉, bytes 설정이 우선이다.
* vm.dirty_writeback_centisecs = 500
  * 동기화 작업을 하는 flush 커널 스레드를 몇 초 간격으로 깨울지를 결정한다. 단위가 1/100초이므로 500이면 5초다.
  * 0 이면 주기적인 동기화를 하지 않음.
* vm.dirty_expire_centisecs = 1000
  * dirty_writeback_centisecs 로 깨어난 flush 커널 스레드가 동기화에 사용하는 기준값. 단위가 1/100초이며 1000이면 dirty page로 분류된 후 30초 동안 디스크로 동기화되지 않은 페이지들을 동기화한다.



### 동기화의 종류

1. 백그라운드 동기화 : dirty_background_ratio, dirty_ratio 값등으로 발생하는 동기화. dirty page가 생길때 마다 현재까지 생성된 dirty page와 전체 메모리의 비율을 바탕으로 진행
2. 주기적인 동기화 : dirty_writeback_centisecs, dirty_expire_centisecs 값을 통해 주기적으로 flush 데몬이 깨어나 동기화를 수행
3. 명시적인 동기화 : sync, fsync 등의 명령어를 통해 명시적으로 동기화



* 백그라운드, 주기적인 동기화는 둘 다 각각 설정된 조건에 맞추어 동작한다. 예를 들어 주기적인 동기화를 자주 발생하게 설정하면 백그라운드 동기화가 발생하는 임계치까지 dirty page가 증가하지 않을 수도 있다.



### 연관명령어

* iostat : I/O 사용량을 확인할 수 있음
* `cat /proc/meminfo | grep -i dirty` : 현재 dirty page 크기 확인



### 튜닝 포인트

* 너무 자주 동기화시키면 flush 커널 스레드가 너무 자주 깨어남 ()
* 너무 늦게 동기화시키면 동기화할 dirty page가 너무 많아 dirty_ratio에 도달할 가능성이 커짐.
* 서버의 CPU, 디스크 쓰기 속도, 운용중인 어플리케이션의 종류에 따라 적절한 튜닝이 필요함
* 예를 들어 디스크 쓰기속도가 10MB/s 인 서버 A와 100MB/s 인 서버B가 있고 둘 다 매초 100MB의 쓰기가 발생하는 애플리케이션을 운용중이라면 A서버는 10MB 단위로 동기화가 발생하도록 하는 것이 유리할 것이고 (만약 100MB 단위로 동기화할 경우 동기화 속도보다 쌓이는 속도가 더 빨라 dirty_ratio에 도달하여 큰 성능 저하가 발생) B서버는 100MB 단위로 동작하게 설정해도 충분하리라 예상할 수 있음.



## 11장 I/O 작업이 지나가는 관문, I/O 스케줄러

* 사용자가 발생시키는 모든 I/O작업은 가상 파일 시스템, 로컬 파일 시스템 등의 경로를 거쳐 블록 디바이스로 전달되기 전에 I/O 스케줄러를 거치게 됨
* I/O 스케줄러는 상대적으로 속도가 느린 디스크에 대한 성능을 극대화하기 위한 프로그램
* 기본적으로 병합과 정렬이라는 두 가지 방식으로 성능 최적화를 함
* 이 중에서 정렬은 HDD와 같이 물리적인 헤드 이동이 필요했던 디스크에는 필요했지만 헤드이동 시간이 없는 SSD의 경우 정렬이 별 의미가 없고 되려 성능이 나빠질 수 있음.
* 병합은 HDD, SSD 둘 다 사용함.



### I/O 스케줄러 확인 및 변경

* `cat /sys/block/<block device>/queue/scheduler`
  * ex : `cat /sys/block/sda/queue/scheduler` 
  * cfq, deadline, noop
  * 스케줄러마다 튜닝가능한 파라미터가 다름
    * 스케줄러에 따라 `/sys/block/<block device>/queue/iosched` 경로의 파일들이 달라지는데 여기에 있는 파일들이 현재 설정된 스케줄러가 설정가능한 튜닝값들
* `echo cfq > /sys/block/<block device>/queue/scheduler`
  * echo 명령어로 변환 가능
  * 위는 cfq로 바꾸는 예제



### cfq I/O 스케줄러

* Completely Fair Queueing : 완전 공정 큐잉
* 이름대로 모든 프로세스에서 I/O 요청들이 공정하게 실행되는 특징을 가짐



### deadline I/O 스케줄러

* I/O 요청별로 완료되어야 하는 dealine 을 가지는 스케줄러
* 가능한 한 주어진 deadline을 넘기지 않도록 동작함



### noop I/O 스케줄러

* 병합만 하는 스케줄러
* SSD와 같은 플래시 디스크에 권장
* 튜닝 값 없음



### 연관 명령어

* iotop : 프로세스별로 발생시키는 I/O 작업 현황을 파악할 수 있음.
* iosnoop
  * 현재 시스템에서 발생하는 I/O 요청들의 섹터 주소로 볼 수 있어 순차 접근이 많은지 임의 접근이 많은지에 대한 I/O 워크로드 패턴을 살펴보는데 도움이 됨
  * https://github.com/brendangregg/perf-tools/blob/master/iosnoop



### 튜닝 포인트

* 모든 워크로드를 만족시키는 스케줄러는 없고 시스템의 I/O 워크로드를 파악해서 어떤 스케줄러를 사용할지 판단해야 함
* cfq : 프로세스별로 대등하게 I/O 시간이 주어져야하는 경우에 유리함. 
  * ex) 웹 서버
* dealine : 프로세스들의 I/O 요청중 몇몇 개의 프로세스가 많은 I/O 요청을 일으켜서 빨리 처리되어야 하는 경우에 적합. 또는 데이터베이스, 파일 서버에 적합
* noop : 플래시 메모리 기반 디스크에 적합.



## 12장 애플리케이션 성능 측정과 튜닝

* 예제 프로그램으로 성능 최적화 테스트



### 테스트 python 코드

```python
import redis
import time
from flask import Flask
app = Flask(__name__)

# 1. 매번 새로운 redis 연결 생성
@app.route("/test/<key>")
def testApp(key):
    r = redis.StrictRedis(host='localhost', port=6379, db=2)
    r.set(key, time.time())

    return r.get(key)

# 2. redis 연결에 커넥션 풀 사용
pool = redis.ConnectionPool(host='localhost', port=6379, db=0)
@app.route("/test2/<key>")
def testApp2(key):
    r = redis.Redis(connection_pool=pool)
    r.set(key, time.time())

    return r.get(key)

if __name__ == "__main__":
    app.run(host='0.0.0.0')
```



### 성능 측정

* 환경 : ubuntu 20.04 on wsl (4core, 6GB) / (호스트 pc : 하스웰 4670 4core)
* 성능측정 도구 : siege
  * https://www.joedog.org/siege-home/
  * apt로도 설치가능했음
  * 100명이 30초간 지속적으로 트래픽을 발생시키는 테스트
* redis

```bash
docker run -it --rm -p 6379:6379 redis
# tcp-keepalive 옵션은 기본으로 적용됨 (300초)
```

* 하나의 wsl에서 redis, 테스트 앱서버, siege를 다 돌린 것임을 감안하여 참고
* python dependency

```bash
pip install flask
pip install redis
pip install gunicorn
pip install eventlet
```



### 측정1 : 순수 flask

* flask 내장 앱서버를 그대로 사용하므로 싱글 스레드 기반으로 동작

```
-- flask
-- python3 performance.py
-- redis 연결 : 매번 새로운 커넥션 생성
-- siege -c 100 -b -t30s http://localhost:5000/test/1
Transactions:                   9096 hits
Availability:                 100.00 %
Elapsed time:                  29.21 secs
Data transferred:               0.15 MB
Response time:                  0.32 secs
Transaction rate:             311.40 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   99.35
Successful transactions:        9096
Failed transactions:               0
Longest transaction:            0.75
Shortest transaction:           0.03
```



### 측정2 : flask + gunicorn

* cpu 활용을 최대화하기 위해 gunicorn 적용 후 복수의 워커 설정
* 최소 서버의 cpu 코어수만큼 설정하고 부하에 따라 워커의 개수를 조정할 수 있음
* `ss -s` 로 확인해보면 많은 time_wait 소켓이 생김
  * `netstat -anpo | grep -ic time_wait`

```
-- flask + gunicorn (4 workers, sync worker)
-- gunicorn -w 4 -b 0.0.0.0:5000 performance:app
-- redis 연결 : 매번 새로운 커넥션 생성
-- siege -c 100 -b -t30s http://localhost:5000/test/1
The server is now under siege...
Lifting the server siege...
Transactions:                  19043 hits
Availability:                 100.00 %
Elapsed time:                  29.72 secs
Data transferred:               0.32 MB
Response time:                  0.16 secs
Transaction rate:             640.75 trans/sec
Throughput:                     0.01 MB/sec
Concurrency:                   99.37
Successful transactions:       19043
Failed transactions:               0
Longest transaction:            0.45
Shortest transaction:           0.05
```



### 측정3 : flask + gunicorn + redis connection pool

* app -> redis 연결을 할 때마다 매번 커넥션을 맺지 않고 connection pool을 사용하는 url을 호출함
* 한 번 맺은 세션을 계속 사용하므로 TCP Handshake에 대한 오버헤드가 줄어들어 성능이 개선됨
* 그러나 여전히 앱서버쪽에는 time_wait 소켓이 많이 생김

```
-- flask + gunicron (4 workers, sync worker)
-- gunicorn -w 4 -b 0.0.0.0:5000 performance:app
-- redis 연결 : connection pool 적용
-- siege -c 100 -b -t30s http://localhost:5000/test2/1
Transactions:                  45869 hits
Availability:                 100.00 %
Elapsed time:                  29.15 secs
Data transferred:               0.78 MB
Response time:                  0.06 secs
Transaction rate:            1573.55 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                   99.73
Successful transactions:       45869
Failed transactions:               0
Longest transaction:            0.19
Shortest transaction:           0.03
```



### 측정4 : flask + gunicorn (keep-alive, eventlet worker) + redis connection pool

* 앱서버의 time_wait 소켓을 없애기 위해 gunicorn 에 keepalive 를 설정함
* gunicorn의 디폴트 worker인 sync는 keepalive를 지원하지 않으므로 eventlet worker를 적용함
* keepalive 설정과 비동기 worker인 eventlet 이 성능을 향상시킨 것

```
-- flask + gunicron (4 workers, keep-alive, eventlet worker)
-- gunicorn -w 4 -b 0.0.0.0:5000 performance:app --keep-alive 10 -k eventlet
-- redis 연결 : connection pool 적용

-- 테스트전 ~/.siege/siege.conf 에서 
-- conncection 값을 close -> keep-alive 로 변경

-- siege -c 100 -b -t30s -H "Connection: keep-alive" http://localhost:5000/test2/1
Transactions:                  55468 hits
Availability:                 100.00 %
Elapsed time:                  29.34 secs
Data transferred:               0.94 MB
Response time:                  0.05 secs
Transaction rate:            1890.52 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                   99.70
Successful transactions:       55468
Failed transactions:               0
Longest transaction:            0.41
Shortest transaction:           0.00
```

* 측정3과 같이 gunicorn을 keepalive를 지원하지 않는 상태로 띄워놓고 siege는 keep-alive 지원으로 테스트해보면 아래와 같이 금새 테스트가 실패한다.
* error 메시지로 보아 RST packet 을 받아 소켓이 비정상종료되었음을 알 수 있다. 즉 서버 입장에서는 이미 연결이 종료된 소켓인데 클라이언트는 keep-alive 일거라 생각하고 계속 요청을 보내니 서버로부터 RST 패킷을 받은 것.

```
...
[error] socket: read error Connection reset by peer sock.c:539: Connection reset by peer
...
Transactions:                   1123 hits
Availability:                  52.31 %
Elapsed time:                   1.16 secs
Data transferred:               0.02 MB
Response time:                  0.10 secs
Transaction rate:             968.10 trans/sec
Throughput:                     0.02 MB/sec
Concurrency:                   92.42
Successful transactions:        1123
Failed transactions:            1024
Longest transaction:            0.14
Shortest transaction:           0.0
```



### nginx 적용

* 통상적으로 gunicorn과 같은 앱서버로 직접 사용자 요청을 받지 않는다.
* 앞에 nginx, apache 등의 웹서버를 두는 것이 보통
* 이유는 보안 설정, virutal host 등의 다양한 라우팅 정책 설정 등에서 유리하기 때문
  * 내의견) cdn이 없다면 정적자원 배포역할을 해줄 수도 있고 reverse proxy 역할도 할 수 있음



* 예제에서는 nginx를 디폴트 상태로 gunicorn과 합치고 사용자수를 늘리다보면 nginx -> gunicorn 으로의 전달에서 로컬 포트가 고갈되어 문제가 생길 수 있는 것을 보여줌
* 첫번째 해결방법은 `net.ipv4.tcp_tw_resue` 를 1로 설정하여 time_wait 상태의 로컬포트를 즉시 재사용할 수 있게 하는 것
* 더 근본적인 해결방법은 nginx 설정을 조정하여 nginx -> gunicorn 구간을 keepalive 모드로 동작시키는 것

```
# nginx.conf

...
upstream gunicorn {
  server my.gunicorn.com:5000;
  keepalive 1024;
}
```

* 로컬포트와는 별도로 동시 접속자수가 계속 늘어나면 nginx의 worker 수 자체가 모자라서 에러가 발생할 수도 있는데 이 때는 worker 설정을 조정해야함

```
[alert] 7431#0: 1024 worker_connections are not enough
```

```
# nginx. conf

...
events {
  use epoll;
  worker_connections 10240;
  multi_accept on;
}
```

* worker의 개수를 늘렸고
* select 방식의 이벤트 처리 모듈에서 epoll 방식으로 변경함. 일반적으로 epoll 방식이 성능이 좋다고함.



### 정리

* CPU, 소켓 등의 서버 자원은 여유로운데 성능이 만족스럽지 않다면 애플리케이션 코드의 성능 최적화도 중요하지만 서버 설정도 함께 확인해야한다.
* 물론 다양한 시나리오의 성능 테스트로 정상 동작함을 검증해야한다.



* CPU 자원을 제대로 활용하고 있는지 확인한다. 애플리케이션의 워커수를 조정하여 최적의 값을 찾아야한다. 최소한 cpu 코어 수 만큼은 워커 개수를 설정하자.
* 다수의 time_wait 소켓이 발견된다는 것은 tcp handshake가 빈번히 발생함을 의미한다. 어느 구간에서 time_wait 소켓이 발생하는지 확인하고 keepalive 등을 적용해 성능을 높일 수 있다.

* 다른 서비스들과 연동하는 경우 keepalive 옵션 등을 이용해 연결을 만들어 놓고 커넥션 풀 방식으로 사용하면 성능을 높일 수 있다.
* 이외에도 시스템 리소스는 넉넉한데 서비스 응답 속도가 느리거나 장애가 발생한다면 애플리케이션의 워커 설정 등 애플리케이션이 제공하는 설정을 점검하자. 잘못된 설정을 무시하고 무턱대고 서버를 늘린다고 해도 금방 문제가 재발할 가능성이 높다.



## Appendix B strace를 통한 애플리케이션 분석

* strace 로 프로세스의 시스템 콜 호출 과정을 추적할 수 있고 이를 통해 대략적인 동작 원리를 파악할 수 있음
* 주요 옵션
  * -s : 출력 결과물의 길이 설정
  * -f : 멀티 스레드, 멀티 프로세스 방식으로 동작할 때 그 자식 스레드 혹은 프로세스에 대한 추적을 가능하게 함
  * -t : 타임 스탬프를 찍어줌. 주로 -tt와 같이 microsecond 단위로 기록하도록 사용함.
  * -T : 각 시스템 콜의 호출 시간 기록
  * -o : 출력 결과를 파일로 저장
  * -p : 이미 동작 중인 프로세스를 추적할 때 사용. 프로세스 동작에 부하를 줄 수 있으므로 주의 필요

## Appendix C tcpdump와 와이어샤크를 통한 TCP 패킷 분석

* tcpdump로 네트워크 패킷을 수집할 수 있고 wireshark를 통해 GUI로 확인할 수 있다.

```
tcpdump -nn -vvv -A -w dump.pcap
```

* -nn : 프로토콜과 포트를 단어로 변환하지 않고 원래 값 그대로 출력. (ex. 80 port 패킷이면 http로 변환하지않고 80 포트 그대로 출력)
* -vvv : 패킷의 상세 내용 출력. dump 사이즈는 커지지만 문제 확인에 도움이 됨.
* -A : 패킷 수집 결과를 ASCII 형태로 출력. 평문으로 데이터를 주고받는 경우 내용을 바로 볼 수 있어 문제 파악이 도움이 됨.
* -w : 파일로 저장.
* -Z : tcpdump는 기본적으로 tcpdump 기본 계정으로 동작하는데 이 옵션으로 특정 계정으로 실행할 수 있음. 패킷 수집 내용을 저장하거나할 때 권한 이슈가 있다면 적용 검토.
* host : 특정 IP가 포함된 패킷 필터링
* port : 특정 port가 포함된 패킷 필터링

```
tcpdump -nn -vvv -A -w dump.pcap host 192.168.1.194 and port 50000
```

* -G : 지정한 초마다 pcap 파일을 새로 생성함. 오랫동안 덤프를 떠야할때 단일 덤프파일이 너무 커지는 것이 우려될 경우 이 옵션을 적용할 수 있음. (일종의 rotate 기능임) 이 때 -w는 적절한 표현식을 사용하여 파일명이 중복되지 않도록 해야함. (중복되면 예전 dump는 날라가는 것)
  * (201114) 아래 예제는 wsl 오류인지 `tcpdump: Couldn't change ownership of savefile` 라는 에러가 발생하면서 되지 않았음. 뒤에 `-Z root` 로 줘도 마찬가지였음. 표현식 없으면 -G 자체는 동작하는걸로봐서 -G와 -w의 표현식이 함께 쓰일 경우의 권한 오류가 있는듯함.

```bash
tcpdump -nn -vvv -A -G 10 -w dump_${hostname}_%Y%m%d-%H%D%S.pcap -Z root
```



### wireshark 로 보는 팁

* ip, port, 프로토콜 등으로 필터링해서 볼 수 있다. 필터 적용 필드 왼쪽의 북마크 버튼을 눌러보면 다양한 필터 예제가 나오니 참고
* 두 개 이상 조건을 섞을 때는 and, or 사용
* 패킷 우클릭 > Follow > TCP Stream을 눌러보면 해당 패킷의 시작부터 종료까지의 전체 흐름만 필터링해서 볼 수 있다. 예를 들어 RST 패킷이 나타났다면 Follow 기능을 사용하여 최초 패킷 생성시점부터 RST 패킷까지의 일련의 흐름을 볼 수 있다.
* 필터중 `frame contains gunicorn` 은 해당 패킷의 내용에 `gunicorn` 이 있는 것만 필터링해줌. 평문으로 수집된 패킷에서 문자열로 검색할 수 있어서 유용함.
  * ex) 특정 url 호출만 골라내기 `frame contains "/api/list"`
  * 내의견) was 에서 수집하면 다 평문일테니 종종 발생하는 패킷 유실 분석에 도움이 될 듯.



## 종합소감

* 실습과그림으로배우는리눅스구조 책과 겹치는 부분이 많지만 좀 더 구체적인 예가 많고 특히 tcp 관련 내용을 많이 다루고 있어 성능의 관점에서 생각의 폭을 넓혀줬음. 이 정리문서를 기본으로 각 애플리케이션마다 적용해나가면 평타는 치지 않을까.