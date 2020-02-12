# Redis와 Rabbitmq

* redis는 일반적으로 cache 저장소로 많이 쓰지만 Publish/Subscribe 기능도 제공함
* 이를 queue 처럼 쓸 수 있지만 Pub/Sub은 어디까지나 발행자의 이벤트를 모든 구독자가 받아보는 형식이므로 일반적인 큐와는 다름
  * redis에 하나의 key를 스택으로 사용하여 lock으로 사용하는 방법도 있으나 트래픽제어를 redis client 단에서 해야한다는 점에서 적절하지 않음
* 반면에 rabbitmq와 같은 queue들은 트래픽제어를 큐가 자체적으로 해줌