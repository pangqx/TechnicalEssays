# Managing Tables 表的管理
## Online Table Redefinition 表的在线重定义
### 目的
调整表的逻辑或物理结构：包括分区结构、列的数据类型、增加或删除列、存储参数、所属表空间等。

### 特性
- 对表的可用性影响极小：表的在线重定义大部分步骤不会中断表的查询和DML，在一个很短的时间窗口会锁表，具体时间取决于表的大小和表DML的频繁程度。
- 支持在某个中间步骤出现错误而失败，修复该错误后从上次失败的步骤继续执行。
- 支持回滚到表的原始状态，并且保留表的在线重定义期间对表的DML。
- 可以监控表的在线重定义进度、状态和错误信息。

### 限制
- 至少有跟原表大小相当的空闲表空间。
- 包含LONG列的表可以被在线重定义，但是这些列必须转换为CLOB类型。
- SYS和SYSTEM用户的表不能被在线重定义。

### 步骤
用 DBMS_REDEFINITION 包中的多个存储过程实现表的在线重定义。
1. 选择重定义方式：KEY或ROWID。
- KEY - 主键或唯一且非空的列，首选和默认的方式。
- ROWID - 当KEY方式不可用时选择该方式。
2. 调用存储过程 CAN_REDEF_TABLE 验证表是否支持在线重定义。
3. 创建影子表（interim table）。影子表的索引、约束、权限、触发器不需要创建，第7步会自动从原表复制。
4. （可选）如果用ROWID方式对一个分区表在线重定义，影子表需要enable row movement: `ALTER TABLE ... ENABLE ROW MOVEMENT;`
5. （可选）提升对一个大表在线重定义时的性能：
```
    ALTER SESSION FORCE PARALLEL DML PARALLEL N;
    ALTER SESSION FORCE PARALLEL QUERY PARALLEL N;
```
6. 调用存储过程 START_REDEF_TABLE 开始在线重定义。
7. 调用存储过程 COPY_TABLE_DEPENDENTS 自动复制依赖原表的对象（索引、约束、权限、触发器、统计信息）到影子表。
8. 调用存储过程 SYNC_INTERIM_TABLE 同步增量数据到影子表，可以多次执行。
9. 调用存储过程 FINISH_REDEF_TABLE 完成在线重定义。
10. 删除影子表。


### 示例
- 原表 hr.emp 定义:
```
  empno     NUMBER(4,0)  PRIMARY KEY
  ename     VARCHAR2(10) NOT NULL
  job       VARCHAR2(9)
  hiredate  DATE         NOT NULL
  sal       NUMBER(7,2)
```
- 验证表 hr.emp 是否支持在线重定义
```
BEGIN
  DBMS_REDEFINITION.CAN_REDEF_TABLE(
    uname        => 'hr',
    tname        => 'emp',
    options_flag => DBMS_REDEFINITION.CONS_USE_PK);
--  options_flag => DBMS_REDEFINITION.CONS_USE_ROWID);
END;
```
- 创建影子表 hr.emp_int，按字段empno分区
```
CREATE TABLE hr.emp_int
( empno     NUMBER(4,0),
  ename     VARCHAR2(10),
  job       VARCHAR2(9),
  hiredate  DATE,
  sal       NUMBER(7,2),
)
PARTITION BY RANGE(empno)
(
    PARTITION emp1000 VALUES LESS THAN (1000),
    PARTITION emp2000 VALUES LESS THAN (2000)
);
```
- 开始在线重定义
```
BEGIN
  DBMS_REDEFINITION.START_REDEF_TABLE(
    uname        => 'hr', 
    orig_table   => 'emp',
    int_table    => 'emp_int',
    col_mapping  => NULL,
--  col_mapping  => 'empno empno, ename ename, job job, hiredate hiredate, sal*1.2 sal',
    options_flag => DBMS_REDEFINITION.CONS_USE_PK,
--  options_flag => DBMS_REDEFINITION.CONS_USE_ROWID,
    enable_rollback => FALSE);
END;
```
- 将依赖表的对象从原表拷贝到影子表
```
DECLARE
num_errors PLS_INTEGER;
BEGIN
  DBMS_REDEFINITION.COPY_TABLE_DEPENDENTS(
    uname            => 'hr', 
    orig_table       => 'emp',
    int_table        => 'emp_int',
    copy_indexes     => DBMS_REDEFINITION.CONS_ORIG_PARAMS, 
    copy_triggers    => TRUE, 
    copy_constraints => TRUE, 
    copy_privileges  => TRUE, 
    copy_statistics  => TRUE,
    ignore_errors    => FALSE, 
    num_errors       => num_errors);
END;
```
- 同步原表的增量数据到影子表
```
BEGIN 
  DBMS_REDEFINITION.SYNC_INTERIM_TABLE(
    uname      => 'hr', 
    orig_table => 'emp', 
    int_table  => 'emp_int');
END;
```
- 完成在线重定义，原表会短暂锁表
```
BEGIN
  DBMS_REDEFINITION.FINISH_REDEF_TABLE(
    uname      => 'hr', 
    orig_table => 'emp', 
    int_table  => 'emp_int',
    dml_lock_timeout => NULL);
END;
```
- 删除影子表
```
drop table hr.emp_int;
```
- （如有必要）中断在线重定义，在START_REDEF_TABLE之后及FINISH_REDEF_TABLE之前执行
```
BEGIN 
  DBMS_REDEFINITION.ABORT_REDEF_TABLE(
    uname      => 'hr', 
    orig_table => 'emp', 
    int_table  => 'emp_int');
END;
```

