
## SQL执行顺序
SQL的执行顺序：from---where--group by---having---select---order by

## MySQL是如何执行一条SQL的
![SQL执行的全部过程-MlS1d5](https://gitee.com/dreamcater/blog-img/raw/master/uPic/SQL执行的全部过程-MlS1d5.png)

**MySQL内部可以分为服务层和存储引擎层两部分：**

1. **服务层包括连接器、查询缓存、分析器、优化器、执行器等**，涵盖MySQL的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。
2. **存储引擎层负责数据的存储和提取**，其架构模式是插件式的，支持InnoDB、MyISAM、Memory等多个存储引擎。现在最常用的存储引擎是InnoDB，它从MySQL 5.5.5版本开始成为了默认的存储引擎。

**Server层按顺序执行sql的步骤为**：
客户端请求:
- **连接器**（验证用户身份，给予权限） 
- **查询缓存**（存在缓存则直接返回，不存在则执行后续操作）
- **分析器**（对SQL进行词法分析和语法分析操作） 
- **优化器**（主要对执行的sql优化选择最优的执行方案方法） 
- **执行器**（执行时会先看用户是否有执行权限，有才去使用这个引擎提供的接口）
- **去引擎层获取数据返回**（如果开启查询缓存则会缓存查询结果）