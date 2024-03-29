# SRE-with-Java

## 3.1 관찰 가능성의 세 주축과 두 분류

로그, 분산 추적, 메트릭을 주축으로 형성된다.

각자 고유한 특성을 지닌 텔레메트리 형태지만 용도는 1. 가용성 증명, 2. 근원 식별 디버깅(?) 으로 나뉜다.

### 3.1.1 로그

모든 소프트웨어 스택에 존재한다. 로그의 양은 시스템의 처리량에 비례한다. 로그로 측정하는 코드 실행 경로가 많을 수록 더 많은 로그 데이터가 발행된다.

로그의 존재 의의는 명백히 디버깅에 있다. 로그 데이터를 유지하고, 지속적으로 집계하고 작업 페이로드를 할당하면 그에 상응하는 비용이 발생한다.

### 3.1.2 분산 추적

분산 추적 시스템은 사용자와 시스템 사이의 상호 작용을 시작점부터 끝점까지 관찰한다. 따라서 특정 요청이 성능 저하를 발생시키면 엔드 투 엔드 추적 관찰을 통해 시스템의 어떤 부분이 저하되었는지 확인할 수 있다.

원격 추적은 일반적인 로그보다 샘플링 비율이 높다. 

### 3.1.3 메트릭

메트릭은 개별적인 상호작용의 정보를 얻기보다는 주로 서비스 수준 지표를 이해하는 용도로 쓰이고, 전체적으로 집계된 결과를 나타낸다. 기존 시스템에 메트릭 측정을 탑재하려면 부분적으로 수작업이 필요하다. 그 외에는 프레임워크와 라이브러리가 기본 제공하는 측정 기능을 활용한다.

## 4.7 모든 자바 마이크로 서비스에 통용되는 서비스 수준 지표

### 4.7.1 에러

코드 블록 측정 결과를 성공과 실패로 구분 시 두 가지 효과가 생긴다.

1. 전체 실행 중 실패한 실행의 비중이 곧 시스템에 발생한 에러의 빈도가 된다.
2. 시스템의 레이턴시를 지나치게 비관적으로 보게된다. (전체 흐름의 초기에 발생한 에러와 마지막쯤 발생한 에러의 레이턴시는 분명히 다르지만, 동일하게 실패로 측정될 것)

### 4.7.2 레이턴시 



## 5.7 블루/그린 배포

블루/그린 배포 전략을 수립하려면 최소 두 개의 마이크로서비스 복사본을 프로비저닝 해야한다. 블루/그린 배포 전략은 일반적으로 클러스터에 1:N 관계로 형성된다. 활성 서버 그룹 하나와 N개의 비활성 서버 그룹의 관계다.

서버 그룹의 규모에 따라 롤백 속도와 운영 비용이 달라지기 때문에, 절충점을 가늠해야한다.

## 5.8 카나리 분석 자동화

..