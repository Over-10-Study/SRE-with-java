## 2.12 장기 작업 타이머 (long task timer)

측정하고 있는 이벤트가 실행되는 동안 계속 시간을 측정할 수 있는 타이머

아래와 같은 통계를 제공

- Active(활성): 현재 진행 중인 작업의 개수
- Total Duration(총 시간): 측정 코드 블록을 실행하고 있는 모든 작업의 실행 시간 합계
- Max(최대): 가장 오래 진행되고 있는 작업의 실행 시간
- Histogram(히스토그램): 진행 중인 작업의 이산화된 버킷 집합
- Percentile(백분위수): 진행 중인 작업의 실행 시간으로 사전 계산한 백분위수

### 장기작업 타이머와 타이머가 근본적으로 어떻게 다른가?

장기작업 타이머는 작업이 시작되면 총 기간은 늘어나지만 작업이 완료되면 더 이상 늘어나지 않는다. 반면에 타이머로 측정한 작업은 완료되기 전에는 보고되지 않는다.

장기 작업 타이머 평균은 현 시점에서 실행 중인 활성 작업 시간의 평균이다. 최댓값은 현 시점에서 실행중인 작업 중 가장 오래 실행되고 있는 작업을 나타내고, 타이머랑 비슷한 방식으로 폐기된다.  

- 장기 작업 타이머를 사용한 좋은 예시 - https://github.com/Netflix/edda
  - 메타데이터를 갱신할 때 소요되는 전체 시간을 추적할 수 있다.

## 2.13 미터 타입 선정

> 셀 수 있는 것을 게이지로 측정하지 말고 시간으로 측정할 수 있는 것을 세려고 하면 안 된다.

일반적으로 매번 2분을 초과하는 요청을 측정할 때는 LongTaskTimer를 선택한다. in-flight 요청, 특히 실패 가능성이 있고 예상 시간이 수 분, 수 시간 단위로 늘어날 위험이 있는 요청도 LongTaskTimer로 측정한다.

## 2.14 비용 제어

컴포넌트를 추가해서 측정 영역이 확장된다고 해도 반드시 텔레메트리 비용이 증가하는 것은 아니다. 가능한 모든 key-value 태그 조합을 고려해서 카디널리티 경계를 정해야한다. 넷플릭스에서 일반적으로 제시하는 선은 시계열이 100만을 넘지 않을 정도로 카디널리티를 유지한다.

스프링부트의 WebMVC 및 Webflux를 측정하면 유용한 태그들이 추가된다. 

### 메서드

HTTP 메서드(GET, POST 등)

### 상태

HTTP 상태 코드 (200~202: 성공/생성/수락, 304: 변경없음, 400: 잘못된 요청, 500: 내부 서버 오류)

### 응답 결과

상태 코드 요약 정보 (ex. SUCCESS, CLIENT_ERROR, SERVER_ERROR 등)

### URI

태그 카디널리티가 순식간에 손 쓸 수 없는 지경에 이르는 주요한 원인이다. 만약 엔드포인트에 경로 변수나 매개변수가 포함되면 프레임워크는 원래 경로(변수가 포함된 경로)대신 변수로 대체되지 않은 경로를 지정한다.

예를 들어 /api/customer/123 이나 /api/customer/456 는 /api/customer/{id} 로 대체한다.

그리고 /api/doesntexist/1, /api/doesntexistagin 처럼 존재하지 않는 URI에 접근해 서버가 404를 반환할 때도 URI를 제한해야한다. 그렇지 않으면 잘못된 URI가 모두 새 태그값으로 기록된다. 스프링부트는 상태 코드가 404일 때마다 URI 태그 값에 NOT_FOUND를 지정한다. 그리고 403일 때는 REDIRECT를 지정한다. 요청 페이지가 존재하든 존재하지 않든, 서버는 인증되지 않은 요청을 항상 리다이렉션한다.

### 예외

요청 결과로 500 Inernal Server Error 발생하면 예외 클래스 명이 기록된다.



비용을 과도하게 최적화하려는 충동은 자제해야 한다. 기본적으로 제공되는 측정 방식으로 태그 카디널리티를 제한하기 어려울 때는 메트릭 집합의 설정을 통해 제한 방식을 정의할 수 있다. (Jetty의 HttpClient 측정)

