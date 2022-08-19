# 6.2 릴리즈 버전 관리

273p

이미지 버전 태그로 코드를 고유하게 식별하려면 소스 코드와 의존성의 조합마다 고유하게 태그를 부여하게 한다. 즉 애플리케이션을 구성하는 소스코드 또는 의존성이 변경되면 버전도 변경되어야 한다. 이 버전이야말로 아티팩트를 식별하는 고유 버전이다.

배포 입력 아티팩트 유형은 클라우드 플랫폼에 따라 다르다. 클라우드 파운드리 같은 PaaS는 JAR 또는 WAR을 사용하며 쿠버네티스는 컨테이너 이미지를 쓴다.

AWS EC2는 아마존 머신 이미지를 사용하고 데비안이나 RPM 같은 시스템 패키지를 통해 이미지를 굳힌다.

### 6.2.1 메이븐 저장소

274p

메이븐 스냅샷 저장소는 구조가 약간 다르다.

ex) rsocket-core:1.0.0-RC7-SNAPSHOT 메이븐

파일들:

RC-7-20200224.195351-javadoc.jar

RC-7-20200224.195351-sources.jar

…

RC7-SNAPSHOT 이름은 유지한 채, 위 javadoc.jar은 20220816.1234-javadoc.jar.. 이런 식으로 언제든지 수정될 수 있다고 이해했다.

고유성:

소스 코드와 의존성의 조합을 고유하게 식별하는 버전 번호

메이븐 스냅샷은 고유성 원칙을 만족시키지 못한다.

그런데 메이븐 스냅샷 기능을 사용할 상황이 있을까?

## 6.2.2 빌드 도구를 이용한 릴리즈 버전 관리

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/01ea473f-efe7-4d21-85c0-cdbc12aa8f8c/Untitled.png)

태그에 어디에 지정되는 건지 모르겠다.(커밋 태그?)

# 6.5 의존성 관리

실제로 버전이 다른 라이브러리 두 개를 사용하고 있으면 어떻게 될까?

build.gradle:

```python
dependencies {
    // Use JUnit Jupiter for testing.
    testImplementation 'org.junit.jupiter:junit-jupiter:5.8.2'

    // This dependency is used by the application.
    implementation 'com.google.guava:guava:31.0.1-jre'

    implementation group: 'org.checkerframework', name: 'checker-qual', version: '2.11.1'

}
```

'com.google.guava:guava:31.0.1-jre'에서 checker-qual:3.12를 사용하고 있다.

./gradlew dependencies를 실행한다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/82b91084-077c-4f82-99eb-fca14132076c/Untitled.png)

[https://stackoverflow.com/questions/27952388/what-does-arrow-mean-in-gradles-dependency-graph](https://stackoverflow.com/questions/27952388/what-does-arrow-mean-in-gradles-dependency-graph)

gradle에서 자동으로 dependency 충돌을 해결한다고 합니다.

두 개의 버전이 있을 때 더 최신인 버전을 고른다고 하네요.

# 7.7 호출 복원 패턴

## 재시도

[https://resilience4j.readme.io/docs/retry](https://resilience4j.readme.io/docs/retry)

ex)

```python
RetryConfig config = RetryConfig.custom()
  .maxAttempts(2)
  .waitDuration(Duration.ofMillis(1000))
  .retryOnResult(response -> response.getStatus() == 500)
  .retryOnException(e -> e instanceof WebServiceException)
  .retryExceptions(IOException.class, TimeoutException.class)
  .ignoreExceptions(BusinessException.class, OtherBusinessException.class)
  .failAfterMaxAttempts(true)
  .build();

// Create a RetryRegistry with a custom global configuration
RetryRegistry registry = RetryRegistry.of(config);

// Get or create a Retry from the registry - 
// Retry will be backed by the default config
Retry retryWithDefaultConfig = registry.retry("name1");

// Get or create a Retry from the registry, 
// use a custom configuration when creating the retry
RetryConfig custom = RetryConfig.custom()
    .waitDuration(Duration.ofMillis(100))
    .build();

Retry retryWithCustomConfig = registry.retry("name2", custom);
```

## 비율제한

[https://resilience4j.readme.io/docs/ratelimiter](https://resilience4j.readme.io/docs/ratelimiter)

ex)

```python
RateLimiterConfig config = RateLimiterConfig.custom()
  .limitRefreshPeriod(Duration.ofMillis(1))
  .limitForPeriod(10)
  .timeoutDuration(Duration.ofMillis(25))
  .build();

// Create registry
RateLimiterRegistry rateLimiterRegistry = RateLimiterRegistry.of(config);

// Use registry
RateLimiter rateLimiterWithDefaultConfig = rateLimiterRegistry
  .rateLimiter("name1");

RateLimiter rateLimiterWithCustomConfig = rateLimiterRegistry
  .rateLimiter("name2", config);
```

## 벌크헤드

[https://resilience4j.readme.io/docs/bulkhead](https://resilience4j.readme.io/docs/bulkhead)

ex)

```python
// Create a custom configuration for a Bulkhead
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(150)
    .maxWaitDuration(Duration.ofMillis(500))
    .build();

// Create a BulkheadRegistry with a custom global configuration
BulkheadRegistry registry = BulkheadRegistry.of(config);

// Get or create a Bulkhead from the registry - 
// bulkhead will be backed by the default config
Bulkhead bulkheadWithDefaultConfig = registry.bulkhead("name1");

// Get or create a Bulkhead from the registry, 
// use a custom configuration when creating the bulkhead
Bulkhead bulkheadWithCustomConfig = registry.bulkhead("name2", custom);
```

## 서킷 브레이커

[https://resilience4j.readme.io/docs/circuitbreaker](https://resilience4j.readme.io/docs/circuitbreaker)

```
// Create a custom configuration for a CircuitBreaker
CircuitBreakerConfig circuitBreakerConfig = CircuitBreakerConfig.custom()
  .failureRateThreshold(50)
  .slowCallRateThreshold(50)
  .waitDurationInOpenState(Duration.ofMillis(1000))
  .slowCallDurationThreshold(Duration.ofSeconds(2))
  .permittedNumberOfCallsInHalfOpenState(3)
  .minimumNumberOfCalls(10)
  .slidingWindowType(SlidingWindowType.TIME_BASED)
  .slidingWindowSize(5)
  .recordException(e -> INTERNAL_SERVER_ERROR
                 .equals(getResponse().getStatus()))
  .recordExceptions(IOException.class, TimeoutException.class)
  .ignoreExceptions(BusinessException.class, OtherBusinessException.class)
  .build();

// Create a CircuitBreakerRegistry with a custom global configuration
CircuitBreakerRegistry circuitBreakerRegistry = 
  CircuitBreakerRegistry.of(circuitBreakerConfig);

// Get or create a CircuitBreaker from the CircuitBreakerRegistry 
// with the global default configuration
CircuitBreaker circuitBreakerWithDefaultConfig = 
  circuitBreakerRegistry.circuitBreaker("name1");

// Get or create a CircuitBreaker from the CircuitBreakerRegistry 
// with a custom configuration
CircuitBreaker circuitBreakerWithCustomConfig = circuitBreakerRegistry
  .circuitBreaker("name2", circuitBreakerConfig);
```
