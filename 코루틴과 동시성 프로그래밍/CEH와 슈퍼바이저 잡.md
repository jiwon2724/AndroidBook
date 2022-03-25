### GlobalScope
어디에도 속하지 않지만 원래부터 존재하는 전역 `GlobalScope`가 있다. 이 전역 스코프를 이용함녀 코루틴을 쉽게 수행할 수 있다.

~~~kotlin
fun main(){
    val job = GlobalScope.launch(Dispatchers.IO) {
        launch { printRandom() }
    }
    Thread.sleep(1000L) // main이 runBlocking이 아니기 때문에 사용한다. delay사용 불가
}

suspend fun printRandom() {
    delay(500L)
    println(Random.nextInt(0, 500))
}
~~~

`GlobalScope`는 어디에도 속하지않고, 계층적인 구조가 아니기 때문에 관리할 포인트가 어려워진다.<br>
영원히 동작하게 된다는 문제점이 있다. 일반적으론 `GlobalScope`를 잘 사용하지 않는다.

### CoroutineScope
`GlobalScope`보다 권장되는 형식은 `CoroutineScope`를 사용하는 것이다.

~~~kotlin
fun main(){
    val scope = CoroutineScope(Dispatchers.Default)
    val job = scope.launch(Dispatchers.IO) {
        launch{ printRandom() }
    }
    Thread.sleep(1000)
}

suspend fun printRandom() {
    delay(500L)
    println(Random.nextInt(0, 500))
}
~~~

하나의 코루틴 엘리먼트, 디스패처 `Dispatchers.Default`만 넣어도 코루틴 컨텍스트가 만들어지기 때문에 이렇게 사용이 가능하다.<br>
이제부터 `scope`로 계층적으로 형성된 코루틴을 관리할 수 있다.<br>

### CEH(코루틴 익셉션 핸들러)
예외를 가장 체계적으로 다루는 방법은 CHE(Coroutine Exception Handler)를 사용하는 것이다.<br>
`CoroutineExceptionHandler`를 이용해서 우리만의 CEH를 만든 다음 상위 코루틴 빌더의 컨텍스트에 등록한다.

~~~kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(Dispatchers.Default)
    val job = scope.launch(ceh) {
        launch{ printRandom() }
        launch{ printRandom2() }
    }
    job.join()
}


suspend fun printRandom() {
    delay(1000)
    println(Random.nextInt(0, 500))
}

suspend fun printRandom2() {
    delay(500L)
    throw ArithmeticException()
}

val ceh = CoroutineExceptionHandler { _, exception -> 
    println("Something happend: $exception")
}
~~~

실행결과 : <br>
Something happend: java.lang.ArithmeticException <br>

### runBlocking과 CEH
`runBlocking`에서는 CEH를 사용할 수 없습니다. `runBlocking`은 자식이 예외발생으로 인해 종료되면 그냥 종료된다.

### SupervisorJob
슈퍼 바이저 잡은 예외에 의한 취소를 아래쪽으로 내려가게 한다.

~~~kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(Dispatchers.Default + SupervisorJob() + ceh)
    // 슈퍼바이저잡은 예외가 발생했을 때, 예외를 아래로만 보낸다.
    val job1 = scope.launch { printRandom() }
    val job2 = scope.launch { printRandom2() }
    joinAll(job1, job2)
}


suspend fun printRandom() {
    delay(1000)
    println(Random.nextInt(0, 500))
}

suspend fun printRandom2() {
    delay(500L)
    throw ArithmeticException()
}

val ceh = CoroutineExceptionHandler { _, exception -> 
    println("Something happend: $exception")
}
~~~

printRandom2는 실패했지만, printRandom은 제대로 수행된다.<br>
`joinAll`은 복수개의 `Job`에 대해 `join`을 수행하여 완전히 종료될 때까지 기다린다.

### SupervisorScope
코루틴 스코프와 슈퍼비이저 잡을 합친듯 한 `supervisorScope`가 있다.

~~~kotlin
fun main() = runBlocking {
    val scope = CoroutineScope(Dispatchers.Default)
    // 슈퍼바이저잡은 예외가 발생했을 때, 예외를 아래로만 보낸다.
    
    val job = scope.launch {
        supervisorFunc()
    }
}

suspend fun printRandom() {
    delay(1000)
    println(Random.nextInt(0, 500))
}

suspend fun printRandom2() {
    delay(500L)
    throw ArithmeticException()
}

suspend fun supervisorFunc() = supervisorScope {
    launch { printRandom() }
    launch(ceh) { printRandom2() }
    // 무조건 예외가 발생하는 곳에 ceh를 붙여줘야한다.
}

val ceh = CoroutineExceptionHandler { _, exception -> 
    println("Something happend: $exception")
}
~~~

