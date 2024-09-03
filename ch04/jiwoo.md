# 04. Architecture
![image](https://github.com/user-attachments/assets/cad9fa33-2ca1-4df7-a896-1134bdb7dc39)
https://dev.mysql.com/doc/refman/8.4/en/pluggable-storage-overview.html

### MySQL Connectors, MySQL Shell  
: 사용자의 입장에서 접속되는 방식이 다름을 구분하여 표현한 것으로 생각된다.
그러나 DB의 입장에서는 접속하는 프로그램은 상관이 없다. 모두 각각 하나의 클라이언트, 접속 요청으로 생각될 뿐이다.

### MySQL Server Process (mysqld)  
: mysql daemon을 의미, 위 그림에서 하위의 NoSQL, SQL Interpace 등이 프로세스를 의미하는 것은 아니다. 하기에 기입된 아키텍처가 mysql 인스턴스를 구성한다는 의미로 결국 mysql instance 혹은 DB라고 부르는 대상이 된다.

### NoSQL  
: 공식문서에서 표현된 부분에서 가장 이해가 어려운 부분이다. 부가 설명이 없어서 단순하게 CRUD 작업을 처리하는 중간 매체 정도로만 이해하고 있는데 왜 이에 대한 이름은 NoSQL로 표현한 것인지 의문이다.

### SQL Interface  
: Command (사용자가 입력한 쿼리 등) 을 수신하고 이의 결과를 송신하는 인터페이스, 그림에서 DDL, DML, View, Procedures 등이라고 표현한 것은 SQL들을 의미하기에 해당 내용을 포함한 것으로 생각된다.

### Parser  
: 입력된 쿼리를 해석하는 부분(파싱 트리를 생성), 그림에서 표현된 Query Translation은 Syntax check를 의미(문법적 오류 검증), 다음으로 부가 설명없이 Object Privilege 라고 표현되어 있는데 이는 Symantic Check를 의미하는 것으로 보인다.  

    (Symantic Check 단계에서는 문법적 검증이 끝난 쿼리에 대해서 테이블, 뷰, 컬럼 등의 오브젝트가 존재하는 지, 존재하더라도 현재 접속한 유저가 해당 오브젝트에 접근할 수 있는 권한이 존재하는 지 검증하게 된다. - 다만 그림에서는 단순히 Privilege 라는 표현만 존재해서 의문)

### Optimizer  
: 모든 검증이 끝난 쿼리에 대해서 실행 계획을 수립한다.
크게 Rule based optimization 과 Cost based optimization이 존재하나 대부분의 DB가 Cost based 방식을 사용하는 것으로 알고 있다. 이 때 모든 경우의 수를 판단하여 최저 비용의 실행 계획을 채택한다.

```
cf . Oracle의 경우 실행계획이 수립된 쿼리에 대해서 해당 정보를 Library cache 라는 메모리 영역에 저장한다. 이 덕분에 동일한 쿼리(동일한 SQL_ID)의 경우 위의 파싱 및 실행계획 수립과정이 생략되는데 MySQL의 경우 Library cache 역할의 구조가 보이지 않아 의문이다.  

 과거 Query cache가 존재하였으나 이는 결과가 같이 저장되는 구조이며 성능 이슈로 인해 더 이상 사용되지 않는다. 해당 부분은 추가 확인이 필요하다.
```

### Caches & Buffers
: 위 그림에서는 cache와 buffer 가 하나의 묶음으로 존재하는 것처럼 묘사되어 있다. 아마 관련성 때문에 하나로 묶은거 같은데 cache와 buffer 는 별도로 존재하는 것으로 이해가 필요하다. 

- caches : Query cache, Thread cache, Table metadata cache 등

- buffers : redo log buffer, innodb buffer pool 등  

(InnoDB가 스토리지 엔진에 해당되기 때문에 innodb buffer pool이 스토리지 엔진 위치에 존재하는 것이 아닌가 생각될 수 있다. 이곳 저곳 찾아보았을 때 + 위 아키텍처 그림에서 묘사하는 설명을 보았을 때 innodb buffer pool은 Caches & buffers 에 포함된다. 다시 말해서 MySQL 서버의 메모리 공간에 innodb_buffer_pool 이 할당되고 스토리지 엔진인 InnoDB가 이를 관리한다고 이해하자.)

### Storage engine  
: MySQL plugin 형태로 여러 스토리지 엔진을 장착할 수 있는 것으로 보인다. 다만 여러 스토리지 엔진이 연결되었다고 해서 하나의 오브젝트가 여러 스토리지 엔진 기능을 활용하는 것은 아니고 하나의 스토리지 엔진에만 종속된다.

### File System, Files & Logs  
: 이 부분도 2개로 구분하는 것이 아닌 File system 하위에 Files, Logs를 포함해야 하는 것이 아닌가 생각이 든다. 위와 같이 표현되면 마치 별개로 존재하는 것처럼 보인다. 결국 File system 은 File 등을 관리하고 접근할 수 있게 하는 체제기 때문이다.  

어쨌든 물리적인 파일들을 의미한다고 이해하면 된다. Data file, Log file(Dump성 데이터), Redo log file 등으로 생각하자.

참고로 Redo log buffer와 Redo log file을 동일하게 생각하지 말자. 전자의 경우 메모리 상의 개념을 의미하고 후자는 메모리 상의 내용을 Disk에 작성된 파일을 의미한다.

# 동작 흐름
### 1. Connection 생성
사용자 connection 요청 발생 시 (JDBC, WAS, Tool, 서버 내부 MySQL 콘솔 접속 등)  이를 Thread 기반으로 Foregound Thread 가 1대1로 연결 (Thread pool 사용 시 다대1)  
생성된 User Thread는 사용자 권한 및 자격 검증 진행 (계정 존재, 접속 IP, 패스워드 등)

### 2. 사용자 SQL 요청

(쿼리 파서) Syntax check : SQL 문법 검증, 오타, 문법 순서 (where, group... 순서)  
(전처리기) Symantic check : 객체 검증, 객체의 존재 유무, 객체 권한 유무  
(옵티마이저) 실행계획 수립 : 모든 경우의 수를 판단 이후 가장 최소 비용의 실행계획 수립 (Cost based optimizer)  
스토리지 엔진 Handler 호출 : Memory Hit 점검 (Inno_db_buffer_pool 내 데이터 page 존재 점검)  
Hit 실패시 Disk Read 진행 후 buffer pool 에 데이터 적재


-  SELECT 일 경우  
Inno_db_buffer_pool 캐싱된 데이터 사용자에게 전달  
이 때 실행되는 쿼리의 시점과 동일한 시점의 데이터를 Read 해야 하는데 이를 위해 Undo의 기록을 활용한다.  

- DML 일 경우  
데이터 page 변경전 과거 데이터를 Undo log에 기록  
Inno_db_buffer_pool 의 데이터 page 변경 (Dirty buffer)
변경 내용은 Redo log buffer 에 기록 (Redo는 DB 서버의 모든 변경 사항이 기록됨)
        
        InnoDB의 경우 Undo의 무한 누적을 허용한다. (8KB Page 기준 32TB 이 한계값이나 사실상 무한에 가까움)
        이는 중요한데 타 DB인 Oracle의 경우 Undo file 의 무한 누적을 허용하지 않는다.  
        (Undo 데이터 파일 한계 사이즈가 존재. 8KB Page 기준 32GB)  
        
        이 때 Undo는 과거의 여러 시점의 데이터를 보존해야 하는데 너무나도 많은 트랜잭션이 몰아치거나, 실행하는 쿼리가 너무나도 오래 실행된다면 해당 시작 시점의 데이터는 이미 다른 트랜잭션에 의해 덮어 씌여져 더 이상 존재하지 않고 시점 일치가 불가하여 쿼리가 실패하게 된다.
        (덮어 씌어지는 것은 commit된 데이터)

        결론적으로 Undo의 무한 증가는 결국 Undo의 파일 크기 제한이 없고 무한 누적을 InnoDB가 허용하기 때문이다.
        

### 3. 결과 전달
도출된 결과는 사용자에게 전달된다.  
다만 전달과는 별개로 메모리 상의 Dirty page 와 같은 변경 사항 등이 Disk에 반영된 것은 아니므로 이후에 쓰기 작업이 진행된다.

# 참고
- ### Redo log

    대부분의 DBMS는 WAL(Write ahead log) 방식을 채택중이며 이는 데이터 파일에 쓰기 전 변경 사항을 필수적으로 먼저 기록한다는 것을 의미한다.
    (실제로 PostgreSQL은 WAL Log 라고 명칭)  

    Buffer pool의 데이터 변경 시 이는 Redo log buffer에 반영되고
    Commit 발생 혹은, Redo Log buffer가 특정 크기 이상 찼을 때, Log switch 등 발생 시 Redo log file에 기록한다.

    추가로 책에서는 Redo log buffer 의 크기를 16MB 크기로 권장한다.
    여기서 Redo log file의 크기도 동일하지는 않다. buffer의 크기를 작게 설정하는 것은 결국 memory의 내용을 Disk에 쓰는 시간을 줄이기 위함인데, 크기가 커지게 되면 이 시간이 길어져 성능 저하의 원인이 된다.

    내 경험상 보통 Redo log file의 경우 최초 크기를 300MB, DB 구모가 커진다면 GB 단위로 설정하였다. (buffer의 경우 변화 X)


- ### Buffering
    <Disk I/O의 최소화>  
    : DB의 입장에서 성능 저하의 최대 원인은 결국 Disk I/O 이다.  
    -> 가능하다면 모든 요청에 의한 데이터를 메모리에서 읽으면 좋지만 크기의 한계가 존재하기 때문에 모든 데이터를 적재할 수는 없다.

    그렇다면 Disk I/O 발생을 최소화하면 어떨까?  
    -> Dirty page, redo log 등 휘발성이 존재하는 메모리에 영구 보존할 수는 없기에 언젠가는 Disk에 쓰기 작업이 진행되어야 한다. 
    
    그렇다면 데이터 유실을 막기 위해 모든 발생하는 변경사항에 대해서 즉각적으로 쓰기 작업을 진행하자!   
    -> DB 성능은 말도 안되게 낮아질 것이다.

    위 사항의 절충안으로 책에서도 설명하는 Buffer의 방식으로 일정량이 모였을 때, 특정 기준에 따라 Disk 쓰기 작업을 진행하는 것이다. (Redo log buffer, innodb buffer pool 모두 buffer라는 키워드가 포함되어 있다.)

- ### Buffer pool의 크기 설정

    모든 DB에서 가장 중요하게 설정해야 하는 메모리 구조이다. 결국 실질적으로 데이터가 메모리에 올라가는 영역이기 때문에 해당 영역의 크기가 성능을 좌우한다. (Memory Hit ratio 때문)

    보통 1 Server 1 DB Instance 기준으로 물리 메모리의 약 50% 정도를 할당하곤 한다. (DB의 타 메모리 영역이 사용하는 공간도 있기 때문)

    이는 DB 서버의 리소스를 DB에게 대부분 할당하기 위해서 큰 공간을 부여한다. 다만 OS 기본 프로세스가 사용하게 되는 영역도 고려하여야 하는데 보통 10% 정도로 생각한다. 절대값으론 1G ~ 2G 정도로 알고 있으나 환경마다 다를 순 있다.

    하지만 DB 규모가 커짐에 따라 메모리 규모 또한 100G 이상으로 리소스 증설이 진행되는 경우가 많은데 이 때는 50% ~ 60% 의 비율을 유지하기엔 실제 OS 가 사용하는 절대값 1G ~ 2G 에 비해 낭비되는 리소스가 커지는 것이다. 

    예를 들어 100G 기준으로 DB가 50G를 쓰고 OS가 2G를 사용중이라면 약 40G에 가까운 메모리 영역이 놀게 되는 것이다. -> 곧 비용 낭비로 이어진다. 

    이 때문에 물리 메모리의 크기가 커질수록 할당하는 메모리 영역의 크기를 점진적으로 늘리게 된다.

    (물론 해당 DB 서버가 멀티 인스턴스 구조로 DB가 존재하거나 타 솔루션이 실행되어 메모리 영역을 보다 적게 사용해야 한다면 이 또한 고려하여 메모리 크기 할당이 필요하다.)

