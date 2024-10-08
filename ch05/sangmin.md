# Chapter 5. 트랜잭션과 잠금
## 5.1 트랜잭션
### 5.1.1 MySQL에서의 트랜잭션
트랜잭션은 하나의 논리적인 작업 셋에 하나의 쿼리가 있든 두 개 이상의 쿼리가 있든 관계없이 논리적인 작업 셋 자체가 100% 적용되거나 아무거나 적용되지 않아야 함을 보장해 주는 것이다.

InnoDB에서는 트랜잭션을 지원하지만, MyISAM과 메모리 스토리지 엔진에서는 트랜잭션을 지원하지 않는다.
### 5.1.2 주의사항
트랜잭션 또한 DBMS의 커넥션과 동일하게 꼭 필요한 최소의 코드에만 적용하는 것이 좋다.

다음과 같이 사용자가 게시판에 게시물을 작성한 후 저장 버튼을 클릭하여 서버에 처리한다고 가정해본다.
```text
1) 처리 시작
    ==> 데이터베이스 커넥션 생성
    ==> 트랜잭션 시작
2) 사용자의 로그인 여부 확인
3) 사용자의 글쓰기 내용의 오류 여부 확인
4) 첨부로 업로드된 파일 확인 및 저장
5) 사용자의 입력 내용을 DBMS에 저장
6) 첨부 파일 정보를 DBMS에 저장
7) 저장된 내용 또는 기타 정보를 DBMS에서 조회
8) 게시물 등록에 대한 알림 메일 발송
9) 알림 메일 발송 이력을 DBMS에 저장
    ==> 트랜잭션 종료(commit)
    ==> 데이터베이스 커넥션 반납
10) 처리 완료
```

### 개선점
1. 실제로 DBMS에 데이터를 저장하는 작업은 5번부터 시작하기 때문에, 1번부터 4번까지의 작업은 트랜잭션에 포함시킬 필요가 없다.
2. 더 큰 위험은 8번 작업이다. 메일 전송이나 FTP 파일 전송 작업 또는 네트워크를 통해 원격 서버와 통신하는 등과 같은 작업은 어떻게 해서든 DBMS의 트랜잭션 내에서 제거하는 것이 좋다.
3. 사용자가 입력한 정보를 저장하는 5번과 6번 작업은 반드시 하나의 트랜잭션으로 묶어야 하며, 7번 작업은 단순 확인 및 조회이므로 트랜잭션에 포함할 필요는 없다.
    9번 작업도 별개의 트랜잭션으로 묶는다.

개선된 트랜잭션 처리 절차
```text
1) 처리 시작
2) 사용자의 로그인 여부 확인
3) 사용자의 글쓰기 내용의 오류 여부 확인
4) 첨부로 업로드된 파일 확인 및 저장
    ==> 데이터베이스 커넥션 생성
    ==> 트랜잭션 시작
5) 사용자의 입력 내용을 DBMS에 저장
6) 첨부 파일 정보를 DBMS에 저장
    ==> 트랜잭션 종료(commit)
7) 저장된 내용 또는 기타 정보를 DBMS에서 조회
8) 게시물 등록에 대한 알림 메일 발송
    ==> 트랜잭션 시작
9) 알림 메일 발송 이력을 DBMS에 저장
    ==> 트랜잭션 종료(commit)
    ==> 데이터베이스 커넥션 반납
10) 처리 완료
```

## 5.2 MySQL 엔진의 잠금
MySQL에서 사용되는 잠금은 크게 스토리지 엔진 레벨과 MySQL 엔진 레벨로 나눌 수 있다. 
MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치지만, 스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향을 미치지는 않는다.
MySQL 엔진에서는 테이블 데이터 동기화를 위한 테이블 락 이외에도 테이블의 구조를 잠그는 메타데이터 락 그리고 사용자의 필요에 맞게 사용할 수 있는 네임드 락이라는 잠금 기능도 제공한다.

