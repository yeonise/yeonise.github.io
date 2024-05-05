---
layout: post
title: "redisson tryLock()"
subtitle: 
date: 2024-05-05 02:00:00 +0900
tags: [server, database]
---

### Redisson Lock 사용 예시

``` java
// Returns Lock instance by name.
RLock nonFairlock = redissonClient.getLock(key);  // doesn't guarantees an acquire order by threads.
RLock fairLock = redissonClient.getFairLock(key);  // guarantees an acquire order by threads.

try {
    boolean available = nonFairlock.tryLock(10, 1, TimeUnit.SECONDS);  // set wait time and lease time.

    if (!available) {
        System.out.println("Failed to acquire lock.");
    }

    serviceMethod(key);
} catch (InterruptedException e) {
    throw new RuntimeException();
} finally {
    lock.unlock();
}
```

위 코드의 흐름은 다음과 같다.

1. Lock 인스턴스를 생성한다.
2. Lock 획득을 시도한다.
3. Lock 획득에 성공하면 서비스 메서드를 호출한다.
4. 작업이 끝나면 Lock을 반납한다.

<br />

## tryLock()

``` java
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
  long time = unit.toMillis(waitTime);
  long current = System.currentTimeMillis();
  long threadId = Thread.currentThread().getId();
  Long ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
  // lock acquired
  if (ttl == null) {
      return true;
  }
  ...
```

`tryAcquire()` 를 통해 Lock 획득을 시도한다.  
`ttl` 이 `null` 이라면 Lock 획득에 성공했다고 판단한다.

`ttl` 이 `null` 인 경우 Lock 획득에 성공했다고 판단하는 이유는 `tryAcquire()` 내부에서 호출하는 `tryLockInnerAsync()` 를 살펴보면 알 수 있다.

``` java
<T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
  return evalWriteSyncedAsync(getRawName(), LongCodec.INSTANCE, command,
      "if ((redis.call('exists', KEYS[1]) == 0) " +
      "or (redis.call('hexists', KEYS[1], ARGV[2]) == 1)) then " +
      "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
      "redis.call('pexpire', KEYS[1], ARGV[1]); " +
      "return nil; " +
      "end; " +
      "return redis.call('pttl', KEYS[1]);",
    Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
}
```

`evalWriteSyncedAsync()` 를 사용하여 Redis 서버에서 Lua 스크립트를 실행한다.

1. Lock이 존재하는지 확인한다.  
KEYS[1]에 저장된 이름의 키가 Redis에 존재하는지 `exists` 명령을 사용하여 확인한다.  
키가 존재하지 않으면 Lock을 아무도 소유하고 있지 않다고 판단하고 다음 단계를 진행한다.
``` lua
if ((redis.call('exists', KEYS[1]) == 0) or ...
```

2. 또는 Lock 소유권을 확인한다.  
`hexists` 명령을 사용하여 Lock 키 내에 `threadId` 필드가 존재하는지 확인한다.  
필드가 존재한다면 현재 스레드가 이미 Lock을 소유하고 있다고 판단하고 다음 단계를 진행한다.  
``` lua
... (redis.call('hexists', KEYS[1], ARGV[2]) == 1)) then ...
```

3. Lock이 존재하지 않거나 소유권 확인을 통과하면 Lock 획득을 시도한다.  
`hincrby` 명령을 사용하여 Lock 키 내의 `threadId` 필드를 증가시킨다.  
이는 Lock이 취득된 횟수를 추적하는 카운터 역할을 할 수 있다.  
그리고 `pexpire` 명령을 사용하여 Lock 키에 대한 만료 시간을 설정한다.  
이는 특정 시간이 지난 후 락이 자동으로 해제되도록 한다.
``` lua
redis.call('hincrby', KEYS[1], ARGV[2], 1);
redis.call('pexpire', KEYS[1], ARGV[1]);
```

4. `return nil` 은 Lua에서 아무런 값도 반환하지 않는다는 의미이다.  
이는 Lock 획득에 성공했음을 나타낸다.
``` lua
return nil;
end;
```

5. 다른 스레드가 이미 Lock을 가지고 있다면 KEYS[1]에 해당하는 Lock 키의 남은 유효 시간을 `pttl` 명령을 사용하여 조회하고 반환한다.
``` lua
return redis.call('pttl', KEYS[1]);
```

<br />

다시 `tryLock()` 으로 돌아가자.

``` java
@Override
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
  ...
  // lock acquired
  if (ttl == null) {
      return true;
  }
  
  time -= System.currentTimeMillis() - current;
  if (time <= 0) {
      acquireFailed(waitTime, unit, threadId);
      return false;
  }
  ...
```

Lock 획득에 실패하면 우선 `waitTime` 이 초과되었는지 확인한다.  
만약 `waitTime` 을 초과했다면 Lock 획득에 실패한다.

<br />

아직 `waitTime` 을 초과하지 않았다면 다음 단계를 진행한다.
Lock을 획득할 수 있는

