# 단일 PC 모델: MySQL Server + On-premises Data Gateway + Power Platform 연동 가이드

### 작성자: Microsoft PowerPlatform SE 이영서
### 마지막 수정일: 260401

> **목적**: Windows **단일 PC 1대**에 **SQL Server**와 **On-premises Data Gateway**를 함께 설치하고, 이를 **Power Automate / Copilot Studio / Power Apps** 같은 **Power Platform** 서비스에서 사용하는 전체 절차를 정리합니다.

> **참고 자료**:
> - https://learn.microsoft.com/en-us/power-platform/admin/wp-onpremises-gateway
> - https://learn.microsoft.com/ko-kr/copilot/finance/get-started/sap/administer/set-up-gateway
> - https://learn.microsoft.com/en-us/analysis-services/azure-analysis-services/analysis-services-gateway?view=sql-analysis-services-2025
> - MySQL 관련: https://learn.microsoft.com/en-us/connectors/mysql/
> - Power Automate에서 MySQL Connection 관련: https://learn.microsoft.com/en-us/power-automate/add-manage-connections


## 1. 개요

이 문서는 SQL 서버와 On Premises Gateway가 단일 PC에 설치된 **단일 PC(All-in-One) 모델**만 다룹니다.

구조는 아래와 같습니다.

```text
Power Automate / Copilot Studio / Power Apps (Cloud)
                ↓ (요청)
      On-premises Data Gateway (같은 PC)
                ↓ (로컬 접속: localhost:3306)
             MySQL Server (같은 PC)
```

핵심은 다음과 같습니다.

- 클라우드 서비스가 SQL Server에 **직접 접속하지 않습니다**.
- **같은 PC에 설치된 Gateway**가 클라우드 요청을 받아 로컬 SQL Server에 접속합니다.
- 게이트웨이는 저장된 데이터 원본 자격 증명(credentials)으로 SQL Server 작업을 실행합니다.
- 게이트웨이는 **인바운드 포트를 열지 않고**, Azure Service Bus / Relay와 **아웃바운드 연결만 유지**합니다.



## 2. 이 모델이 적합한 경우

이 구성은 다음 용도에 적합합니다.

- 빠른 PoC
- 랩 / 데모
- 네트워크 변수를 최소화한 테스트
- 온프렘 게이트웨이 동작 원리 학습

장점:

- VM 간 사설 IP, NSG, 서브넷, 라우팅 이슈가 없음
- DB와 Gateway가 같은 머신이므로 구성 단순
- SQL Server 연결 시 `localhost` 또는 `127.0.0.1` 사용 가능

단점:

- 단일 장애 지점(SPOF)
- 운영(Production) 환경에는 비권장


## 3. 사전 요구 사항

### 3.1 계정

- Power Platform 테넌트 계정 
- SQL Server를 설치 / 관리할 로컬 관리자 권한

### 3.2 필수 원칙

- Gateway는 **Standard mode**로 설치



## 4. SQL Server 설치 및 준비


### 4.1 단일 PC에 MySQL을 설치합니다.
- https://www.mysql.com/downloads/


### 4.2 테스트용 DB / 로그인 / 사용자 생성

아래 예시는 **CompanyDB** 데이터베이스와 **pa_user** 로그인 / 사용자를 만드는 예시입니다.

```sql
-- 1) 테스트용 DB 생성
CREATE DATABASE CompanyDB;
GO

-- 2) DB 선택
USE CompanyDB;
GO

-- 3) 샘플 테이블 생성
CREATE TABLE dbo.OrgInfo (
    Id INT PRIMARY KEY,
    OrgName NVARCHAR(200) NOT NULL
);
GO

-- 4) 샘플 데이터 입력
INSERT INTO dbo.OrgInfo (Id, OrgName)
VALUES (1, N'초기값');
GO

-- 5) SQL 로그인 생성
USE master;
GO
CREATE LOGIN pa_user WITH PASSWORD = 'Password123!';
GO

-- 6) DB 사용자 생성 및 매핑
USE CompanyDB;
GO
CREATE USER pa_user FOR LOGIN pa_user;
GO

-- 7) 최소 권한 부여 - 로컬 접속만 허용
GRANT SELECT, INSERT, UPDATE, DELETE ON dbo.OrgInfo TO pa_user@'localhost' ;
GO
```

