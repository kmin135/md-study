# memcached 의 memory

* memcachced는 메모리 옵션을 줄 수 있다.

```bash
# memcached 실행 자체가 5MB 정도 사용
# memcached가 사용할 메모리로 64MB 할당
docker run -it -m 128m -p 11211:11211 memcached memcached -m 64
```



* 64MB를 초과하면 어떻게 될까?

```java
// spymemcached 사용함
// 파라미터를 조정하여 memcached가 사용 가능한 메모리를 초과
OperationFuture<Boolean> future = null;
for(int i=0;i<setCount;i++) {
    key++;
    String realValue = basicValue + key;
    future = mc.set(String.valueOf(key), 1000, realValue);
}

// set 동작은 비동기라 마지막 set이 완료되는것을 기다림
while(!future.isDone()) {
    Thread.sleep(100);
}
```

* 결론은 에러 없이 정상적으로 실행된다.
* 정말 메모리를 초과하여 set 되는걸까?

```java
long keySize = 0L;
long size = 0L;
for(int key = 1;key<=setCount;key++) {
    String currentKey = String.valueOf(key);
    String getValue = (String) mc.get(currentKey);
    /*
    1~N까지의 key는 데이터가 제거되어 null이 리턴된다.
    N+1~M까지는 정상적으로 저장되어있다.
    */
    if(getValue != null) {
        keySize += currentKey.getBytes().length;
        size += getValue.getBytes().length;
    }
}
// Total은 64MB 보다도 꽤 작은 40MB 정도.
// 메모리 확보 정책에 따라 1개씩 지우는게 아니라 일정 범위의 데이터를 일괄 제거함을 알 수 있음.
System.out.println("keySize 는 " + keySize);
System.out.println("Size 는 " + size);
System.out.println("Total 은 " + (keySize + size));
```

* 메모리를 초과하면 예전 데이터부터 제거됨을 알 수 있고 (LRU) 제거할 때는 1개씩 지우는게 아니라 일정 사이즈 단위로 제거됨을 알 수 있음.



## 공부할 거리

* [Memcached의 확장성 개선](https://d2.naver.com/helloworld/151047)

