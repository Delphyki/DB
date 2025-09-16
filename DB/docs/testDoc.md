运行(服务客户端均用终端)

 $env:PYTHONPATH = "D:\\Training3\\IMoocDB-master\\imoocdb"（重启终端需要）

 python imoocdb/main.py



###### 一、基础 DDL 功能测试

-- 1. 创建表

CREATE TABLE test\_users (id int, name text, age int);

*验证 SQL 解析器对 CREATE TABLE 语句的处理（对应 test\_sql\_parser.py 中的 test\_parse\_ddl\_statement）。*

*检查 Catalog 模块是否正确记录表元数据（对应 test\_catalog.py 中的表元数据存储逻辑）。*



-- 2. 创建索引

CREATE INDEX idx\_users\_id ON test\_users (id);（单列）

CREATE INDEX idx\_users\_id\_name ON test\_users (id, name);（或复合）

*验证索引元数据是否被 Catalog 记录（对应 test\_catalog.py 中的 test\_catalog\_index）。*

*检查存储引擎是否正确生成 B + 树索引文件（对应 test\_bplustree.py 中的索引序列化逻辑）*



-- 3. 查看表结构

SELECT \* FROM test\_users;

SELECT \* FROM test\_users WHERE test\_users.id = 1;

*验证表结构定义是否正确生效，字段类型是否匹配（INT/TEXT 类型处理）。*



###### 二、DML 功能测试

-- 1. 插入数据

INSERT INTO test\_users VALUES (1, 'Alice', 25);

INSERT INTO test\_users VALUES (2, 'Bob', 30);

INSERT INTO test\_users VALUES (3, 'Charlie', 35);

*测试插入逻辑（对应 test\_executor.py 中的 PhysicalInsert 算子）。*

*验证事务日志（Redo/Undo Log）是否正确记录（对应 test\_redo\_undo\_log.py）。*



-- 2. 查询数据（全表，条件，聚合）

SELECT \* FROM test\_users;

SELECT \* FROM test\_users WHERE test\_users.id = 1;

SELECT \* FROM test\_users WHERE test\_users.age > 25;

*全表扫描验证 TableScan 算子（test\_executor.py 中的 test\_table\_scan）。*

*条件查询验证过滤逻辑（BinaryOperation 解析与执行）。*

*聚合查询验证 HashAgg 算子（test\_executor.py 中的 test\_hash\_agg）。*



-- 3. 更新数据

UPDATE test\_users SET test\_users.age = 26 WHERE test\_users.name = 'Alice';

*验证 PhysicalUpdate 算子（test\_executor.py 中的 test\_physical\_dml）。*

*检查事务 ACID 特性：更新操作是否被正确记录和提交。*



--4.删除数据

DELETE FROM test\_users WHERE test\_users.id = 2;

*验证 PhysicalDelete 算子（test\_executor.py 中的 test\_physical\_dml）。*

*检查存储引擎是否标记元组为 “死元组”（test\_storage.py 中的 table\_tuple\_is\_dead 逻辑）。*



###### 三、索引功能测试

--1.索引扫描查询

SELECT \* FROM test\_users WHERE test\_users.id = 2;

*验证索引扫描替代全表扫描（对应 test\_executor.py 中的 test\_index\_scan）。*

*检查 B + 树索引的查找效率（test\_bplustree.py 中的 find 方法）。*



--2.覆盖索引查询

SELECT test\_users.id, test\_users.name FROM test\_users WHERE test\_users.id = 1;

*验证覆盖索引逻辑（test\_storage.py 中的 covered\_index\_tuple\_get\_range），直接从索引获取数据而非表。*



###### 四、执行计划测试（EXPLAIN）

EXPLAIN SELECT \* FROM student WHERE id > 1;

Query

&nbsp; -> IndexScan (index: idx\_student\_id, condition: id > 1)

*验证执行计划生成逻辑（test\_planner.py 中的 query\_logical\_plan 和 query\_physical\_plan）。*

*检查是否正确选择索引扫描或全表扫描。*



###### 六、异常场景测试

--1.语法错误

-- 1.1 缺少分号

SELECT test\_users.id, test\_users.name FROM test\_users

-- 1.2 错误的关键字

SELEC test\_users.id FROM test\_users;

-- 1.3 缺少必需子句

SELECT FROM test\_users;

-- 1.4 错误的运算符

SELECT test\_users.id FROM test\_users WHERE test\_users.id => 1;

-- 1.5 括号不匹配