> 참고
>- SQL Server는 **로그인(Login)** 과 **데이터베이스 사용자(User)** 를 분리합니다.
> - Power Platform 연결에는 `pa_user` 같은 **전용 SQL 로그인**을 따로 만드는 것이 좋습니다.



## 5. On-premises Data Gateway 설치

### 5.1 설치 모드

반드시 **Standard mode**로 설치합니다.

> Personal mode는 Power BI 전용이며, Power Automate / Power Apps / Copilot Studio 용도가 아닙니다.

### 5.2 설치 절차

1. On-premises Data Gateway 설치 파일 실행
2. **Standard mode** 선택
3. Power Platform 테넌트 계정으로 로그인 (게이트웨이가 그 테넌트에 귀속되어 있어야 Power Automate에서 보입니다)
4. 새 Gateway 이름 지정
5. **Recovery Key** 설정 및 보관

### 5.3 설치 후 확인

Power Automate에서:

- **Data > Gateways** 이동
- 방금 만든 Gateway가 **Online** 상태인지 확인

### 5.4 Region 주의

Gateway Region은 **Power Platform Environment Region과 일치**해야 합니다.

- Region이 다르면 Power Automate에서 Gateway가 안 보일 수 있음
- Region은 설치 후 변경 불가
- 틀리면 재설치 필요

## 6. MySQL Connector / NET 설치

MySQL에 접속하는 주체가 Power Automate 클라우드가 아니라 게이트웨이가 됩니다.
게이트웨이는 저장된 데이터 원본 자격 증명으로 데이터 소스에 접속하여 작업을 실행합니다.
즉, 게이트웨이가 MySQL 클라이언트처럼 동작하기에 MySQL용 드라이버 (Connector/NET)가 있어야 합니다.  (게이트웨이가 실제로 MySQL에 붙을 때 사용)

### 6.1. 아래 링크에서 설치
https://dev.mysql.com/downloads/connector/net/

커넥터 설치 후 on premises gateway 재시작을 권장합니다.
```powershell
Restart-Service -Name "On-premises data gateway service"
```


## 7. Power Automate에서 MySQL Server Connection 생성

### 7.1 메뉴 이동

- **Power Automate**
- **Data > Connections**
- **New connection**
- **MySQL** 선택

### 7.2 입력값 예시

단일 PC 구성에서는 아래처럼 입력하면 됩니다.

- **Server name**: `localhost`
  - 또는 `127.0.0.1`
  - 또는 named instance면 `localhost\INSTANCE`
- **Database**: `CompanyDB`
- **Authentication type**: 'Basic'
- **Username**: `pa_user`
- **Password**: `Password123!`
- **Gateway**: 방금 설치한 On-premises Data Gateway 선택



## 8. 연결 테스트 순서

SQL Server Connection 생성 후 아래 순서로 테스트합니다.

1. **Get rows**
2. 특정 행 조회
3. **Insert row**
4. **Update row**

예: `dbo.OrgInfo` 테이블에서 `Id = 1` 행 조회

![alt text](image.png)



## 9. 검증 시나리오 (Kill-Switch 테스트)

이 테스트는 Gateway가 실제로 **유일한 통로**인지 확인 가능합니다.

### 9.1 Gateway 서비스 중지

관리자 PowerShell에서 실행:

```powershell
Stop-Service -Name 'On-premises data gateway service'
```

### 9.2 Flow 실행

Power Automate에서 SQL Server 액션이 포함된 Flow를 실행합니다.

예상 결과:

- 연결 실패해야 정상
- 즉, 클라우드가 DB에 직접 붙는 것이 아니라 **Gateway를 통해서만** 붙는다는 뜻

### 9.3 서비스 재시작

```powershell
Start-Service -Name 'On-premises data gateway service'
```

다시 실행하면 정상 동작해야 합니다.

-

## 10. 체크리스트 요약

- [ ] MySQL Server 설치 및 예시 DB생성 
- [ ] `pa_user` SQL 로그인 생성
- [ ] DB 사용자 매핑 및 최소 권한 부여
- [ ] On-premises Data Gateway Standard mode 설치
- [ ] Power Automate에서 Gateway Online 확인
- [ ] SQL Server Connection 생성 (`localhost` / `CompanyDB` / `pa_user`)
- [ ] Get row / Update row 테스트
- [ ] Kill-Switch 테스트 통과

