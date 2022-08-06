# sre with java

## 3.1 관찰 가능성의 세 주축

logging

metric

tracing

https://www.oreilly.com/library/view/distributed-systems-observability/9781492033431/ch04.html

## 4.7 모든 자바 마이크로서비스에 통용되는 서비스 지표

### 에러

```python
sum(
  rate(
     http_server_requests_seconds_count{outcome="SERVER_ERROR", uri="$ENDPOINT"}[2m]
  )
)

```

### 레이턴시

```python
http_server_requests_seconds_max > $THRESHOLD
```

### 클라이언트(아웃바운드) 요청

```python
topk(
  1,
  sum(
     rate(
       http_client_requests_seconds_max{serviceName="CALLED", uri="/api/.."}[2m]
     )
  ) by (clientName)
) > $THRESHOLD
```

### 가비지 수집 다운타임

```python
sum(
   sum_over_time(
      sum(increase(jvm_gc_pause_seconds_sum[2m]))[1m:]
   )
)
```

### 힙 사용률

이든 공간(young generation)

신규 객체는 모두 이곳에 할당된다. 이 공간이 가득 차면 보조 가비지 수집 이벤트가 발생한다.

생존자 공간(survivor space)

보조 가비지 수집이 발생하면 모든 생존 객체가 생존자 공간에 복사된다. 생존 객체는 참조가 남아 있으므로 수집 대상이 아니다. 생존자 공간에 도착한 객체는 생존 기간이 임계점에 도달하면 지난 세대로 승격된다. 생존자 공간이 어린 세대의 모든 개체를 담을 수 없는 상황일 때는 생존자 공간을 건너뛰고 바로 승격되기도 한다. 할당 압력의 위험 수준을 측정하는 핵심 요소가 바로 이러한 이른 승격이다.

지난 세대(old generation)

오래 살아남은 객체가 저장되는 공간이다. 객체가 이든 공간에 저장되는 순간 생존 기간이 설정되고 일정한 시기에 다다르면 지난 세대로 옮겨간다.

기본적으로 우리가 파악해야 할 상황은 이들 공간 중 한 곳 이상이 지나치게 ‘가득 차' 있는지다. 공간이 가득차는 순간은 모니터링하기 까다롭다. jvm 가비지 수집은 공간이 채워지자 마자 비워버리도록 설계됐기 때문이다. 따라서 공간이 가득차는 현상 자체는 문제가 아니다.

가득 찬 상태가 유지되고 있을 때가 진정한 문제 상황이다.

```python
max_over_time(
   (
      jvm_memory_used_bytes{id="G1 Old Gen"} /
      jvm_memory_committed_bytes{id="G1 Old Gen"}
   )[5m:]
)
```

### CPU

```python
process_cpu_usage> 0.8
```

### file descriptor

http 접속으로 file descriptor가 고갈됐을 때 톰캣의 로그

```python
java.net.SocketException: Too many open files
```

```python
process_open_fds / process_max_Fds > 0.8
```

### 비정상 트래픽

```python
sum(
   rate(
      http_server_requests_seconds_count{status="403", uri="$ENDPOINT"}[2m]
   )
)
```

# 5.7 블루 그린 배포

…

블루/그린 전략이라는 용어의 의미는 청색 또는 녹색으로 지정된 두 서버 그룹이 각각 트래픽을 처리한다는 뜻이다.

그러나 이 전략이 항상 두 그룹으로 진행되지는 않는다. 또한 색상 지정 자체에 의미를 부여하면 안 된다. 특정 서버 그룹이 계속 유지되다가 새로운 서비스 버전으로 전환된다는 뜻이 아니다.
