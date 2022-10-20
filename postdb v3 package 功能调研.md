---
title: postdb v3 package 功能调研
tags: 
grammar_cjkRuby: true
---
# opengauss实现

 - package只支持集中式，无法在分布式中使用。
 - 在package specification中声明过的函数或者存储过程，必须在package body中找到定义。
 - 在实例化中，无法调用带有commit/rollback的存储过程。
 - 不能在Trigger中调用package函数。
 - 不能在外部SQL中直接使用package当中的变量。
 - 不允许在package外部调用package的私有变量和存储过程。
 - 不支持其它存储过程不支持的用法，例如，在function中不允许调用commit/rollback，则package的function中同样无法调用commit/rollback。
 - 不支持schema与package同名。
 - 只支持A风格的存储过程和函数定义。
 - 不支持package内有同名变量，包括包内同名参数。
 - package的全局变量为session级，不同session之间package的变量不共享。
 - package中调用自治事务的函数，不允许使用package中的cursor变量，以及递归的使用package中cursor变量的函数。
 - package中不支持声明ref cursor变量。
 - package默认为SECURITY INVOKER权限，如果想将默认行为改为SECURITY DEFINER权限，需要设置guc参数behavior_compat_options='plsql_security_definer'。
 - 被授予CREATE ANY PACKAGE权限的用户，可以在public模式和用户模式下创建PACKAGE。
 - 如果需要创建带有特殊字符的package名，特殊字符中不能含有空格，并且最好设置GUC参数behavior_compat_options=“skip_insert_gs_source”,否则可能引起报错。


# AnalyticDB PostgreSQL

- 可以使用开源工具ora2pg进行最初的Oracle应用转换。您可以使用ora2pg将Oracle的package等语法转换成PostgreSQL兼容的语法。

PL/PGSQL不支持Package，需要把Package转换成schema，同时Package里面的所有procedure和function也需要转换成AnalyticDB PostgreSQL的function。

```
CREATE OR REPLACE PACKAGE pkg IS 

…

END;
```
转换为：

```
CREATE SCHEMA pkg;
```

- Package定义的变量
Procedure/Function的局部变量保持不变，全局变量在AnalyticDB PostgreSQL中可以使用临时表进行保存。

- Package初始化块
请删除，若无法删除请使用Function封装，在需要的时候主动调用该Function。

- Package内定义的Procedure/Function
Package内定义的Procedure和Function需要转成AnalyticDB PostgreSQL的Function，并把Function定义到Package对应的Schema内。

例如，有一个Package名为pkg中有如下函数：
```
FUNCTION test_func (args int) RETURN int is
var number := 10;
BEGIN
… …
END;
```

转换成如下AnalyticDB PostgreSQL的Function：

```
CREATE OR REPLACE FUNCTION pkg. test_func(args int) RETURNS int AS
$$

  … …

$$
 LANGUAGE plpgsql;
```


# IvorySQL

Packages in IvorySQL database. This support includes the following:

- Support of Oracle compatible syntax for function and procedure creations.
- Support of Package creation and persistence using the oracle syntax,
  (including package specification, package body and constructor/initializer)
- Support of PL/iSQL language. This language is added to implement the PL/SQL
  statements in PostgreSQL. Most of the common statements have been added.
- Support of DROP PACKAGE statement.
- Support of GRANT and REVOKE of privileges on Packages.
- Support of Packages syntax in PSQL client. So that oracle style packages
  can be created from PSQL.
- Support of complete Package compilation.
- Support of keeping a Package compilation state in-memory to avoid
  further repeated compilations and also access information from this
  in-memory state.
- Support of Package constructors/initializer compilation and execution.
- Support of Cursors declaration and creation at package level.
- Support of Package level variables (This includes the package level
  variables to be accessed from any pl-block of plisql language).
- Support for package record types.
- Support of package ref cursors.
- Support of Definer/Invoker rights for packages
- Support of pg_dump and pg_restore
- Support of ALTER PACKAGE OWNER statement.
- Test coverage

问题：
目前看还有报bug，不清楚稳定程度如何。


# IvorySQL Package功能
## pg14 commit: sql standard function

```
SQL-standard function body

This adds support for writing CREATE FUNCTION and CREATE PROCEDURE
statements for language SQL with a function body that conforms to the
SQL standard and is portable to other implementations.

Instead of the PostgreSQL-specific AS $$ string literal $$ syntax,
this allows writing out the SQL statements making up the body
unquoted, either as a single statement:

    CREATE FUNCTION add(a integer, b integer) RETURNS integer
        LANGUAGE SQL
        RETURN a + b;

or as a block

    CREATE PROCEDURE insert_data(a integer, b integer)
    LANGUAGE SQL
    BEGIN ATOMIC
      INSERT INTO tbl VALUES (a);
      INSERT INTO tbl VALUES (b);
    END;

The function body is parsed at function definition time and stored as
expression nodes in a new pg_proc column prosqlbody.  So at run time,
no further parsing is required.

However, this form does not support polymorphic arguments, because
there is no more parse analysis done at call time.

Dependencies between the function and the objects it uses are fully
tracked.

A new RETURN statement is introduced.  This can only be used inside
function bodies.  Internally, it is treated much like a SELECT
statement.

psql needs some new intelligence to keep track of function body
boundaries so that it doesn't send off statements when it sees
semicolons that are inside a function body.

Tested-by: Jaime Casanova <jcasanov@systemguards.com.ec>
Reviewed-by: Julien Rouhaud <rjuju123@gmail.com>
Discussion: https://www.postgresql.org/message-id/flat/1c11f1eb-f00c-43b7-799d-2d44132c02d7@2ndquadrant.com
```