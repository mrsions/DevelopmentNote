# 1. Task

- 비동기 프로그래밍이 아니더라도 어떤 행동을 기다리는 등의 로직처리를 할때 사용된다.
- 얼마전 기존에 사용하던 방식 이외의 비동기 처리방식을 발견하여 정리한다.

### 1.1. 예시
- 아래 예시는 일반적인 async 대기문이다.

```csharp
bool hasEnter = false;

// hasEnter가 false일 때 대기하고, true일 때 Thanks를 답하는 비동기 처리문이다.
Task.Run(async delegate
{
    while (!hasEnter)
    {
        Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Wait for enter.");
        await Task.Delay(100);
    }

    Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Thanks");
});

// 엔터가 입력될때까지 대기
Console.ReadLine();

// 엔터가 입력된 뒤 메세지 출력
Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Enter");
hasEnter = true;
```

- 결과는 아래와 같다.
```csharp
[19:33:03.391] Wait for enter.
[19:33:03.500] Wait for enter.
[19:33:03.607] Wait for enter.
[19:33:03.716] Wait for enter.

[19:33:03.763] Enter
[19:33:03.825] Thanks
```
- 이 코드에는 두가지 문제가 있다.

### 1.2. 이벤트타임이 일치하지 않는다.
- 위 결과를 보면 실제 Enter를 입력한 시점부터 task완료시까지 **62ms**의 차이가 발생한다.
- 일반적으로 대기시간이 적어야한다면 아래와 같은 코드로 변경할 것이다.
```csharp
while (!hasEnter)
{
    Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Wait for enter.");
    await Task.Delay(1);
}
...
결과
[19:42:40.193] Enter
[19:42:40.206] Thanks // 13ms 차이
```
- os 최소 슬립시간에 의해 정확히 1ms 차이가 아닌 더 넓은 폭의 차이가 날 수 있다.\
- 대기시간이 정말 중요하다면 아래의 코드도 하나의 방법이다.
```csharp
while (!hasEnter)
{
    Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Wait for enter.");
    await Task.Yield();
}
...
결과
[19:44:45.429] Enter
[19:44:45.429] Thanks // 0ms 차이
```

### 1.3. 리소스를 점유하게된다.
- 위에서 소개한 방식들은 잘 작동하며 웬만해서는 문제가 없다.
- 하지만 결국에는 모두 스케쥴러에 포함되어 리소스를 소모하게된다.
- 특히 두번째 Yield() 방식의 경우에는 여러 Task를 오가야하기때문에 자원소모는 더 심하다.



# 2. TaskCompletionSource

- 위와같은 시나리오에서 무한정으로 스케쥴링을 돌며 관찰하는 것이 아니라, 무언가의 완료를 기다리고 콜백을 받는 형태일 경우에 사용한다.
- TaskCompletionSource에서 생성된 Task를 await할 경우 해당 Task는 TaskScheduler에서 제외되며 결과를 입력할때 다시 TaskScheduler에 삽입된다.


```csharp
 var enterTaskSource = new TaskCompletionSource();

// enterTaskSource에 값이 입력될때 까지 대기
Task.Run(async delegate
{
    await enterTaskSource.Task;

    Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Thanks");
});

// 엔터가 입력될때까지 대기
Console.ReadLine();

// 엔터가 입력된 뒤 메세지 출력
Console.WriteLine($"[{DateTime.Now:HH:mm:ss.fff}] Enter");
enterTaskSource.TrySetResult();

결과
[21:04:53.404] Enter
[21:04:53.412] Thanks // 8ms : 워밍업 상태라고 보인다.

[21:04:56.320] Enter
[21:04:56.320] Thanks // 0ms

[21:04:57.657] Enter
[21:04:57.657] Thanks // 0ms

[21:04:58.675] Enter
[21:04:58.676] Thanks // 1ms
```

