# sql tunning study  

## I 튜닝 원리 이해
tkprof
후보실행계획 찾기 -> 오브젝트, 시스템 통계 기반 예상 경로 -> 실행계획 비교 및 최저비용 선택
규칙기반(미리정해놓은 규칙) -> 비용기반(통계정보 바탕으로 최소비용 선택)

explain plan for
select /*+ rule ^/ * from kosta.emp order by empno;


select * from table(dbms_xplan.display);

CREATE TABLE T_EMP AS
SELECT B.NO, A.*
FROM (SELECT * FROM KOSTA.EMP WHERE ROWNUM <=10) A
, (SELECT ROWNUM NO FROM DUAL CONNECT BY LEVEL <= 100) B;

exec dbms_stats.gather_table_stats(user, 'KOSTA.T_EMP', method_opt => 'for all columns size 5');

[실행계획등록]
explain plan for
select * from kosta.emp where empno > 7400;

explain plan for
select * from kosta.emp where empno <> 7400;

[실행계획 조회]
select * from table(dbms_xplan.display);


user_table_columns <- num_distinct


[sqlplus 명령어 화면설정법]
1. SQL>set linesize 300
2. SQL>set pagesize 100

[설정값 확인 방법]
SQL> show pagesi
SQL> show linesize

[튜닝]
캐시구조 & 옵티마이저
정규화 & 반정규화
파티셔닝


[라이브러리 캐시 최적화]
/* shared_pool 캐싱 영역을 비우기 */
alter  system flush shared_pool;
select /*AAA*/* from kosta.emp where ename = 'SMITH';

select sql_id, parse_calls, loads, executions, invalidations,
decode(sign(invalidations), 1, (loads-invalidations), 0) reloads
from v$sql
where sql_text like '%/*AAA*/%'
and sql_text not like '%v$sql%';

[라이브러리 최적화]
통계정보 명령어 (*가급적이면 gather_dbms_stats 사용해라)
analyze 와  				gather_dbms_stats 차이
수집의 순차,						병렬
파티션 테이블에 대한 수집잘 못함, 		잘함
통계정보에 대한 export, import 안됨, 	됨

create table t as select * from all_objects;
update t set object_id = rownum;
create unique index t_idx on t(object_id);
analyze table t compute statistics;
alter system flush shared_pool;

## II 실행 계획을 통한 SQL 분석 및 튜닝
[실행계획을 통한 SQL 분석]
통계정보산출
EXPLAIN_PLAN -> DBMS_XPLAN
AUTOTRACE
DBMS_XPLAN
10046 TRACE => tkprof

[튜닝준비]
v$sql : SGA에 실행되는 쿼리들
데이터모델, DB 설계 > 어플리케이션 튜닝 > 데이터베이스 튜닝 > 시스템 튜닝

어플리케이션 튜닝 : one-sql, Array-Processing, fetch size
데이터베이스 튜닝 : parameter 설정

[실행계획]
SQL + Optimizer => 실행계획 (EXPLAIN_PLAN, DBMS_XPLAN, AUTOTRACE, 10046 EVENT)

조인순서, 조인기법, 액세스 기법, 최적화정보, 연산 등

[통계정보는 미리 생성해 놓아야 함-시간이 걸림]
GATHER_INDEX_STATS : 인덱스 통계정보 생성
GATHER_TABLE_STATS : 테이블 통계정보 생성
GATHER_SCHEMA_STATS : 스키마 통계정보 생성
GATHER_DATABASE_STATS : 데이터베이스 모든 통계정보 생성
GATHER_SYSTEM_STATS : 시스템(CPU,IO) 통계정보 생성


[통계정보 분석 비교]
아래 이유로 analyze 보다 dbms_stats 사용
ANALYZE : DBMS_STATS
--------------------
serial : parallel
partition  떨어짐 : partition 잘됨
empty_block : X 

[explain plan]
1. 
create table kosta.tb_stat
(stat_id number(10), stat_num number(10), stat_name varchar2(20));

begin
for i in 1001 .. 5000 loop
insert into kosta.tb_stat values( i, i , 'statname');
end loop;
end;
/

select * from kosta.tb_stat;

select num_rows, blocks, empty_blocks , avg_space, avg_row_len, last_analyzed
from dba_tables
where owner = 'KOSTA'
and table_name = 'TB_STAT';

exec dbms_stats.gather_table_stats('KOSTA', 'TB_STAT');


2.
explain plan set statement_id = 'kosta_query01' for
select * from kosta.emp where empno = 7900;

@?/rdbms/admin/utlxpls

PLAN_TABLE_OUTPUT

3.
explain plan set statement_id = 'kosta_query02' for
select * from kosta.emp where empno > 7400 and ename = 'JAMES';
@?/rdbms/admin/utlxpls

[autotrace]
set autotrace on;


[dispaly cursor 설정 전에 레벨 조정]  
alter session set statistics_level = ALL;  


select /*+gather_plan_statistics*/ * from kosta.emp inner join kosta.dept on emp.deptno = dept.deptno where empno = 7900;
select * from table(DBMS_XPLAN.DISPLAY_CURSOR(null, null, 'ALLSTATS LAST'));


[10046 trace]
실측정보
SPID 확인 -> Trace 활성화 -> Trace 비활성화 -> Session 종료

1. SPID 확인
select p.spid from v$process p, v$session s
where p.addr = s.paddr
and s.audsid = USERENV('SESSIONID');

34712
2. Trace 활성화
alter session set sql_trace = true;
alter session set timed_statistics = true;
alter session set events '10046 trace name context  forever, level 12';

2-1. trc 파일 생성 위치 확인
show parameter user_dump_dest;
NAME                                 TYPE                   VALUE
------------------------------------ ---------------------- ------------------------------
user_dump_dest                       string                 C:\oraclexe\app\oracle\diag\rdbms\xe\xe\trace

3. 여러가지 sql 문장 실행


4. 종료
alter session set sql_trace = false;

5. 파일을 열어 분석하거나 tkprof 이용
5.1 파일
C:\oraclexe\app\oracle\diag\rdbms\xe\xe\trace\xe_ora_34712.trc

5.2 tkprof
c:\study\sql>tkprof C:\oraclexe\app\oracle\diag\rdbms\xe\xe\trace\xe_ora_34712.trc
output = c:\study\sql\result.txt

6. 다른 세션 모니터링
Exec dbms_monitor.session_trace_enable(session_id => 1, serial_num => 1, waits => true, binds => true);
Exec dbms_monitor.session_trace_disable(session_id => 1, serial_num => 1);

7. 단위 모니터링 (SID 단위 등)
Exec dbms_monitor.serv_mod_act_trace_enable(
    service_name => 'ORCL', module_name => dbms_monitor.all_module, action_name -> dbms_monitor.all_actions, waits => true, binds => true
);

[튜닝을 위한 데이터 베이스 설정]

set linesize 200
set pagesize 100

# 아래는 운영 서버에서 하지 말길
Alter system flush shared_pool;
Alter system flush buffer_cache;


[힌트사용]
1. 힌트 구문
SQL 문을 바꾸지 않고 옵티마이저에게 지시하는 구문
힌트사용 : SELECT /*+ 힌트문 */ COLUMN_1, COLUMN_2 FROM TABLE

2. 힌트 사용 용도
액세스 경로, 조인 순서, 처리 방법, 데이터값 정렬, 드라이빙 테이블 선정 등
