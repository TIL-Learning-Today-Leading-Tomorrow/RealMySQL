# Chapter 4. 아키텍처

## 4.1 MySQL 엔진 아키텍처
### 4.1.1 MySQL의 전체 구조
![image](https://github.com/user-attachments/assets/c0be8f70-21a7-4481-afa7-b66141fab397)
MySQL 서버는 크게 `MySQL엔진`과 `스토리지 엔진`으로 구분할 수 있다. 그리고 이 둘을 모두 합쳐서 그냥 MySQL 또는 `MySQL 서버`라고 표현한다.

`MySQL 엔진`은 클라이언트로부터의 접속 및 쿼리 요청을 처리하는 커넥션 핸들러와 SQL 파서 및 전처리기, 쿼리의 최적화된 실행을 위한 옵티마이저가 중심을 이룬다.

MySQL 엔진은 요청된 SQL 문장을 분석하거나 최적화하는 등 DBMS의 두뇌에 해당되는 처리를 수행하고, 
실제 데이터를 디스크 스토리지에 저장하거나 디스크 스토리지로부터 데이터를 읽어오는 부분은 `스토리지 엔진`이 전담한다.

MySQL 엔진의 쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때는 각 스토리지 엔진에 쓰기 또는 읽기를 요청하는데, 이러한 요청을 `핸들러 요청`이라고 한다.

### 4.1.2 MySQL 스레딩 구조
MySQL 서버는 프로세스 기반이 아니라 스레드 기반으로 작동하며, 크게 포그라운드 스레드와 백그라운드 스레드로 구분할 수 있다.

- 포그라운드 스레드(클라이언트 스레드)
  - 포그라운드 스레드는 최소한 MySQL 서버에 접속된 클라이언트의 수만큼 존재하며, 주로 각 클라이언트 사용자가 요청하는 쿼리 문장을 처리한다.
  - 포그라운드 스레드는 데이터를 MySQL의 데이터 버퍼나 캐시로부터 가져오며, 버퍼나 캐시에 없는 경우에는 직접 디스크의 데이터나 인덱스 파일로부터 데이터를 읽어와서 작업을 처리한다.
- 백그라운드 스레드
  - InnoDB는 다음과 같이 여러 가지 작업이 백그라운드로 처리된다.
    - 인서트 버퍼를 병합하는 스레드
    - 로그를 디스크로 기록하는 스레드
    - InnoDB 버퍼 풀의 데이터를 디스크에 기록하는 스레드
    - 데이터를 버퍼로 읽어 오는 스레드
    - 잠금이나 데드락을 모니터링하는 스레드
  - 사용자의 요청을 처리하는 도중 데이터의 쓰기 작업은 지연되어 처리될 수 있지만 데이터의 읽기 작업은 절대 지연될 수 없다.

### 4.1.3 메모리 할당 및 사용 구조
MySQL에서 사용되는 메모리 공간은 크게 글로벌 메모리 영역과 로컬 메모리 영역으로 구분할 수 있다.
글로벌 메모리 영역은 클라이언트 스레드의 수와 무관하게 하나의 메모리 공간만 할당된다.
로컬 메모리 영역은 세션 메모리 영역이라고도 표현하며, MySQL 서버상에 존재하는 클라이언트 스레드가 쿼리를 처리하는 데 사용하는 메모리 영역이다.
로컬 메모리는 각 클라이언트 스레드별로 독립적으로 할당되며 절대 공유되어 사용되지 않는다는 특징이 있다.
- 글로벌 메모리 영역의 종류
  - 테이블 캐시
  - InnoDB 버퍼 풀
  - InnoDB 어댑티브 해시 인덱스
  - InnoDB 리두 로그 버퍼
- 로컬 메모리 영역의 종류
  - 정렬 버퍼
  - 조인 버퍼
  - 바이너리 로그 캐시
  - 네트워크 버퍼

### 4.1.4 플러그인 스토리지 엔진 모델
MySQL 서버에서는 스토리지 엔진뿐만 아니라 다양한 기능을 플러그인 형태로 지원한다.

### 4.1.5 컴포넌트
MySQL 8.0부터는 기존의 플러그인 아키텍처를 대체하기 위해 컴포넌트 아키텍처가 지원된다.
기존 플러그인은 다음과 같은 단점이 있다.
- 플러그인은 오직 MySQL 서버와 인터페이스할 수 있고, 플러그인끼리는 통신할 수 없음
- 플러그인은 MySQL 서버의 변수나 함수를 직접 호출하기 때문에 안전하지 않음
- 플러그인은 상호 의존 관계를 설정할 수 없어서 초기화가 어려움

### 4.1.6 쿼리 실행 구조
![image](https://github.com/user-attachments/assets/1b20ab1a-72be-49b2-9663-d7da438a9af8)
`쿼리 파서`는 사용자 요청으로 들어온 쿼리문장을 토큰으로 분리해 트리 형태의 구조로 만들어 내는 작업을 의미한다.

`전처리기`는 파서 과정에서 만들어진 파서 트리를 기반으로 쿼리 문장에 구조적인 문제점이 있는지 확인한다.

`옵티마이저`란 사용자의 요청으로 들어온 쿼리 문장을 저렴한 비용으로 가장 빠르게 처리할지를 결정하는 역할을 담당하며, DBMS의 두뇌에 해당한다고 볼 수 있다.

`실행 엔진`은 만들어진 계획대로 각 핸들러에게 요청해서 받은 결과를 또 다른 핸들러 요청의 입력으로 연결하는 역할을 수행한다.

`핸들러(스토리지 엔진)`은 MySQL 서버의 가장 밑단에서 MySQL 실행 엔진의 요청의 입력으로 연결하는 역할을 수행한다.

### 4.1.7 복제
MySQL 서버에서 복제는 매우 중요한 역할을 담당한다. 2대 이상의 DBMS를 나눠서 데이터를 저장하는 방식이며, 사용하기 위한 최소 구성은 Master/Slave 구성이다.

### 4.1.8 쿼리 캐시
쿼리 캐시는 빠른 응답을 필요로 하는 웹 기반의 응용 프로그램에서 중요한 역할을 담당했지만, 성능 저하 문제 때문에 MySQL 8.0부터는 완전히 제거되었다.
성능 저하의 이유는 테이블의 데이터가 변경된다면 변경된 테이블과 관련된 것들은 모두 삭제 되어야 했기 때문에 동시 처리 성능 저하와 많은 버그를 야기 했기 때문이다.

### 4.1.9 스레드 풀
스레드 풀은 내부적으로 사용자의 요청을 처리하는 스레드 개수를 줄여서 동시 처리되는 요청이 많다하더라도 MySQL 서버의 CPU가 제한된 개수의 스레드 처리에만 집중할 수 있게 해서 서버의 자원 소모를 줄이는 것이 목적이다.
스케줄링 과정에서 CPU 시간을 제대로 확보하지 못하는 경우에는 쿼리 처리가 더 느려지는 사례도 발생할 수 있다.

### 4.1.10 트랜잭션 지원 메타데이터
MySQL 5.7버전까지 메타데이터를 파일기반으로 저장해 왔다. MySQL 8.0버전부터는 테이블의 구조정보나 스토어드 프로그램의 코드 관련 정보를 모두 InnoDB의 테이블에 저장하도록 개선했다.
그래서, 이제 스키마 변경 작업 중간에 MySQL서버가 비정상적으로 종료된다고 하더라도 스키마 변경이 완전한 성공 또는 완전한 실패로 정리된다.

## 4.2 InnoDB 스토리지 엔진 아키텍처
InnoDB는 MySQL의 스토리지 엔진 가운데 가장 많이 사용되고 레코드 기반의 잠금을 제공하여 높은 동시성 처리가 가능하고 안정적이며 성능이 뛰어나다.

### 4.2.1 프라이머리 키에 의한 클러스터링
InnoDB의 모든 테이블은 기본적으로 `프라이머리 키를 기준으로 클러스터링`되어 저장된다. 프라이머리 키가 클러스터링 인덱스이기 때문에 프라이머리 키를 이용한 레인지 스캔은 상당히 빨리 처리 될 수 있다.

### 4.2.2 외래 키 지원
InnoDB 스토리지 엔진 레벨에서 `외래 키`를 지원한다. 외래 키 제약 조건을 설정하면 부모 테이블의 데이터가 변경되거나 삭제될 때 자식 테이블의 데이터도 자동으로 변경되거나 삭제된다.
그로 인해 데드락이 발생할 때가 많으므로 개발할 때도 외래 키의 존재를 주의하는 것이 좋다.

### 4.2.3 MVCC
`MVCC`는 레코드 레벨의 트랜잭션을 지원하는 DBMS가 제공하는 기능이며, MVCC의 가장 큰 목적은 잠금을 사용하지 않는 일관된 읽기를 제공하는 데 있다.
InnoDB는 `언두 로그`를 이용해 이 기능을 구현한다.
```sql
update member set m_area='경기' where m_id=12;
```
![image](https://github.com/user-attachments/assets/a0efb406-67dd-45fc-a589-259c05e91f69)

### 4.2.4 잠금 없는 일관된 읽기
InnoDB 스토리지 엔진은 MVCC 기술을 이용해 잠금을 걸지 않고 읽기 작업을 수행한다. 잠금을 걸지 않기 때문에 InnoDB에서 읽기 작업은 다른 트랜잭션이 가지고 있는 잠금을 기다리지 않고, 읽기 작업이 가능하다.

### 4.2.5 자동 데드락 감지
InnoDB 스토리지 엔진은 내부적으로 잠금이 교착 상태에 빠지지 않았는지 체크하기 위해 잠금 대기 목록을 그래프 형태로 관리한다. 
데드락 감지 스레드가 주기적으로 잠금 대기 그래프를 검사해 교착 상태에 빠진 트랜잭션들을 찾아서 그중 하나를 강제종료한다.

### 4.2.6 자동화된 장애 복구
InnoDB에는 손실이나 장애로부터 데이터를 보호하기 위한 여러 가지 메커니즘이 탑재돼 있다. 보통은 자동복구가 되지만 안되는 경우에는 `innodb_force_recovery` 시스템 변수를 설정해서 MySQL 서버를 시작해야 한다.

### 4.2.7 InnoDB 버퍼 풀
### 버퍼 풀의 크기 설정

디스크의 데이터 파일이나 인덱스 정보를 메모리에 캐시해 두는 공간이다. 쓰기 작업을 지연시켜 일괄 작업으로 처리할 수 있게 해주는 버퍼 역할도 같이한다.

운영체제와 각 클라이언트 스레드가 사용할 메모리를 충분히 고려해서 버퍼 풀의 크기를 설정해야한다. MySQL5.7부터 InnoDB 버퍼 풀의 크기를 동적으로 조절할 수 있게 개선됐다.
버퍼 풀의 크기를 크게 변경하는 작업은 서비스 영향도가 크지 않지만 버퍼 풀의 크기를 줄이는 작업은 서비스 영향도가 크기 때문에 하지 않도록 주의하자.

InnoDB 버퍼 풀은 전통적으로 버퍼 풀 전체를 관리하는 잠금(세마포어)으로 인해 내부 잠금 경합을 많이 유발해왔는데. 이런 경합을 줄이기 위해 버퍼 풀을 여러 개로 쪼개어 관리할 수 있게 개선됐다.

### 버퍼 풀의 구조
InnoDB 스토리지 엔진은 버퍼 풀이라는 거대한 메모리 공간을 페이지 크기의 조각으로 쪼개어 InnoDB 스토리지 엔진이 데이터를 필요로 할 때 해당 데이터 페이지를 읽어서 각 조각에 저장한다.
버퍼 풀의 페이지 크기 조각을 관리하기 위해 InnoDB 스토리지 엔진은 크게 `LRU 리스트`와 `플러시 리스트`, 그리고 `프리 리스트`라는 3개의 자료 구조를 관리한다.

LRU 리스트를 관리하는 목적은 디스크로부터 한 번 읽어온 페이지를 최대한 오랫동안 InnoDB 버퍼풀의 메모리에 유지해서 디스크 읽기를 최소화하는 것이다.
플러시 리스트는 디스크로 동기화되지 않은 데이터 페이지의 변경 시점 기준의 페이지 목록을 관리한다.

### 버퍼 풀과 리두 로그
InnoDB의 버퍼 풀은 디스크에서 읽은 상태로 전혀 변경되지 않은 클린 페이지와 함께 INSERT, UPDATE, DELETE 명령으로 변경된 데이터를 가진 더티페이지도 가지고 있다.
더티페이지는 디스크와 메모리의 데이터 상태가 다르기 때문에 언젠가는 디스크로 기록돼야 한다.하지만 더티 페이지는 버퍼 풀에 무한정 머무를 수 있는 것은 아니다.
데이터 변경이 발생하면 리두 로그는 1개 이상의 고정 크기 파일을 연결해서 순환 고리처럼 사용한다.

이처럼 InnoDB 버퍼 풀의 더티 페이지는 특정 리두 로그 엔트리와 관계를 가지고, 체크포인트가 발생하면 체크포인트 LSN보다 작은 리두 로그 엔트리와 관련된 더티 페이지는 모두 디스크로 동기화돼야 한다.
물론 당연히 체크포인트 LSN보다 작은 LSN 값을 가진 리두 로그 엔트리도 디스크로 동기화 돼야 한다.

## 4.3 MyISAM 스토리지 엔진 아키텍처
## 4.4 MySQL 로그 파일