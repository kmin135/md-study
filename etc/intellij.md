# Tip



## 애플리케이션 여러개 띄우기

* Empty Project를 생성 후 Module로 애플리케이션들을 생성한다.
* Spring Boot 앱 여러개 띄울 때 유용하다.



## livereload

1. `spring-boot-devtools` 을 추가한다. 추가하면 기본적으로 활성화되므로 application.yml 에서 해줄 작업은 없다.
1. Settings > Advanced Setting > 'Allow auto-make to start even if ...` 체크
  * 대충 21년도 이전글을 보면 이상한 고급옵션을 키는글이 나올텐데 21.3부터인가 이 옵션으로 바뀌었다고 함.
2. Settings > Build, Execution, Deployment > Compiler > 'Build project automatically' 체크
3. Edit Configuration 의 `On update action`, `On frame deactivation` 을 모두 `Update classes and resources` 로 변경
  * 이걸 안 하면 5초 정도 걸리던게 거의 1초 이내에 반영되었음.

---

* livereload는 기본 적용되고 chrome에도 확장프로그램 livereload를 설치하고 작업바에서 활성화주면 된다.
* 이후에는 코드수정 후 주기적으로 자동빌드된 후 브라우저도 자동 리프레시되고 `Ctrl+F9` 으로 빌드하면 즉시 재빌드 후 브라우저도 리프레시된다.
  * html만 수정한 경우에도 `Ctrl+F9` 로 빌드를 수행하면 빌드할 게 없으니 작업자체는 바로 끝나면서 html도 바로 갱신되는 효과가 있다.
  * 아니면 `Ctrl+S` 로 저장해도 되는데 빌드하는것보다 한박자 느리게 반영된다.
* 참고 : https://stackoverflow.com/questions/33869606/intellij-15-springboot-devtools-livereload-not-working#answer-63188493





# 트러블슈팅



## hyper-v 사용하는 windows에서 시작실패



* 프로그램 시작 중 아래 에러 발생하며 시작실패 (Ultimate 2020.3.2 에서 발생)

```java
Internal error. Please report to http://jb.gg/ide/critical-startup-errors
java.util.concurrent.CompletionException: java.net.BindException: Address already in use: bind
at ...
```

* 원인은 IDE 실행시 필요한 포트범위를 hyper-v가 다 가져가서 그런듯함. issue보면 해결방법은 몇 개 있는데 내가 해결본건 아래와 같음
* 관련 issue
  * [Revise IDE folders locking mechanism (don't fail startup if all ports in range are taken) : IDEA-238995 (jetbrains.com)](https://youtrack.jetbrains.com/issue/IDEA-238995?_ga=2.210649425.536837968.1612074844-2146385194.1599746225)
  * [IntelliJ IDEA(JetBrains IDE) 시작 오류 조치 (velog.io)](https://velog.io/@erpsm92/IntelliJ-IDEAJetBrains-IDE-시작-오류-조치)
* 해결

```powershell
dism.exe /Online /Disable-Feature:Microsoft-Hyper-V
# 이후 윈도우를 재시작
netsh int ipv6 add excludedportrange protocol=tcp startport=6942 numberofports=10
# JetBrains 사의 IDE를 동시에 10개 이상은 안 쓸 거 같아 10개만 예약해뒀다.
dism.exe /Online /Enable-Feature:Microsoft-Hyper-V /All
# 한 번 더 재시작
```