``` java
current = System.currentTimeMillis();
CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
try {
    subscribeFuture.get(time, TimeUnit.MILLISECONDS);
} catch (TimeoutException e) {
      if (!subscribeFuture.completeExceptionally(new RedisTimeoutException(
        "Unable to acquire subscription lock after " + time + "ms. " +
          "Try to increase 'subscriptionsPerConnection' and/or 'subscriptionConnectionPoolSize' parameters."))) {
        subscribeFuture.whenComplete((res, ex) -> {
          if (ex == null) {
              unsubscribe(res, threadId);
          }
        });
      }
      acquireFailed(waitTime, unit, threadId);
      return false;
} catch (ExecutionException e) {
      acquireFailed(waitTime, unit, threadId);
      return false;
}
```

1. `threadId` 로 채널을 구독한다.
``` java
CompletableFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
```

2. `CompletableFuture.get()` 을 호출하여 응답이 올 때까지 대기한다.
``` java
try {
    subscribeFuture.get(time, TimeUnit.MILLISECONDS);
}
```
3. `waitTime` 을 초과했고 구독에 대한 응답이 없다면 `TimeoutException` 이 발생하고 Lock 획득에 실패한다.

여기서 `subscribe()` 는 내부적으로 세마포어를 사용한다.

``` java
public CompletableFuture<E> subscribe(String entryName, String channelName) {
  AsyncSemaphore semaphore = service.getSemaphore(new ChannelName(channelName));
  CompletableFuture<E> newPromise = new CompletableFuture<>();

  semaphore.acquire().thenAccept(c -> {
    if (newPromise.isDone()) {
        semaphore.release();
        return;
    }

    E entry = entries.get(entryName);
    if (entry != null) {
        entry.acquire();
        semaphore.release();
        entry.getPromise().whenComplete((r, e) -> {
            if (e != null) {
                newPromise.completeExceptionally(e);
                return;
            }
            newPromise.complete(r);
        });
        return;
    }

    E value = createEntry(newPromise);
    value.acquire();
    ...
```

1. `service.getSemaphore(new ChannelName(channelName))` 을 통해 채널 이름을 기반으로 `locks` 에서 `AsyncSemaphore` 를 가져온다.
``` java
public AsyncSemaphore getSemaphore(ChannelName channelName) {
    return locks[Math.abs(channelName.hashCode() % locks.length)];
}
```

2. `semaphore.acquire()` 를 통해 세마포어 획득을 시도한다.  
세마포어 획득에 성공한 스레드는 구독을 요청하고 응답을 기다린다.
``` java  
public CompletableFuture<Void> acquire() {
    CompletableFuture<Void> future = new CompletableFuture<>();  // 허가 취득 시도의 완료 여부를 알리기 위해 사용한다.
    listeners.add(future);  // listeners 대기열에 future 객체를 추가한다.
    tryRun();
    return future;
}
```
세마포어 획득 과정에서 반복문이 수행된다.
``` java
private void tryRun() {
  while (true) {
      if (counter.decrementAndGet() >= 0) {  // 0보다 크다면 허가가 가능하다.
          // listeners에서 첫 번째 CompletableFuture 객체를 제거한다. 이는 가장 오래 기다린 요청을 나타낸다.
          CompletableFuture<Void> future = listeners.poll();
          if (future == null) {  // 현재 요청을 포함하여 대기 중인 요청이 없는 경우
              counter.incrementAndGet();  // decrease를 취소하고 종료한다.
              return;
          }

          // null이 아닌 future 객체를 반환하면 세마포어 획득에 성공했다고 판단한다.
          if (future.complete(null)) {  // 완료를 알리고 종료한다.
              return;
          }
      }

      // 실패하면 decrease를 취소하고 종료한다.
      if (counter.incrementAndGet() <= 0) {
          return;
      }
  }
}
```
<br />

**_This post is still in progress..._**

<br />

> 지금까지 이해한 것
> 
> 1. Redisson은 Lua 스크립트를 사용해 Lock 획득을 위한 첫 요청을 보낸다.
> 2. Lock 획득에 성공하면 그대로 종료하고, 획득에 실패하면 구독을 요청한다.
> 3. 구독을 요청하는 과정에서 세마포어를 사용하여 구독을 관리한다.
> 
> 학습이 필요한 것
> 
> 1. 구독에 실패하면 `tryLock()` 호출이 종료될 것으로 예상하는데 이후 반복문을 사용하여 Lua 스크립트를 호출하는 코드가 있는 이유
> 2. 구독에 실패하면 재구독을 요청하지 않고 반복문을 통해 Lock 획득을 시도하는 이유
> 3. 50개의 세마포어를 생성한 다음 `threadId.hashCode() % 50` 값으로 스레드를 분류하여 관리하는 이유

<br />

#### reference

[Redisson Source Code](https://github.com/redisson/redisson)
