## 오라클 인메모리 컬럼스토어 사용 방법

오라클 인메모리 옵션은 모든 데이터를 기반으로 전체 데이터 분석 업무를 빠르게 처리해야 하는 경우, 별도의 DW 서버 또는 데이터 분석 전용서버를 운영하기 어려운 환경, 운영 DB에서 실시간 데이터 분석작업이 필요한 업무 등 운영 DB에서 빠른 시간안에 분석 보고서 작성이 필요한 곳에서 빠른 결과를 얻는데 도움을 줌. 특히 다음과 같은 조건에서 효과를 발휘함.

1. Full Scan을 사용해야 하는 경우 Disk I/O가 Memory I/O로 대체됨으로 성능이 향상됨.
2. 대용량 분석 및 집계 업무에 최적화됨. 메모리 배열을 사용하여 Fact 테이블 스캔(Full scan)과 집계값 누적을 통시에 처리하는 vector group by 기술 사용
3. Where절에 Filter 조건으로 걸러지는 경우가 많을 때 속도 최적화됨(Storage Index Pruning, Dictionary Pruning 기술 사용).
4. SQL에서 사용하는 컬럼 개수가 작을 때 보다 더 최적화됨.
5. SELECT 결과 건수가 많으면 fetch 시간이 많이 걸림으로 In-Memory Option의 장점이 반감됨.

### 인모메모리 적격성 테스트(In-Memory Eligibility Test) 유틸리티

인모메모리 적격성 테스트는 AWR과 ASH를 사용하여 과거 데이터베이스 워크로드를 평가하고 해당 데이터베이스가 인메모리 기능을 사용하기에 적합한지 여부를 평가하는 유틸리티임. 인모메리 Advisor 자체를 실행하기 전에 인메모리 적격성 테스트를 실행하는 것이 좋음. 적격성 테스트 결과가 TURE 결과 경우에 인모메리 기능 사용하기에 적합한 후보임을 나타냄. 이 테스트는 인메모리 옵션을 활성화 하기 전에 실행해야함.
 

아래 예제는 최근 4시간의 AWR, ASH 기록으로 테스트를 실행하는 예제임. 

```sql

sqlplus / as sysdba

set serveroutout on

declare
 inmem_eligible BOOLEAN;
 analysis_summary VARCHAR2(4000);
begin
 dbms_inmemory_advise.is_inmemory_eligible(sysdate-4/24, sysdate, inmem_eligible, analysis_summary);
 DBMS_OUTPUT.PUT_LINE('====================================================================');
 dbms_output.put_line('Is In-Memory Elegible..: ' || to_char(inmem_eligible));
 dbms_output.put_line('Analysis Summary.......: ' || analysis_summary);
 DBMS_OUTPUT.PUT_LINE('====================================================================');
end;
/

====================================================================
Is In-Memory Elegible..: TRUE
Analysis Summary.......: Observed Analytic Workload Percentage is 97.56% is greater than target Analytic Workload Percentage 20%
====================================================================

PL/SQL procedure successfully completed.
```

### 인메모리 어드바이저(IM Advisor)

In-Memory Advisor는 워크로드에 대한 자세한 분석을 실행하여 인메모리 사이즈, 권고 테이블, 권고 파티션 목록 등을 출력해줌. 인메모리 어드바이저 실행은 인메모리 옵션 활성화 전에 실행해야하며 실행 절차는 1.heap_map 활성화-2.어드바이저 타스크 생성-3.타스크 중단-4.분석결과 생성-5.결과 조회 단계로 진행하게됨. 23ai 버전에서는 DBMS_INMEMORY_ADVISE 패키지를 이용하여 분석 정보를 얻을 수 있음.

다음은 인메모리 어드바이저 실행 예제임.

```sql
1.heap_map 활성화

alter system set heat_map=on;

2.어드바이저 타스크 생성

set serveroutput on;

variable task_id NUMBER;

exec dbms_inmemory_advise.start_tracking(:task_id);

print task_id

    Task
      ID  
----------
         5

3.타스크 중단

# 애플리케이션이 실행되는 환경에서 SAG에 있는 쿼리를 분석할 충분한 snapshot 시간 동안 대기 필요

EXEC DBMS_INMEMORY_ADVISE.STOP_TRACKING();

4. 분석결과 생성

EXEC DBMS_INMEMORY_ADVISE.GENERATE_ADVISE();

5. 결과 조회

SELECT  
  task_id,  
  INMEMORY_SIZE,  
  ESTIMATED_DB_TIME_LOW,  
  ESTIMATED_DB_TIME_HIGH,  
  ESTIMATED_DB_TIME_ANALYTICS_LOW,  
  ESTIMATED_DB_TIME_ANALYTICS_HIGH,  
  TO_CHAR(RECOMMENDED_OBJ_LIST) RECOMMENDED_OBJ_LIST  
FROM  
  DBA_INMEMORY_ADVISOR_RECOMMENDATION;  

```