SELECT test\_users.id FROM test\_users WHERE (test\_users.age > 25;

*返回语法错误提示（对应 test\_sql\_parser.py 中的错误处理）。*



--2.数据类型错误

-- 2.1 字符串作为数字

INSERT INTO test\_users VALUES ('invalid\_id', 'Test\_User', 25);

-- 2.2 数字作为字符串

INSERT INTO test\_users VALUES (300, 12345, 25);

-- 2.3 超出范围的值

INSERT INTO test\_users VALUES (9999999999, 'Test\_User', 25)；



--3.表不存在

-- 3.1 查询不存在的表

SELECT \* FROM non\_existent\_table;

-- 3.2 向不存在的表插入数据

INSERT INTO non\_existent\_table VALUES (1, 'test', 25);

-- 3.3 更新不存在的表

UPDATE non\_existent\_table SET column = 'value' WHERE id = 1;

-- 3.4 删除不存在的表

DELETE FROM non\_existent\_table WHERE id = 1;

*对应 sql/utils.py 中的 table\_exists 和 column\_exists 检查*



--4.事务冲突

-- 4.1 长时间运行的事务

BEGIN;

INSERT INTO test\_users VALUES (301, 'Long\_Running\_User', 30);

-- 等待超时时间...

COMMIT;

-- 4.2 事务中的错误操作

BEGIN;

INSERT INTO test\_users VALUES (302, 'User\_With\_Error', 31);

-- 故意制造错误

SELECT test\_users.non\_existent\_column FROM test\_users;

COMMIT;

-- 4.3 嵌套事务

BEGIN;

BEGIN;

INSERT INTO test\_users VALUES (303, 'Nested\_User', 32);

COMMIT;

COMMIT;

--4.4并发事务

-- 会话1：

BEGIN;

INSERT INTO test\_users VALUES (304, 'Session1\_User', 33);

-- 会话2：

BEGIN;

-- 尝试插入相同ID（主键冲突）

INSERT INTO test\_users VALUES (304, 'Session2\_User', 34);

COMMIT;

-- 会话1：

COMMIT;

*验证事务冲突处理（如锁机制），避免脏写。*



--5.系统资源限制(大量数据插入,测试内存限制)

BEGIN;

INSERT INTO test\_users VALUES (307, 'User1', 39);

INSERT INTO test\_users VALUES (308, 'User2', 40);

-- ... 插入大量数据

COMMIT;

-- 复杂查询（测试CPU限制）

SELECT test\_users.id, test\_users.name, test\_users.age 

FROM test\_users 

WHERE test\_users.age > 20 

AND test\_users.name LIKE '%A%' 

AND test\_users.id IN (SELECT test\_users.id FROM test\_users WHERE test\_users.age < 30);



--6.边界值错误

-- 6.1 极大值测试

INSERT INTO test\_users VALUES (2147483647, 'Max\_Int\_User', 150);

-- 6.2 极小值测试  

INSERT INTO test\_users VALUES (-2147483648, 'Min\_Int\_User', 0);

-- 6.3 超长字符串

INSERT INTO test\_users VALUES (306, 'A' || REPEAT('B', 1000), 38);



--7.数据完整性错误

-- 7.1 空值违反约束（如有NOT NULL约束）

INSERT INTO test\_users VALUES (NULL, 'Null\_ID\_User', 35);

-- 7.2 重复

INSERT INTO test\_users VALUES (1, 'Duplicate\_ID\_User', 36);  -- 假设id=1已存在

-- 7.3 违反唯一约束

INSERT INTO test\_users VALUES (305, 'Alice', 37);  -- 假设name='Alice'已存在



###### 五、事务管理测试

--1.事务提交

BEGIN;

INSERT INTO student VALUES (3, 'Charlie', 23);

COMMIT;

SELECT \* FROM student WHERE id = 3; -- 预期返回数据

*验证事务提交后数据持久化（transaction\_mgr.commit\_transaction）。*

*检查 Redo Log 是否正确写入（test\_redo\_undo\_log.py 中的 test\_redo）。*



--2.事务回滚

BEGIN;

UPDATE student SET name = 'David' WHERE id = 2;

ROLLBACK;

SELECT name FROM student WHERE id = 2; -- 预期返回'Bob'（未修改）

*验证事务回滚后数据恢复（transaction\_mgr.abort\_transaction）。*

*检查 Undo Log 是否正确回放（test\_redo\_undo\_log.py 中的 test\_undo）。*



--3.并发事务

客1BEGIN;

UPDATE student SET age = 25 WHERE id = 2;

-- 不提交

客2SELECT age FROM student WHERE id = 2; -- 预期返回更新前的值（22）

客户端 1 执行 COMMIT 后，客户端 2 再次查询：

SELECT age FROM student WHERE id = 2; -- 预期返回25

*验证并发事务隔离性（对应 test\_concurrency.py），未提交事务的修改对其他事务不可见。*



###### 小结：该测试用例覆盖范围

功能模块           	测试点	                                                           对应代码模块

SQL 解析	    DDL/DML 语法解析、错误处理     	           sql/parser、test\_sql\_parser

执行计划	    逻辑计划、物理计划生成	                        sql/optimizier、test\_planner

执行引擎	    各类算子（Scan/Join/Agg 等）	                       executor、test\_executor

存储引擎 	    表 / 索引数据读写、B + 树操作	     storage、test\_storage、test\_bplustree

事务管理	    提交 / 回滚、Redo/Undo Log  storage/transaction、test\_redo\_undo\_log

并发控制	    多线程事务隔离	                                                     test\_concurrency

元数据         表 / 索引元数据存储

管理

