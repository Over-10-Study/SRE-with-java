# 2장 애플리케이션 메트릭

## 2.1 블랙박스 vs. 화이트박스 모니터링

관찰 가능한 요소가 무엇인지에 따라 블랙박스, 화이트박스 모니터링으로 분류한다.

### 블랙박스

수집기가 입력과 출력을 관찰할 수 있지만, 내부 작동 메커니즘은 알 수 없다. HTTP 요청과 응답이 대표적인 예

사용자로서 관찰하듯 보이는 현상을 외부에서 테스팅하는 것

### 화이트박스

수집기가 입력과 출력에 더해 내부 작동 메커니즘을 관찰할 수 있다. 화이트박스 수집기는 애플리케이션 코드 내부에서 작동함

에이전트가 애플리케이션 프로세스에 부착돼서 상태를 관찰할 수 있는데, 이것이 화이트박스 수집기처럼 보이기도 하지만 블랙박스 수집기다..

결국에 화이트박스 수집기 = 블랙박스 수집기가 감지하는 모든 항목 + 자세한 내부 정보

로그, JVM의 프로파일링과 같은 인터페이스류, 또는 내부 통계정보를 보내주는 HTTP 핸들러를 포함한 시스템의 내부에 의해 노출된 측정 기준에 근거한 모니터링

> 반드시 어느 한 쪽을 선택해야 하는 조건은 없다.

## 2.2 차원형 메트릭

최신 모니터링 시스템은 하나의 Metric 명 - 여러 key-value로 구성된 스키마를 채택

메트릭 저장 비용 = 모든 고유한 key-value pair

주기적으로 메트릭 데이터를 저장소로 옮기는 케이스도 있을 수 있다. 버릴 건 버리고 취할 것만 취하면서 스토리지 사용량을 줄일 수도 있다.

### 예시

`http.server.requests{method=GET,status=200,uri=/a1}` 에서 method, status, uri value가 바뀌면서 다양한 데이터 수집

## 2.3 계층형 메트릭

key-value pair 없이 이름으로만 메트릭을 정의한다. 계층형 메트릭은 태그를 온전히 변환할 수 없고, 와일드카드는 매우 부실함.

### 예시

위에서 말한 케이스는 `httpSeverRequest.method.GET`, `httpSeverRequest.method.POST` 와 같은 형태로 표현한다.

## 2.4 마이크로미터의 미터 레지스트리

마이크로미터는 자바로 제작된 차원형 메트릭 측정 라이브러리.

마이크로미터에서 **Meter**는 애플리케이션 **측정 결과를 수집**하는 인터페이스, 이 측정 결과들을 개별적으로 보면 메트릭.

MeterRegistry 구현 라이브러리는 Maven Central, JCenter에 공개된다. (ex - `io:micrometer:micrometer-registry-prometheus`)

```java
MeterRegistry prometheusMeterRegistry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
MeterRegistry atlasMeterRegistry = new AtlasMeterRegistry(AtlasConfig.DEFAULT);

MeterRegistry registry = new CompositeMeterRegistry();
registry.add(prometheusMeterRegistry);
registry.add(atlasMeterRegistry);

// 여기서 prometheusMeterRegistry랑 atlasMeterRegistry에 각각 설정할 필요 없이, compositeRegistry에 
// 설정하면 된다.
registry.counter("my.counter");
```

복합 레지스트리는 다른 복합 레지스트리에 다시 등록할 수도 있다.

slf4j의 LoggerFactory처럼 CompositeMeterRegistry를 Global Static으로 패키징한다. 의존성 주입으로 MeterRegistry를 주입받지 못하는 케이스에 정적 레지스트리를 사용한다.

## 2.5 미터 생성

마이크로미터가 지원하는 Meter 타입들은 두 가지 방식으로 메트릭을 등록할 수 있다. 메트릭 설정 조건이 얼마나 상세한가에 따라서 어느 쪽을 선택할 지 결정한다.

1. 플루언트 빌더
2. 축약 구문

### 플루언트 빌더

```java
Counter counter = Counter.builder("이름")
  .tag("status", "200")
  .tags("method", "GET", "outcome", "SUCCESS") // 복수 태그
  .description("description")
  .baseUnit("requests")
  .register(registry);
```

### 축약 구문

```java
Counter counter = registry.counter("이름", "status", "200", "method", "GET", "outcome", "SUCCESS");
```

## 2.6 메트릭명

메트릭 명을 정하는 것도 중요하다. 변수명을 잘 지어야하는 것처럼, 정보를 충분히 제공하고, 메트릭명만으로도 의미 있는 값을 얻을 수 있도록 해야한다.

- 이름의 일부를 구분할 때는 항상 마침표를 사용한다.
- 단위를 나타내는 단어 또는 total 등의 단어를 미터명에 추가하지 않는다.

## 2.7 미터 클래스

메트릭 수집기는 다양한 미터 클래스를 제공하고, 각 클래스는 하나 이상의 메트릭이나 통계를 발행한다. 이 중 필요에 따라 적절한 미터 타입을 사용해야 한다.

## 2.8 게이지

게이지는 시간의 흐름에 따라 증가, 감소하는 값을 순간적으로 측정한 값, 순간 값을 측정해 샘플링하고, 좌표에 표시하면 시계열 그래프가 된다. 게이지는 관찰전까지 변하지 않고 관찰하는 순간 변경되는 미터다.

