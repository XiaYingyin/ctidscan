# ctidscan

ctidscan是一个使用Custom Scan Provider API的demo。他提供了一个使用不等运算符访问系统列ctid的功能。其中ctid是PostgreSQL中的一个系统列，用来表示数据记录的物理行信息，指的是一条记录位于哪个数据块的哪个位移上面。 跟oracle中伪列rowid 的意义一样的，只是形式不一样。

# 如何加载

这个模块是通过扩展的形式集成到Postgresql中的，只需要使用下面的命令即可加载该模块：

```
postgres=# load 'ctidscan';
```

另外，你也可以使用`guc-session-preload-libraries`或者`guc-shared-preload-libraries`配置参数在postgresql.conf配置文件中加载。

```
shared_preload_libraries = '$libdir/ctidscan'
```

注意，修改为配置文件需要重启PostgreSQL：

```
sudo systemctl restart postgresql.service
```

为了不影响其他人正常使用PostgreSQL，建议使用第一种方法，局部加载扩展。

即使加载了扩展之后，仍然可以通过开关参数，开启或关闭（on/off）该模块：

```
postgres=# set enable_ctidscan=off;
```

# 如何使用

当用户通过**不等**运算符，如`>`、`<`等访问系统列`ctid`时，planner会自动选择代价最低的路径，也就是通过ctidscan访问`ctid`。

比如下面这个例子，首先创建一个表t1（第一列为主键ID，第二列为随机字符串），然后插入随机数据。

```
CREATE TABLE t1 (a int primary key, b text);
INSERT INTO t1 (SELECT s, md5(s::text) FROM generate_series(1,400) s);
```
加载ctidscan：

```
load 'ctidscan';
```

下面使用不等运算符访问该表的ctid系统列，比如查出0号block中item ID小于5的所有记录：

```
select ctid, * from t1 where ctid<'(0,5)';
```

结果如下：

```
INFO:  begin ctidscan extension
INFO:  end ctidscan extension
 ctid  | a |                b
-------+---+----------------------------------
 (0,1) | 1 | c4ca4238a0b923820dcc509a6f75849b
 (0,2) | 2 | c81e728d9d4c2f636f067f89cc14862c
 (0,3) | 3 | eccbc87e4b5ce2fe28308fd9f2a7baf3
 (0,4) | 4 | a87ff679a2f3e71d9181a67b7542122c
(4 rows)
```


