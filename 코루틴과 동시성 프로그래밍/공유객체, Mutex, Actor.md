### 공유 객체 문제

~~~kotlin
fun main() = runBlocking {
    withContext(Dispatchers.Default) {
        massiveRun {
            counter++
        }
    }
    println("Counter : $counter")
}

var counter = 0

suspend fun massiveRun(action: suspend() -> Unit) {
    val n = 100
    val k = 10000
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms동안 ${n * k}개의 액션을 수행했습니다.")
}
~~~

수행결과 : <br>
76 ms동안 1000000개의 액션을 수행했습니다.<br>
Counter : 989014<br>

`withContext`는 수행이 완료될 때 까지 기다리는 코루틴 빌더이다. 뒤의 `println("Countext = $counter")`는<br>
잠이 들었다 `withContext` 블록의 코드가 모두 수행되면 깨어나 호출된다.<br>

위 코드는 불행히도 항상 10000이 되는 것은 아니다. 여러 쓰레드간 서로 다른 정보를 가지고 있기 때문이다.<br>
Dispatchers.Default에 의해서 다른 쓰레드에서 수행될 수 있다. 

### volatile 적용하기
손 쉽게 생각할 수 있는 방법은 `volatile`이다 이는 가시성을 제공해주는 어노테이션이다.<br>
가시성이란? 한 쪽에서 수정 하면 다른쪽에서도 볼 수 있어야함.

~~~kotlin
@Volatile // 어떤 쓰레드에서 변경해도 다른 쓰레드에게 영향을 준다
var counter = 0
~~~

하지만 `volatile`은 가시성 문제만을 해결할 뿐, 동시에 읽고 수정해서 생기는 문제를 해결하지 못한다.<br>
값을 읽는건 문제가 안되지만, 읽고 난 후 값을 수정하는 경우 양쪽에서 발생했을 때 불일치가 발생한다.<br>

### 스레드 안전한 자료구조 사용하기
`AtomicInteger`와 같은 스레드 안전한 자료구조를 사용하는 방법이 있다.<br>
이는 `incrementAndGet()`이라는 메서드를 제공하는데 값을 증가시키고 현재 가지고 있는 값을 리턴하는 메서드다<br>
이때 다른 스레드가 그 값을 변경하지 못하게한다.

~~~kotlin
val counter = AtomicInteger()

massiveRun {
    counter.incrementAndGet()
}
~~~

`AtomicInteger`가 이 문제에는 적합한데 항상 정답은 아니다. 상황에 맞는 최선의 방법을 선택해야한다.

### 스레드 한정
`newSingleThreadContext`를 이용해서 특정한 스레드를 만들고 해당 스레드를 사용할 수 있다.

~~~kotlin
fun main() = runBlocking {
    withContext(counterContext) {
        massiveRun {
            counter++
        }
    }
    println("Counter : $counter")
}

var counter = 0
val counterContext = newSingleThreadContext("CounterContext")
// 하나의 코루틴 컨텍스트를 만들고 특정 스레드 하나를 만들고 이 스레드만 사용한다.
// counterContext를 사용하면 해당 스레드에서만 실행됨.

suspend fun massiveRun(action: suspend() -> Unit) {
    val n = 100
    val k = 10000
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms동안 ${n * k}개의 액션을 수행했습니다.")
}
~~~

### 뮤텍스
뮤텍스는 상호배제(Mutual exclusion)의 줄임말이다.<br>
공유 상태를 수정할 때 임계 영역(critical section 한개에 한사람만)을 이용하게 하며, 임계 영역을 동시에 접근하는 것을 허용하지 않는다.

~~~kotlin
val mutex = Mutex()

massiveRun {
    mutex.withLock {
        // 뮤텍스에 들어가려고 할 때 withLock을 이용해 블록에 들어가면 된다
        // 여러 스레드중 한 스레드만 진입하고, 다른 스레드는 기다렸다 작업이 완료되면 counter++를 수행한다
        counter++
    }
}
~~~

### 액터
액터가 독점적으로 자료를 가지고 그 자료는 다른 코루틴과 공유하지 않고 액터를 통해서만 접근하게 만든다.<br>

먼저 실드 클래스를 만들어서 시작해야한다.<br>

~~~kotlin
sealed class CounterMsg
object IncCounter: CounterMsg()
class GetCounter(val response: CompletableDeferred<Int>): CounterMsg()
~~~

1. 실드(sealed) 클래스란? 외부에서 확장이 불가능한 클래스이다.<br>
2. IncCounter는 싱글톤으로 인스턴스를 만들 수 없다. 액터에게 값을 중가시키기 위한 신호로 쓰임.
3. GetCounter는 값을 가져올 때 쓰며 `CompletableDeferred<Int>`를 이용해 값을 받아온다.
4. CounterMsg는 액터에게 보내기 위함이다. IncCounter와 GetCounter로 보내는 신호가 2가지이다.

~~~kotlin
fun main() = runBlocking {
    val counter = counterActor()
    withContext(Dispatchers.Default) {
        massiveRun {
            counter.send(IncCounter)
        }
    }
    
    val response = CompletableDeferred<Int>()
    counter.send(GetCounter(response))
    println(response.await())
    counter.colse
}

suspend fun massiveRun(action: suspend() -> Unit) {
    val n = 100
    val k = 10000
    val elapsed = measureTimeMillis {
        coroutineScope {
            repeat(n) {
                launch {
                    repeat(k) { action() }
                }
            }
        }
    }
    println("$elapsed ms동안 ${n * k}개의 액션을 수행했습니다.")
}

sealed class CounterMsg
object IncCounter: CounterMsg()
class GetCounter(val response: CompletableDeferred<Int>): CounterMsg()


fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0 // 액터 안에 상태를 캡슐화해두고 다른 코루틴이 접근하지 못하게 한다.
    
    for(msg in channel) { // 외부에서 보내는 것은 채널을 통해서만 받을 수 있다.
        when(msg) {
            is IncCounter -> counter++ // 증가
            is GetCounter -> msg.response.complete(counter) // 현재 상태 반환
        }
    }
}
~~~

액터는 자료를 공유하는게 아니고 관리하는 액터를 만든후 그 액터에게 신호를 보내 원하는 결과를 얻게하는 자료구조이다.







