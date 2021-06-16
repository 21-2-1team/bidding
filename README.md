![00  표지](https://user-images.githubusercontent.com/84000922/122162379-748d9800-ceae-11eb-9825-9472c08d9cc1.png)



# 서비스 시나리오

### 기능적 요구 사항

```
• 입찰담당부서는 입찰공고를 등록한다.
• 입찰공고가 등록되면 입찰공고가 접수된다.
• 조달업체는 입찰서를 등록한다.
• 입찰서가 등록되면 입찰심사가 접수(등록)된다.
• 심사부서는 심사결과를 등록한다.
• 심사결과가 등록되면 입찰공고에 낙찰자 정보가 등록(공지)된다.
• 입찰담당부서는 입찰공고를 취소 할 수 있다.
• 입찰공고가 취소되면 입찰서 등록 및 심사결과도 취소된다.
• 입찰서 등록, 입찰서 등록 취소, 낙찰자 등록 시 조달업체 담당자에게 SMS를 발송한다.
• 조달업체는 입찰현황을 조회 할 수 있다.
※ 위 시나리오는 가상의 절차로, 실제 업무와 다를 수 있습니다.
```

### 비기능적 요구 사항

```
1. 트랜잭션
  - 심사결과가 등록되면 입찰공고에 낙찰자 정보가 등록되어야 한다. (Sync 호출)
2. 장애격리
  - 입찰심사 기능이 수행되지 않더라도 입찰관리, 입찰참여 기능은 365일 24시간 받을 수 있어야 한다. Async (event-driven), Eventual Consistency
  - 입찰참여 기능이 과중되면 사용자를 잠시 동안 받지 않고 예약을 잠시후에 하도록 유도한다. Circuit breaker, fallback
3. 성능
  - 조달업체는 입찰현황조회 화면에서 입찰 상태를 확인 할 수 있어야 한다.CQRS - 조회전용 서비스
```

### Microservice명

```
입찰관리 – BiddingManagement
입찰참여 - BiddingParticipation
입찰심사 - BiddingExamination
```



# 분석/설계

### AS-IS 조직 (Horizontally-Aligned)

![1  AS-IS조직](https://user-images.githubusercontent.com/84000922/122162394-7b1c0f80-ceae-11eb-95c4-8952596bb623.png)




### TO-BE 조직 (Vertically-Aligned)

![2  TO-BE 조직](https://user-images.githubusercontent.com/84000922/122162398-7c4d3c80-ceae-11eb-88b9-863f1e58ba41.png)






### 이벤트 도출

![3  이벤트 도출](https://user-images.githubusercontent.com/84000922/122162410-7fe0c380-ceae-11eb-9822-adb2c3b8d62a.png)




### 부적격 이벤트 탈락

![4  부적격 이벤트 탈락](https://user-images.githubusercontent.com/84000922/122162412-7fe0c380-ceae-11eb-8aba-20f04b2a4dbb.png)

```
- 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행
- 이의제기등록됨 : 후행 시나리오라서 제외
- 가격심사점수등록됨 : 속성 정보여서 제외
- 입찰공고메뉴선택됨, 입찰현황조회됨 : UI 의 이벤트이지, 업무적인 의미의 이벤트가 아니라서 제외 
```




### 액터, 커맨드 부착하여 읽기 좋게

![5  액터, 커맨드 부착하여 읽기 좋게](https://user-images.githubusercontent.com/84000922/122162413-80795a00-ceae-11eb-9b06-668274f351f7.png)




### 어그리게잇으로 묶기

![6  어그리게잇으로 묶기](https://user-images.githubusercontent.com/84000922/122162415-80795a00-ceae-11eb-8b57-846e1779e420.png)

```
- 입찰관리, 입찰참여, 입찰심사는 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌
```




### 바운디드 컨텍스트로 묶기

![7  바운디드 컨텍스트로 묶기](https://user-images.githubusercontent.com/84000922/122162416-8111f080-ceae-11eb-87be-10c03082eab2.png)

```
도메인 서열 분리
- Core Domain: 입찰관리, 입찰참여: 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 입찰관리배포주기는 1개월 1회 미만, 입찰참여 배포주기는 1주일 1회 미만
 - Supporting Domain: 입찰심사 : 경쟁력을 내기 위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함. 
- General Domain: Notification : 알림서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)
```




### 폴리시 부착, 이동 및 컨텍스트 매핑(점선은 Pub/Sub, 실선은 Req/Resp)

![8  폴리시 부착, 이동 및 컨텍스트 매핑](https://user-images.githubusercontent.com/84000922/122162417-8111f080-ceae-11eb-8b7d-12e4362ea464.png)




### 완성된 1차 모형

![9  완성된 1차 모형](https://user-images.githubusercontent.com/84000922/122162419-81aa8700-ceae-11eb-95cb-183ad9e5706f.png)




### 1차 완성본에 대한 기능적 요구사항을 커버하는지 검증 (1/2)

![10  1차 완성본에 대한 기능적](https://user-images.githubusercontent.com/84000922/122162421-82431d80-ceae-11eb-8a7d-18df4aff613a.png)

```
1) 입찰담당부서는 입찰공고를 등록한다. 입찰공고가 등록되면 입찰공고가 접수된다.
2) 조달업체는 입찰서를 등록한다. 입찰서가 등록되면 입찰심사가 접수(등록)된다.
3) 심사부서는 심사결과를 등록한다. 심사결과가 등록되면 입찰공고에 낙찰자 정보가 등록(공지)된다.
```




### 1차 완성본에 대한 기능적 요구사항을 커버하는지 검증 (2/2)

![11  1차 완성본에 대한 기능적 요구사항](https://user-images.githubusercontent.com/84000922/122162422-82431d80-ceae-11eb-9645-c57cd204c18e.png)

```
1) 입찰담당부서는 입찰공고를 취소 할 수 있다. 
   입찰공고가 취소되면 입찰서 등록 및 심사결과도 취소된다.
2) 입찰서 등록, 입찰서 등록 취소, 낙찰자 등록 시 조달업체 담당자에게 SMS를 발송한다.
3) 조달업체는 입찰현황을 조회 할 수 있다.
```




### 1차 완성본에 대한 비기능적 요구사항을 커버하는지 검증

![12  1차 완성본에 대한 비기능적](https://user-images.githubusercontent.com/84000922/122162424-82dbb400-ceae-11eb-92d5-9f938ac7d6cf.png)

```
1. 트랜잭션
  - 심사결과가 등록되면 입찰공고에 낙찰자 정보가 등록되어야 한다. (Sync 호출)
2. 장애격리
  - 입찰심사 기능이 수행되지 않더라도 입찰관리, 입찰참여 기능은 365일 24시간 받을 수 있어야 한다. 
    Async (event-driven), Eventual Consistency
  - 입찰참여 기능이 과중되면 사용자를 잠시 동안 받지 않고 예약을 잠시후에 하도록 유도한다.
    Circuit breaker, fallback
3. 성능
  - 조달업체는 입찰현황조회 화면에서 입찰 상태를 확인 할 수 있어야 한다.CQRS - 조회전용 서비스
```




### 헥사고날 아키텍처 다이어그램 도출

![13  헥사고날 아키텍처 다이어그램 도출](https://user-images.githubusercontent.com/84000922/122162425-82dbb400-ceae-11eb-9e47-eef31b055935.png)




### Git Organization / Repositories

![14  Git Organization Repositories](https://user-images.githubusercontent.com/84000922/122162427-83744a80-ceae-11eb-987e-b214c71f4213.png)
