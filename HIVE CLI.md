# HIVE CLI

# HIVE CLI란?

- 하이브 CLI(Command Line Interface)는 하이브 쿼리를 실행하는 가장 기본적인 도구

# CLI 옵션

- -f <filename> : 쿼리가 작성된 파일을 실행하기
    - e.g. hive -f test.sql
- - e <quoted-query-string> : 커맨드 라인으로 실행할 쿼리
    - e.g. hive -e 'show tables'
- -S 옵션 : 쿼리가 수행된 결과면 확인하기
    - e.g. hive -S -e 'show tables'
- —hiveconf <property=value> : 하이브 설정값 입력
    - e.g. —hiveconf tez.queue.name=aa_dev
- —hivevar <key=value> : 쿼리에서 사용할 변수 입력
    - e.g. —hivevar targetDate=2021-11-01

# 쿼리 실행

- 하이브 쿼리 실행은 인터렉티브 쉘에서 입력하는 방법, 커멘드 라인 입력, 파일 입력 방법 세가지가 존재

## 1. 인터렉티브 쉘 입력

- 하이브 CLI를 실행하고, 인터렉티브 쉘을 이용하여 입력하는 방법

```sql
-- 쉘에서 hive 실행
hive
-- 쿼리 입력
select 0;
```

### 변수 설정과 출력

```sql
-- 쉘에서 하이브 실행과 동시에 변수를 설정할 때
hive --define date_id=2021-11-01
-- 설정된 변수 확인
set date_id;
-- 인터렉티브 쉘에서 변수를 설정
set hivevar:date_id=2021-11-01
-- 다른 변수의 값 참조
set hive
```

### 변수 사용

```sql
select * 
from user_tory.tb_channel_enter_rsh_cnt 
where date_id='${hivevar:start}' limit 10;
```

## 2. 커맨드 라인 입력

- 커맨드 라인 입력은 -e 옵션을 이용
- 하이브 변수를 커맨드 라인에서 입력할 때, 하이브 변수가 쉘변수로 변환되어서 쿼리가 전달되지 않기 위해, 변수가 문자열로 취급되어야 함.
    - 그러기 위해선 쿼리가 ""이 아닌 ''으로 감싸져야 함

```bash
hive -e "select * from table"
hive -e 'select * from user_tory.tb_channel_enter_rsh_cnt where date_id="${hivevar:start}" limit 10' \
--hivevar start=2021-11-01 \
--hiveconf hive.execution.engine=tez
```

```bash
# hivevar을 이용한 쿼리
#!/bin/bash

start=2021-11-01
end=2021-11-05

while [ "${start}" != ${end} ]
do  
    echo "====================${start}=================="
    hive -e 'SET hive.cli.print.header = TRUE; SET SERDEPROPERTIES ("serialization.encoding"="utf-8");
    select 
        * 
    from user_tory.tb_channel_enter_rsh_cnt 
    where date_id="${hivevar:date}"
    limit 10;
    ' --hivevar date=${start} | sed 's/[\t]/,/g' > ../data/test_${start}.csv

    start=`date -I -d "${start} + 1days"`
done
```

```bash
# 쉘변수를 이용한 쿼리
#!/bin/bash

start=2021-11-01
end=2021-11-05

while [ "${start}" != ${end} ]
do  
    echo "====================${start}=================="
    hive -e "SET hive.cli.print.header = TRUE; SET SERDEPROPERTIES ('serialization.encoding'='utf-8');
    select * 
    from user_tory.tb_channel_enter_rsh_cnt 
    where date_id='${start}'
    limit 10;
    " | sed 's/[\t]/,/g' > ../data/test_${start}.csv

    start=`date -I -d "${start} + 1days"`
done
```

## 3. 파일 입력

- 파일 입력은 쿼리를 파일로 저장해놓고 해당 파일을 지정하는 방식
- -f 옵션을 이용

```bash
# test.sh
#!/bin/bash

start=2021-11-01
end=2021-11-05

while [ "${start}" != ${end} ]
do  
    echo "====================${start}=================="
    hive -f ../hive_query/test.hql --hivevar date=${start} | sed 's/[\t]/,/g' > ../data/test_${start}.csv

    start=`date -I -d "${start} + 1days"`
done
```

```sql
-- test.hql
SET hive.cli.print.header = TRUE; SET SERDEPROPERTIES ('serialization.encoding'='utf-8');
select * 
from user_tory.tb_channel_enter_rsh_cnt 
where date_id='${hivevar:date}'
limit 10;
```

### 로깅 방법

- 하이브 CLI는 log4j를 이용하여 로깅함. 로깅 방법을 변경하고자 할 때는 log4j 설정 파일을 변경하거나 —hiveconf 옵션을 이용

```bash
# 파일 로깅 
-- 로깅 레벨, 파일 위치 변경 
hive --hiveconf hive.log.file=hive_debug.log \
  --hiveconf hive.log.dir=./ \
  --hiveconf hive.root.logger=DEBUG,DRFA

# 콘솔 출력 
hive --hiveconf hive.root.logger=DEBUG,console
```

# **CLI 내부 명령어**

- jar 파일 추가는 사용자가 구현한 UDF, SerDe 등 자바 클래스 사용할 때 jar 파일의 위치를 지정합니다. 파일 추가는 스크립트나 UDF에서 내부적으로 사용하는 메타 파일 등을 추가할 때 사용합니다.

| 커맨드 | 설명 |
| --- | --- |
| exit | 종료 |
| reset | 설정값 초기화 |
| set <key>=<value> | 설정값 입력 |
| set | 하이브의 설정값 출력 |
| set -v | 하둡, 하이브의 설정값 출력 |
| add file <> | 파일 추가 |
| add files <> | 여러개의 파일 추가, 공백으로 경로 구분 |
| add jar <> | jar 파일 추가 |
| add jars <> | 여러개의 jar 파일 추가, 공백으로 경로 구분 |
| !<command> | 쉘 커맨드 실행 |
| dfs <dfs command> | 하둡 dfs 커맨드 실행 |

```sql
-- 옵션 설정
set mapred.reduce.tasks=32;
-- 모든 옵션 값 확인
set;
-- CLI 상에서 설정한 옵션 초기화
reset;
-- 쉘커맨드 실행
!ls -alh;
-- dfs 커맨드 실행
dfs -ls /user/;
-- 파일 추가
add file hdfs:///user/sample.txt;
-- 여러개의 파일 추가. 공백으로 구분
add files hdfs:///user/sample1.txt hdfs:///user/sample2.txt;
-- jar 파일 추가
add jar hdfs:///user/sample.jar;
```

# 더 자세한 설명

- 하이브 CLI 매뉴얼 [바로가기](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Cli) [↩](https://wikidocs.net/24771#fnref:1)
- 하이브 CLI 커맨드 [바로가기](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Cli#LanguageManualCli-HiveInteractiveShellCommands) [↩](https://wikidocs.net/24771#fnref:2)

# 참고

- [https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=kang721219&logNo=220703151674](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=kang721219&logNo=220703151674)
- [https://wikidocs.net/24771#fn:2](https://wikidocs.net/24771#fn:2)