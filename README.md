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