### 인메모리 컬럼 스토어 활성 및 메모리 사이즈 지정

인메모리 컬럼 스토어 기능 활성화는 인메모리 사이즈를 설정해줌으로써 활성화됨. 인메모리 사이즈 지정은 inmemory_size 파라미터를 사용하며 최소 사이즈는 100MB 이상을 설정해주면 됨. inmemory advisor의 권고값을 참고해서 사용할 수 있음. inmemory_size 파라미터를 설정한 경우에는 DB를 재기동해야 적용됨. "In-Memory Area" 값으로 마운트된 인메모리 사이즈를 확인 할 수 있음. 


SGA_TARGET을 지정해서 사용하는 경우에는 인메모리 컬럼 스토어용 메모리 사이즈를 감안해서 일반 SGA 사이즈 + 인메모리 옵션 사이즈의 합 이상이어야 함.
 
```sql
alter system set inmemory_size=1500M scope = spfile;

shutdown immediate;

startup;
Total System Global Area 4731172136 bytes
Fixed Size                  8906024 bytes
Variable Size            1476395008 bytes
Database Buffers         1610612736 bytes
Redo Buffers               58200064 bytes
In-Memory Area           1577058304 bytes
Database mounted.
Database opened.

```

권고사항 : 인메모리 컬럼스토어에 올라가는 테이블의 인덱스는 삭제 또는 invisible 처리할 것을 권고함. 


### 인메모리 컬럼스토어에 객체 올리기

인메모리 컬럼 스토어에 올릴 수 있는 객체 단위는 테이블스페이스, 테이블, 파티션, 컬럼 객체 단위임. 객체 생성 때 또는 alter 구문으로 지정하며, 컬럼인 경우는 제외 대상 컬럼을 지정함으로써 제외시킬 수 있음.

### 1. 수동 팝업(On-demand population)

#### 테이블스페이스 단위 팝업 및 조회

```sql
alter system set optimizer_dynamic_sampling=4;

alter tablespace TPCHTAB default inmemory;

select tablespace_name, def_inmemory from dba_tablespaces;
```

#### 테이블 단위 팝업 

```sql
# 테이블 레벨에서 IM 컬럼스토어 지정

ALTER TABLE sh.customers INMEMORY;

# IM 컬럼스토어 팝업은 객체를 스캔할 때 IM 컬럼스토어로 올라감

select count(*) from customers;

# 또는 프로시저를 이용하여 수동으로 팝업이 가능함. 

execute dbms_inmemory.populate('SH','CUSTOMERS');
# "SH"는 객체 소유자임

```
#### 테이블 팝업 조회

```sql
# IM 컬럼스토어 팝업된 객체 조회

SELECT OWNER, SEGMENT_NAME, BYTES/1024/1024 ORG_MB, INMEMORY_SIZE/1024/1024 IM_MB FROM V$IM_SEGMENTS;

# IM 사용 사이즈 조회

SELECT pool, alloc_bytes/1024/1024 alloc_MB, used_bytes/1024/1024 Used_MB, populate_status, con_id FROM V$INMEMORY_AREA;

```


#### 컬럼 단위 팝업 및 조회

```sql
1. IM에 올라간 테이블의 컬럼 리스트 조회
col owner format a30
col table_name format a30
col column_name format a30
SELECT OWNER, TABLE_NAME, COLUMN_NAME, INMEMORY_COMPRESSION FROM V$IM_COLUMN_LEVEL;

OWNER                          TABLE_NAME                     COLUMN_NAME                    INMEMORY_COMPRESSION
------------------------------ ------------------------------ ------------------------------ --------------------------
....

2. 제외 대상 컬럼을 제외해줌
ALTER TABLE SALES NO INMEMORY(PROD_ID, AMOUNT_SOLD, CHANNEL_ID);

3. IM 컬럼 스토어 확인
COL OWNER FORmat A10
COL SEGMENT_NAME FOR A20
COL TABLE_NAME FORMAT A30
COL COLUMN_NAME FORMAT A30
SELECT OWNER, TABLE_NAME, COLUMN_NAME, INMEMORY_COMPRESSION FROM V$IM_COLUMN_LEVEL;

```