일반적으로 아래처럼 요청을 생성한다.

```java
Request post = httpClient.POST("https://customerservice/api/customer/" + customerId);
post.content(new StringContentProvider("{\\"detail\\": \\"all\\"}"));
ContentResponse response = post.send();
```

그러면 customerId 마다 새로운 URI 태그값이 생긴다. 그래서 여기서도 스프링부트의 WebMVC나 WebFlux 측정처럼 경로 변수를 대체하여 태그값으로 지정해야한다.

```java
HttpClient httpClient = new HttpClient();
httpClient.getRequestListners().add(
	JettyClientMetrics
	.builder(
  	registry,
  	result -> {
      String path = result.getRequest().getURI().getPath();
      if (path.startsWith("/api/customer/")) {
        return "/api/customer/{id}";
      }
      ...
    }
  )
	.build()
);
```



## 2.15 조율된 누락

```
일반적인 HTTP 성능 테스트를 진행할 때, 일반적인 벤치마크는 웹 서버에서 요청을 처리한 시간을 먼저 저장하고, 클라이언트로 전달되기 직전 시간(응답 시간) 사이의 경과된 시간을 측정한다. 하지만, 웹 서버는 서버에서의 지연 시간(latency)만을 측정하지 그 이후는 측정되지 않는다. 
```

조율된 누락은 다음과 같이 여러 원인으로 발생한다.

### 서버리스 함수

서버리스 함수의 실행 시간을 측정할 때는 함수 시작까지 걸리는 시간을 기록하지 않는다.

### 일시정지

GC로 인한 JVM 중단, 데이터베이스 재인덱싱으로 인한 일시적 응답 불가, 캐시 버퍼를 플러시하는 순간 등..

### 부하 테스트

부하 테스트 도구는 서비스가 실제로 포화되기 전에 자신의 스레드 풀에 작업을 백업한다..(????) 



## 2.16 부하테스트

아파치 벤치 같은 테스트 도구는 지정한 속도로 요청을 만들어낸다. 그래서 응답이 수집 버킷 구간을 벗어나면 다음 요청이 늦어지게 된다.

이런 테스트 도구들은 응답 시간이 길어지면 의도치 않게 조정되고 부하 테스트를 무력화한다. 그러나 실제 사용자는 이렇게 조정되지 않고, 서비스를 훨씬 더 포화 시킬 수 있다. Gatling, JMeter는 논블로킹 방식으로 서비스를 포화시킨다.

블로킹방식의 부하테스트에 비해 논블로킹방식의 부하테스트는 레이턴시도 높게 측정되고, 최대 레이턴시도 눈에 띄게 높음을 볼 수 있다. 

프로덕션 환경에서 경고 발생 구간을 모니터링하기 위해서 처리량이나 레이턴시를 관찰 할 때는 가능하면 클라이언트 관점에서 본 상태로 함께 모니터링하는 것이 좋다.



## 2.17 미터 필터

MeterFilter javadoc - https://www.javadoc.io/doc/io.micrometer/micrometer-core/latest/io/micrometer/core/instrument/config/MeterFilter.html

어플리케이션의 여러 부분에 측정 요소가 추가될 수록 메트릭의 종류와 정확성(?)을 제어할 필요성이 커진다. 측정의 충실도(이게 무슨 뜻인지?)를 높여야하지만 누군가가 이러한 충실도를 자신의 목적에 맞게 저하시키는 것도 대비해야 한다.

미터 필터를 사용하면 미터가 등록되는 방식, 시기, 발행 통계의 종류를 설정하고 높은 수준으로 제어할 수 있다.

1. 거부: 미터 등록을 거부하거나 허용한다.
2. 변형: 미터 ID를 변형(이름변경, 태그 추가/제거, 기본 단위 설명 추가)
3. 설정: 일부 미터 타입의 분포 통계를 설정한다.

### 2.17.1 미터 거부/허용

```java
MeterFilter filter = new MeterFilter() {
  @Override
  public MeterFilerReply accept(Meter.Id id) {
    if (id.getName().contains("test")) {
      return MeterFilterReply.DENY;
    }
    return MeterFilterReply.NEUTRAL;
  }
}
```



