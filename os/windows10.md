## 경로

* 시작프로그램
  * `C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp`



## 특정시간 자동 절전모드 진입

* powershell 관리자권한으로

```
powercfg -hibernate off
```

* 작업 스케줄러 등록

```
트리거 : 매일 xx:xx에
동작 : 프로그램 시작
- 프로그램/스크립트 : rundll32.exe
- 인수 추가 : Powrprof.dll,SetSuspendState
```