### 2. 자동 IM 팝업 (Priority-based population)

데이터베이스가 재기동 될때 지정된 우선순위 기반으로 IM컬럼스토어에 자동으로 객체를 팝업 시킬 수 있음.

#### Priority-based population 위한 파라미터 지정

```sql
show paramater INMEMORY_AUTOMATIC_LEVEL 
inmemory_automatic_level             string      NONE

alter system set INMEMORY_AUTOMATIC_LEVEL = HIGH scope=spfile;

# INMEMORY_FORCE = BASE_LEVEL로 설정되어 있는 경우 자동 팝업이 안됨

# 파라미터 수정 후 DB 재기동 필요
```

#### Priority-based population 위한 객체 지정
```sql
alter table VECTOR.DOC_VECTOR INMEMORY PRIORITY CRITICAL;

# INMEMORY PRIORITY = CRITICAL | HIGH | MEDIUM | LOW | NONE
# 디폴트는 NONE임
```

### 3. 기타 사용 방법

#### IM 스토어 객체 취소 방법
```sql
alter table sh.customers no inmemory;
```

#### IM 컬럼 스토어 관련 view

```sql
# In-Memory Area에 Load 된 Object 리스트 확인

COL OWNER FOR A20
COL SEGMENT_NAME FOR A20
col INMEMORY_SIZE for 999,999,999
col IM_MB for 999,999,999
col ORG_MB for 999,999,999

SELECT OWNER, SEGMENT_NAME, BYTES/1024/1024 ORG_MB, INMEMORY_SIZE/1024/1024 IM_MB FROM V$IM_SEGMENTS;

# In-Memory area 조회
col alloc_mb format 999,999
col Used_MB format 999,999
SELECT pool, alloc_bytes/1024/1024 alloc_MB, used_bytes/1024/1024 Used_MB, populate_status, con_id FROM V$INMEMORY_AREA;

# 세그먼트별 인메모리 팝업 상태 조회
col segment_name format a50
COMPUTE SUM LABEL 'TOTAL' OF ORG_MB  ON REPORT
COMPUTE SUM OF IM_MB ON REPORT
BREAK ON REPORT  
SELECT segment_name, bytes/1024/1024 ORG_MB, inmemory_size/1024/1024 IM_MB, populate_status, bytes_not_populated 
from v$im_segments where owner='&owner';

```

#### 팝업 소요 시간 계산

```sql
col object_name for a30 
col diff for a30 
select a.object_name, b.inmemory_priority, b.populate_status, to_char(c.createtime,'hh24:mi:ss.ff2') start_pop, 
to_char(max(d.timestamp), 'hh24:mi:ss.ff2') finish_pop, max(d.timestamp) - c.createtime diff 
from dba_objects a, v$im_segments b, v$im_segments_detail c, v$im_header d 
where a.owner = '&owner' and object_type = 'TABLE' 
and a.object_name = b.segment_name and a.object_type = 'TABLE' and a.object_id = c.baseobj and c.dataobj = d.objd 
group by a.object_name, b.inmemory_priority, b.populate_status,c.createtime;

```

#### 객체 IM스토어 압축 방법

```sql

# 테이블 인메모리 압축(low, high)

alter table sh.sales inmemory memcompress for query low;

# No compressed table on in-memory

alter table sh.sales inmemory no memcompress;

```

#### 압축 비율과 사이즈 확인

```sql
col owner format a30
col segment_name format a30
select owner, segment_name, bytes, (bytes/inmemory_size) com_ratio from v$im_segments order by segment_name;
 
### IM 객체 압축 레벨 확인
col table_name format a100
select table_name, inmemory, inmemory_priority, inmemory_compression from dba_tables 
where owner = '&owner' and inmemory = 'ENABLED'
order by table_name;

```

#### 컬럼 단위 IM compress 정보 확인

```sql
col table_name for a20 
col column_name for a35 
col segment_name format a40

select table_name, column_name, inmemory_compression from v$im_column_level where table_name = '&table_name';

select segment_name, segment_type, inmemory, inmemory_priority, inmemory_compression from dba_segments 
where owner = '&owner' and inmemory = 'ENABLED'
```

-----------------------------------------------------------
end of document
