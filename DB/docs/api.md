# 交互接口介绍
   - 数据库SQL语句的总入口: `exec_imoocdb_query(sql_stmt: str) -> Result` 用于接收用户的访问SQL，协调数据库内各组件
   - SQL引擎SQL解析部分的入口：`query_parse(sql_stmt: str) -> ASTNode` ，用于解析SQL语句，返回AST
   - SQL引擎SQL优化过程的入口：`query_plan(ast: ASTNode) -> PhysicalOperator`，用于将解析后的AST转换为具体的执行计划
   - 执行引擎的入口：`exec_plan(plan_tree: PhysicalOperator) -> Result`，用于将具体的执行计划进行执行，并返回结果
   