MeterFilterReply는 DENY, NEUTRAL, ACCEPT 세가지 상태가 있다. (https://javadoc.io/doc/io.micrometer/micrometer-core/latest/io/micrometer/core/instrument/config/MeterFilterReply.html)

미터 필터는 레지스트리 설정에 추가된 순서대로 적용된다. 예를 들어 다음 예시는 http로 시작하는 메트릭은 허용하고 나머지는 모두 거부한다.

```java
registry.config()
  .meterFilter(MeterFilter.acceptNameStartsWith("http"))
  .meterFilter(MeterFilter.deny());
```

### 2.17.2 메트릭 변형

미터 필터는 미터의 이름, 태그, 설명, 기본 단위를 변환할 수 있다. 가장 일반적인 사용법은 애플리케이션 공통 태그 추가다.

### 2.17.3 분포 통계 설정

필터를 이용해서 분포 통계 정보도 선택적으로 추가할 수 있다. 



## 2.18 플랫폼과 애플리케이션 메트릭 분리

넷플릭스의 운영 엔지니어링 조직을 예시로 설명. 여기서는 플랫폼 팀과 애플리케이션 팀으로 나누어서 설명한다.

플랫폼팀은 프로세서, 메모리, API 에러율 같은 메트릭을 담당.

애플리케이션 팀은 고객지표를 담당한다.

**플랫폼 엔지니어**는 조직 전체에 존재하는 모든 마이크로서비스를 동일한 방식으로 모니터링하기를 원한다. 모든 마이크로서비스들은 플랫폼 엔지니어가 모니터링할 메트릭을 게시해야한다. 그리고 일관적인 형태로 공통 태그를 부여해야한다.

**프로덕트 엔지니어**는 프로덕션과 관련된 마이크로서비스만 모니터링한다. 위에서 플랫폼 엔지니어가 적용한 공통 태그도 물론 도움이 되긴하지만, 개별 인스턴스를 좀 더 차별화할 공통 태그를 추가하기를 원한다. 프로덕트 엔지니어는 최종 사용자 경험을 구체적으로 이해하는데 가장 집중해야한다.

이렇게 각 엔지니어들이 집중하고 있는 메트릭이 다르기 때문에, 이를 서로 침해하지 않으려면 고유한 미터 레지스트리로 메트릭을 분할 게시해야 한다.

## 2.19 모니터링 시스템에 따른 메트릭 분할

만약 프로메테우스를 기본 모니터링 시스템으로 채택했다고 해보자. 그런데 이 프로메테우스가 모든 서비스 구성요소들을 스크래핑하고 있는지를 어떻게 확인할 수 있을까?

그러면 이런 상황에서 AWS CloudWatch에 동시에 메트릭을 게시한다. 그러나 CloudWatch는 구성된 프로메테우스가 제대로 지표를 수집하고 있는지 확인하기 위해서 사용한다고 했을 때, 마이크로미터로 프로메테우스가 스크래핑을 시도하는 횟수를 기록하고, 이 지표를 클라우드와치에서 확인하면서 프로메테우스가 제대로 동작하고 있는지를 확인할 수 있다.

```java
@Bean
MeterRegistryCustomizer<CloudWatchMeterRegistry> cloudWatchCustomizations() {
  return registry -> registry.config()
    											.meterFilter(MeterFilter.acceptNameStartsWith("prometheus"))
    											.meterFilter(MeterFilter.deny());
}
```

## 2.20 미터 바인더

하위 시스템이나 라이브러리를 모니터링 하면 한개 이상의 미터를 사용한다. 마이크로미터는 이 여러 미터를 한 곳에 캡슐화 하도록 고안된 MeterBinder라는 인터페이스를 제공한다.

예를 들어 미터 바인더에 등록된 모든 메트릭에 "vehicle" 처럼 공통 접두어를 선정해서 공유하면 활용성이 높아질 수 있다. 공통 접두어가 있으면 거부 미터 필터를 손쉽게 일괄 적용할 수도 있다.

MeterBinder javadoc - https://www.javadoc.io/doc/io.micrometer/micrometer-core/latest/io/micrometer/core/instrument/binder/MeterBinder.html