### 监控在线重定义过程
- V\$ONLINE_REDEF
```
SQL> SELECT OPERATION, SUBOPERATION, PROGRESS FROM V$ONLINE_REDEF;
OPERATION                SUBOPERATION         PROGRESS
----------------------   ------------------   ----------------
COPY_TABLE_DEPENDENTS    copy the indexes     step 3 out of 7
```
- DBA_REDEFINITION_STATUS
```
SQL> SELECT BASE_TABLE_NAME,  INT_TABLE_NAME, OPERATION, STATUS, RESTARTABLE, ACTION FROM DBA_REDEFINITION_STATUS;

BASE_TABLE_NAME INT_OBJ_NAME OPERATION          STATUS  RESTARTABLE ACTION
--------------- ------------ ------------------ ------- ----------- ---------
EMP             EMP_INT      SYNC_INTERIM_TABLE FAILED  Y           Fix error
```



## Table Creation
- 每个表都应有非业务主键，作为该表每一行记录的唯一标识，同时应有创建时间、更新时间字段，便于数据转移、抽取、同步等操作。
- 出于对表性能影响对考虑，联机交易表不创建外键约束，在应用层面实现数据一致性的控制。
- 不会出现空值的列添加NOT NULL约束，这将直接影响优化器对SQL执行计划的选择，比如索引全扫描的前提条件是目标索引的键值列至少有一个是NOT NULL。


## Altering Tables


## Researching and Reversing Erroneous Table Changes


## Managing Partitioned Tables

## Dropping Tables


## Tables Data Dictionary Views







# Managing Indexes 索引的管理
## Guidelines for Managing Indexes
- 联机交易表原则上不要超过6个索引，索引有且仅有一个好处就是加快查询，过多的索引会降低DML语句执行效率。
- 分区表的索引尽可能使用分区索引，某些场景下可以提高查询效率且便于对索引的维护。分区表上的全局索引在表的分区被drop后会产生碎片，定时对分区表上的全局索引进行rebuild释放表空间。
- OLTP系统创建的索引类型是普通B-Tree索引，不应创建位图索引。
- SQL语句中出现在where条件中的列才需要考虑添加索引。
- 某一列或几列是否适合添加索引取决于where条件的筛选率，当where条件过滤后的结果集行数不超过该表总行数的30%时走索引的效率高于全表扫描。
- 创建多列复合索引时将可选择率低的列作为该索引的前导列，可选择率=该列distinct值数量/表总行数。
- 不会出现重复值的列在创建索引时应创建唯一索引，这将直接影响优化器对SQL执行计划的选择，唯一索引往往比非唯一索引效率更高。

## Creating Indexes


## Altering Indexes

## Index Visibility

## Dropping Indexes

## Indexes Data Dictionary Views