- TimeGauge -> 게시되는 시점에 대상 모니터링 시스템의 기본 시간 단위로 변경된다.
- MultiGauge

**셀 수 있는 것은 절대 게이지로 측정하지 않는다.**

## 2.9 카운터

카운터는 횟수, Counter 인터페이스는 고정된 수량을 측정하고 증가시킨다.

마이크로미터의 내장 메트릭은 다양한 카운터를 포함한다. (jetty.async.requests, postgres.transactions, jvm.classes.loaded, jvm.gc.memory.promoted)

카운터를 기반으로 그래프나 Alert를 설정할경우 일정 시간 간격 동안 어떤 사건이 얼마만큼의 빈도로 발생하는지 관심을 기울여야한다.

특정 시간 간격 당 카운터의 비율을 대시보드에 나타내고 alert 설정할 경우 애플리케이션의 구동 시간과 무관하게 이상 행동을 발견할 수 있다.

일정한 간격을 두고 지속적으로 게시되는 메트릭은 중간에 끊길 위험이 있는데, 다음 주기에 다시 측정될 것을 전제하기 때문에 재시도하지 않는다. 누적 카운터도 마찬가지로 실패하기 직전의 측정값은 저장되지 않는다. 그러나 매우 중요한 카운트(법적인 문제와 관련된다거나..)가 있다면, 메트릭은 참조의 형태로만 사용하고, 저장소에 직접 저장하는 것이 좋다.

- 아틀라스의 카운터 비율 - `name,queue.insert,:eq`
- 프로메테우스의 카운터 비율 - `rate(queu_insert_sum[2m])`

## 2.10 타이머

타이머는 단시간 동안 레이턴시를 측정하고 특정 이벤트의 빈도를 측정할 때 사용한다. 아래 3가지는 기본적으로 보고된다.

### 카운트(count)

- 타이머가 기록하는 개별적인 측정 수치, API 엔드포인트 측정 Timer에서는 API Endpoint에 들어온 요청의 건수다.

### 합계(sum)

- 모든 요청을 성공적으로 응답하는 데 걸린 시간의 합, API 요청의 응답이 10ms, 20ms, 30ms 라면 합계는 60ms다

### 최댓값(maximum)

- 말 그대로 최댓값이다. 주기가 반복될 때 갱신된다. 그렇기때문에, 애플리케이션 시작부터 현재까지 중 최댓값을 의미하지는 않고, `최근`의 최댓값이다. 

타이머는 아래와 같은 통계도 제공할 수 있다(optional)

### 서비스 수준 목표(SLO) boundary

- 특정 경계값보다 작거나 같에 측정된 요청 횟수

### 백분위수(percentile)

- 사전에 계산된 백분위수

### 히스토그램(histogram)

- 버킷(간격) 세트에 저장되는 일련의 카운트로 구성됨

### 위와 같은 통계들을 어떻게 활용할 수 있을지?

1. 카운트는 처리량(Throughput)
2. 카운트와 합계를 이용해서 집계 평균을 낼 수 있다
   1. 여러 인스턴스가 있다면, 각 평균의 평균을 내는 오류를 만들면 안된다.
   2. 평균은 가용성 모니터링에는 적합하지 않다. **가능하면 평균을 아예 사용하지 않는 것을 권한다.**
   3. 가용성 모니터링에는 오히려 최댓값이나 높은 백분위 통계가 더 유용하다.
3. 최댓값 폐기, 푸시간격 (최댓값은 일정 주기가 지나면 폐기되니까..)
   1. distributionStatisticsBufferLength와 distributionStatisticExpiry 설정으로 폐기 간격을 조정할 수 있다.
4. 기간별 합계의 합계
   1. 거의 대부분 유용하게 쓰지 못한다.
5. 시간의 기본 단위
   1. 정답은 없고, 모범적인 정확도도 없다. 관습에 달린 선택지다.

### 마이크로미터 내장 타이머 예시

- `http.server.requests`
- `jvm.gc.pause`
- `mongodb.driver.commands`

### 레이턴시 분포의 일반적 특성

실제 상황에서 실행 시간은 대부분 다봉 분포를 형성한다. 가장 일반적인 자바 실행 시간 분포는 쌍봉 분포다. (봉우리가 두개임.)

### 백분위/분위

> 평균은 최댓값과 1/2 중윗값 사이에 있는 임의의 수

평균은 실행 시간을 모니터링할 때 거의 쓸모가 없다..

사전 계산된 백분위수는 제약을 갖기 때문에, 항상 히스토그램을 지원하는 모니터링 시스템을 써야한다.

### 히스토그램

히스토그램은 개별 측정 시간의 근사치를 나타낼 수 있다. 측정 시간은 버킷(간격)에 나누어 담긴다.

### 서비스 수준 목표 경계(SLO boundary)

percentile이나 histogram 과 비슷하게 Timer 빌더 또는 MeterFilter로 추가할 수 있다.

- 20ms 안에 90%
- 100ms 안에 99ㅣ.99%
- 2s 안에 100%

위와 같은 형식으로 목표를 계층적으로 구성해야 합리적이다.

