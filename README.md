CHANGELOG
=========

##### 1.删除mysqld空闲连接超时断开的错误日志
mysql-server的连接长时间空闲后会断开连接(默认8小时)，此时mysql-otp会打出各种error log,这些错误日志我们并不需要，所以在这里稍作修改，把它去掉了。我们是直接使用的mysql_poolboy项目(该项目也需要稍作修改去掉重连的错误日志)

##### 2.修改mysqld挂了之后的处理
mysql-server挂了之后，mysql_poolboy_sup会不停的重启`mysql(gen_server)`，达到上限后supervisor也会退出，或者导致整个应用都退出。此次主要修改`mysql.erl`文件，使mysql-server挂了之后，`mysql(gen_server)`不会退出，而是每隔一段时间尝试连接mysql-server。

##### 我为什么选择mysql-otp:
`erlang-mysql-driver VS emysql VS mysql-otp`, 这三者我们比较了一下
- `erlang-mysql-driver`的连接池就是一个数组循环利用，不是真正的连接池，当出现大量连续请求时，很容易照成部分连接空闲，而部分连接有多条请求等待处理。
- `emysql`与`erlang-mysql-driver`相比拥有真正的连接池，但是无transaction，后来我们自己添加了transaction，用了一段时间后，有一次发现连接长时间闲置后自动断开不会重连，然后导致各种错误；后来发现了`mysql-otp`我们就舍弃了它
- `mysql-otp` 断连后自动重连，有transaction，配合poolboy使用，我们直接用的mysql_poolboy这个项目


---
原项目地址：https://github.com/mysql-otp/mysql-otp

---

MySQL/OTP
=========

[![Build Status](https://travis-ci.org/mysql-otp/mysql-otp.svg)](https://travis-ci.org/mysql-otp/mysql-otp)

MySQL/OTP is a driver for connecting Erlang/OTP applications to MySQL
databases (version 4.1 and upward). It is a native implementation of the MySQL
protocol in Erlang.

Some of the features:

* Mnesia style transactions:
  * Nested transactions are implemented using savepoints.
  * Transactions are automatically retried when deadlocks are detected.
* Uses the binary protocol for prepared statements.
* Each connection is a gen_server, which makes it compatible with Poolboy (for
  connection pooling) and ordinary OTP supervisors.
* No records in the public API.
* Slow queries are interrupted without killing the connection (MySQL version
  ≥ 5.0.0).

See also:

* [API documenation](//mysql-otp.github.io/mysql-otp/index.html) (Edoc)
* [Test coverage](//mysql-otp.github.io/mysql-otp/eunit.html) (EUnit)
* [Why another MySQL driver?](https://github.com/mysql-otp/mysql-otp/wiki#why-another-mysql-driver) in the wiki
* [MySQL/OTP + Poolboy](https://github.com/mysql-otp/mysql-otp-poolboy):
  A simple application that combines MySQL/OTP with Poolboy for connection
  pooling.

Synopsis
--------

```Erlang
%% Connect
{ok, Pid} = mysql:start_link([{host, "localhost"}, {user, "foo"},
                              {password, "hello"}, {database, "test"}]),

%% Select
{ok, ColumnNames, Rows} =
    mysql:query(Pid, <<"SELECT * FROM mytable WHERE id = ?">>, [1]),

%% Manipulate data
ok = mysql:query(Pid, "INSERT INTO mytable (id, bar) VALUES (?, ?)", [1, 42]),

%% Separate calls to fetch more info about the last query
LastInsertId = mysql:insert_id(Pid),
AffectedRows = mysql:affected_rows(Pid),
WarningCount = mysql:warning_count(Pid),

%% Mnesia style transaction (nestable)
Result = mysql:transaction(Pid, fun () ->
    ok = mysql:query(Pid, "INSERT INTO mytable (foo) VALUES (1)"),
    throw(foo),
    ok = mysql:query(Pid, "INSERT INTO mytable (foo) VALUES (1)")
end),
case Result of
    {atomic, ResultOfFun} ->
        io:format("Inserted 2 rows.~n");
    {aborted, Reason} ->
        io:format("Inserted 0 rows.~n")
end

%% Multiple queries and multiple result sets
{ok, [{[<<"foo">>], [[42]]}, {[<<"bar">>], [[<<"baz">>]]}]} =
    mysql:query(Pid, "SELECT 42 AS foo; SELECT 'baz' AS bar;"),

%% Graceful timeout handling: SLEEP() returns 1 when interrupted
{ok, [<<"SLEEP(5)">>], [[1]]} =
    mysql:query(Pid, <<"SELECT SLEEP(5)">>, 1000),
```

Usage as a dependency
---------------------

Using *erlang.mk*:

    DEPS = mysql
    dep_mysql = git https://github.com/mysql-otp/mysql-otp 1.2.0

Using *rebar*:

    {deps, [
        {mysql, ".*", {git, "https://github.com/mysql-otp/mysql-otp",
                       {tag, "1.2.0"}}}
    ]}.

Contributing
------------

Run the eunit tests with `make tests`. For the suite `mysql_tests` you
need MySQL running on localhost and give privileges to the `otptest` user:

```SQL
grant all privileges on otptest.* to otptest@localhost identified by 'otptest';
```

If you run `make tests COVER=1` a coverage report will be generated. Open
`cover/index.html` to see that any lines you have added or modified are covered
by a test.

Linebreak code to 80 characters per line and follow a coding style similar to
that of existing code.

Keep commit messages short and descriptive. Each commit message should describe
the purpose of the commit, the feature added or bug fixed, so that the commit
log can be used as a comprehensive change log. [CHANGELOG.md](CHANGELOG.md) is
generated from the commit messages.

License
-------

GNU Lesser General Public License (LGPL) version 3 or any later version.
Since the LGPL is a set of additional permissions on top of the GPL, both
license texts are included in the files [COPYING](COPYING) and
[COPYING.LESSER](COPYING.LESSER) respectively.

We hope this license should be permissive enough while remaining copyleft. If
you're having issues with this license, please create an issue in the issue
tracker!