### 5.2.1 글로벌 락
글로벌 락은 `FLUSH TABLES WITH READ LOCK` 명령으로 획득할 수 있으며, MySQL에서 제공하는 잠금 가운데 가장 범위가 크다.
글로벌 락이 영향을 미치는 범위는 MySQL 서버 전체이며ㅡ 작업 대상 테이블이나 데이터베이스가 다르더라도 동일하게 영향을 미친다.

InnoDB에서는 글로벌 락 보다 가벼운 백업락을 제공해 준다. 백업락을 획득하면 모든 세션에서 테이블의 스키마나 사용자의 인증 관련 정보를 변경할 수는 없지만, 일반적인 테이블의 데이터 변경은 허용한다.

### 5.2.2 테이블 락
테이블 락은 개별 테이블 단위로 설정되는 잠금이며, 명시적 또는 묵시적으로 특정 테이블의 락을 획득할 수 있다.

명시적으로는 `LOCK TABLES table_name [READ | WRITE]` 명령을 사용하고, 묵시적인 테이블 락은 쿼리가 실행되는 동안 자동으로 획득됐다가 쿼리가 완료된 후 자동 해제된다.

InnoDB 테이블에도 테이블 락이 설정되지만 레코드 기반의 잠금을 제공하기 때문에 대부분의 데이터 변경(DML) 쿼리에서는 무시되고 스키마를 변경하는 쿼리(DDL)의 경우에만 영향을 미친다.

### 5.2.3 네임드 락
네임드 락은 GET_LOCK() 함수를 이용해 임의의 문자열에 대해 잠금을 설정할 수 있다.

네임드 락은 단순히 사용자가 지정한 문자열에 대해 획득하고 반납하는 잠금이기 때문에 자주 사용되지는 않는다.

배치 프로그램처럼 한꺼번에 많은 레코드를 변경하는 쿼리 작업을 수행할 때, 동일 데이터를 변경하거나 참조하는 프로그램끼리 분류해서 네임드 락을 걸고 쿼리를 실행하면 데드락을 방지할 수 있다.

### 5.2.4 메타데이터 락
메타데이터 락은 데이터베이스 객체의 이름이나 구조를 변경하는 경우에 획득하는 잠금이다.

메타데이터 락은 명시적으로 획득하거나 해제할 수 있는 것이 아니고 "RENAME TABLE tab_a TO tab_b"와 같이 테이블의 이름을 변경하는 경우 자동으로 획득하는 잠금이다.

## 5.3 InnoDB 스토리지 엔진 잠금
InnoDB 스토리지 엔진은 MySQL에서 제공하는 잠금과는 별개로 스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재하고 있다.
InnoDB는 레코드 기반의 잠금 방식 때문에 MyISAM보다는 훨씬 뛰어난 동시성 처리를 제공할 수 있다.

과거 InnoDB는 이원화된 잠금 처리 탓에 MySQL 명령을 이용해 접근하기 까다로웠지만 최근 버전에서는 InnoDB의 트랜잭션과 잠금, 그리고 잠금 대기 중인 트랜잭션의 목록을 조회할 수 있는 방법이 도입되었다.
또한 InnoDB의 잠금에 대한 모니터링도 더 강화되면서 Performance Schema를 이용해 InnoDB 스토리지 엔진의 내부 잠금(세마포어)에 대한 모니터링 방법도 추가됐다.

### 5.3.1 InnoDB 스토리지 엔진의 잠금
InnoDB 스토리지 엔진은 레코드 기반의 잠금 기능을 제공하며, 잠금 정보가 상당히 작은 공간으로 관리되기 때문에 레코드 락이 페이지 락으로, 또는 테이블 락으로 레벨업되는 경우는 없다.

일반 상용 DBMS와 조금 다르게 InnoDB 스토리지 엔진에서는 레코드 락뿐 아니라 레코드와 레코드 사이의 간격을 잠그는 갭락이라는 것이 존재한다.

![image](https://github.com/user-attachments/assets/b1c10b61-1896-42af-bf5d-ac0e2a7c64cd)

### 5.3.1.1 레코드 락
다른 DBMS의 레코드락과 다르게 InnoDB 스토리지 엔진은 레코드 자체가 아니라 인덱스의 레코드를 잠근다.

### 5.3.1.2 갭 락
갭 락은 레코드 자체가 아니라 레코드와 바로 인접한 레코드 사이의 간격만을 잠그는 것을 의미한다.
갭 락의 역할은 레코드와 레코드 사이의 간격에 새로운 레코드가 생성되는 것을 제어하는 것이다. 갭 락은 그 자체보다는 이어서 설명할 넥스트 키 락의 일부로 자주 사용된다.

### 5.3.1.3 넥스트 키 락
레코드 락과 갭 락을 합쳐 놓은 형태의 잠금을 넥스트 키 락이라고 한다.

Statement 포맷의 바이너리 로그를 사용하는 MySQL 서버에서는 REPEATABLE READ 격리 수준을 사용해야한다. 
또한 innodb_locks_unsafe_for_binlog 변수가 비활성화되면 변경을 위해 검색하는 레코드에는 넥스트 키 락 방식으로 잠금이 걸린다.
갭락과 넥스트키 락은 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주 목적이다.

넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 하는 경우가 자주 발생하니 바이너리 로그 포맷을 ROW 형태로 바꿔서 넥스트 키 락과 갭 락을 줄이는 것이 좋다.

### 바이너리 로그의 3가지 종류 
- Statement-based logging

Insert, Update, Delete 에 대한 SQL 문들이 포함된다. Statement base 로 복제를 수행 시 Statement-Based Replication (SBR) 이라고 한다.

이 방식에서는 로그 파일에 적은 데이터가 기록되며, 로그 파일에 필요한 저장 공간이 줄어드는 장점이 있다.

백업에 대한 복구는 Replay 처럼 수행되며 빠르게 복원이 수행될 수 있다.

다만 로그 기반으로 복원 시 제약이 조금 있는데, 예를들어 RAND(), LOAD_FILE, UUID() 등과 같이 Deterministic 하지 않은 동작에 대해서는 정확히 복제가 안된다고 보면 된다. (당연히 RAND() 를 복원해서 나온 두 번의 결과가 같을리는 만무하다!)



- Row-based logging

이 방식은 각 행에 대한 변화를 기록한다. Row-based logging 을 이용해서 Primary → Secondary 로 복제를 수행할 수 있고, 이를 Row-Based Replication(RBR)이라 한다

RBL 의 경우 각 행의 변경 사항을 이진 로그에 기록하므로 로그 파일의 크기가 매우 빠르게 증가할 수 있으며 지연이 발생할 수도 있다.

또한, 임시 테이블의 경우 RBL 기반으로 복제되지 않으므로 임시 테이블 관련 구문은 Statement base 로 기록되어야 한다.



- 혼합형

MySQL 5.1 이상부터 Row-based logging 과 함께 혼합형태가 지원되어진다. 기본적으로는 Statement based logging 을 사용하지만, 스토리지 엔진 및 특정 명령문에 따라 로그가 자동으로 Row based logging 으로 기록되어진다.

물론 사용하는 개발자 입장에서는 큰 차이가 느껴지지는 않는다.

### 5.3.1.4 자동 증가 락
MYSQL에서는 자동 증가하는 숫자 값을 추출하기 위해 AUTO_INCREMENT 라는 칼럼 속성을 제공한다.
중복되지 않고 저장된 순서대로 증가하는 일련번호 값을 가지기 위해 내부적으로 AUTO_INCREMENT 락이라고 하는 테이블 수준의 잠금을 사용한다.

AUTO_INCREMENT 락은 INSERT와 REPLACE 쿼리 문장과 같이 새로운 레코드를 저장하는 쿼리에서만 필요하며, UPDATE나 DELETE와 같이 기존 레코드를 변경하는 쿼리에서는 사용되지 않는다.

### 5.3.2 인덱스와 잠금
InnoDB의 잠금은 레코드를 잠그는 것이 아니라 인덱스를 잠그는 방식으로 처리한다. 즉, 변경해야 할 레코드를 찾기 위해 검색한 인덱스의 레코드를 모두 락을 걸어야 한다.

때에 따라서 불필요한 레코드를 모두 락을 걸어야 하는 경우가 발생할 수 있으므로, 이런 상황이 발생되지 않도록 인덱스를 설계하는 것이 중요하다.

### 5.3.3 레코드 수준의 잠금 확인 및 해제
레코드 수준 잠금은 레코드 각각에 잠금이 걸리므로 그 레코드가 자주 사용되지 않는다면 오랜 시간 동안 잠겨진 상태로 남아 있어도 잘 발견되지 않는다.

예전 버전의 MySQL 서버에서는 레코드 잠금에 대한 메타 정보를 제공하지 않기 때문에 더더욱 어려운 부분이다. 
하지만 MySQL 5.1 부터는 레코드 잠금과 잠그 대기에 대한 조회가 가능 하므로 쿼리 하나만 실행 해 보면 잠금과 잠금 대기를 바로 확인할 수 있다.

MySQL 5.1 부터는 information_schema라는 DB에 INNODB_TRX라는 테이블과 INNODB_LOCKS, INNODB_LOCK_WAITS라는 테이블을 통해 확인 가능하다.
하지만, MySQL8.0버전 부터는 informaiton_schema의 정보들은 조금씩 deprecated 되었으며, Performance Schema의 data_locks와 data_lock_waits 테이블로 대체되고 있다.

## 5.4 MySQL의 격리 수준

|                  | DIRTY READ | NON-REPEATABLE READ | PHANTOM READ  |
|------------------| --- | --- |---------------|
| READ UNCOMMITTED | O | O | O             |
| READ COMMITTED   | X | O | O             |
| REPEATABLE READ  | X | X | O(innoDB는 없음) |
| SERIALIZABLE     | X | X | X             |

### 5.4.1 READ UNCOMMITTED
READ UNCOMMITTED 격리 수준은 가장 낮은 격리 수준이다. 이 격리 수준에서는 다른 트랜잭션이 변경 중인 데이터를 읽을 수 있다. 이러한 현상을 `DIRTY READ`라고 한다.

### 5.4.2 READ COMMITTED
READ COMMITTED 격리 수준은 READ UNCOMMITTED 격리 수준보다 높은 격리 수준이다. 오라클 DBMS의 기본 격리 수준이며, 온라인 서비스에서 가장 많이 선택되는 격리 수준이다.
이 격리 수준에서는 다른 트랜잭션이 변경 중인 데이터를 읽을 수 없다. 이러한 현상을 `NON-REPEATABLE READ`라고 한다.

### 5.4.3 REPEATABLE READ
REPEATABLE READ 격리 수준은 READ COMMITTED 격리 수준보다 높은 격리 수준이다. MySQL의 InnoDB 스토리지 엔진에서 기본으로 사용되는 격리 수준이다.
InnoDB 스토리지 엔진은 트랜잭션이 ROLLBACK될 가능성에 대비해 변경되기 전 레코드를 언두 공간에 백업해두고 실제 레코드 값을 변경한다. 이러한 방식을 MVCC(Multi-Version Concurrency Control)라고 한다.
이 격리 수준에서는 같은 쿼리를 실행해도 결과가 항상 같다. 이러한 현상을 `PHANTOM READ`라고 한다.


### 5.4.4 SERIALIZABLE
SERIALIZABLE 격리 수준은 가장 높은 격리 수준이다. 이 격리 수준에서는 트랜잭션 간에 완벽하게 격리되어 있어, 동시성 처리가 떨어지는 단점이 있다.
InnoDB 스토리지 엔진에서는 갭 락과 넥스트 키 락 덕분에 REPEATABLE READ 격리 수준에서도 PHANTOM READ 현상이 발생하지 않기 때문에 SERIALIZABLE 격리 수준을 사용할 필요가 없다.