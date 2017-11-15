# Go-MySQL-Driver

 Go 连接mysql的驱动包 [database/sql](https://golang.org/pkg/database/sql/) 

![Go-MySQL-Driver logo](https://raw.github.com/wiki/go-sql-driver/mysql/gomysql_m.png "Golang Gopher holding the MySQL Dolphin")

---------------------------------------
  * [特征](#特征)
  * [要求](#要求)
  * [安装](#安装)
  * [用法](#用法)
    * [DSN (数据源名称)](#dsn-数据源名称)
      * [密码](#密码)
      * [协议](#协议)
      * [地址](#地址)
      * [参数](#参数)
      * [例子](#例子)
    * [连接池和超时](#连接池和超时)
    * [支持context.Context](#支持-contextcontext)
    * [支持ColumnType](#支持-columntype)
    * [支持数据加载 ](#load-data-local-infile-support)
    * [支持time.Time](#支持-timetime)
    * [支持 Unicode](#支持-unicode)
  * [测试 / 发展](#测试--发展)
  * [许可证](#许可证)

---------------------------------------

## 特征
  * 轻量级且 [效率快](https://github.com/go-sql-driver/sql-benchmark "golang MySQL-Driver performance")
  * 纯go语言编写
  * 可用 TCP/IPv4, TCP/IPv6, Unix 域协议 或 [自定义协议]连接(https://godoc.org/github.com/go-sql-driver/mysql#DialFunc)
  * 自动处理空闲连接
  * 自动连接池 *(by database/sql package)*
  * 支持大于16MB的查询
  * 完全支持 [`sql.RawBytes`](https://golang.org/pkg/database/sql/#RawBytes) .
  * 智能 `批量` 处理预处理语句
  * 支持白名单文件 和 `io.Reader` 的`数据加载` 
  * 可选 `time.Time` 转换
  * 可选占位符插值

## 要求
  * 最低支持 Go 1.3.
  * MySQL (4.1+), MariaDB, Percona Server, Google CloudSQL 或 Sphinx (2.2.3+)

---------------------------------------

## 安装
Simple install the package to your [$GOPATH](https://github.com/golang/go/wiki/GOPATH "GOPATH") with the [go tool](https://golang.org/cmd/go/ "go command") from shell:
```bash
$ go get -u github.com/go-sql-driver/mysql
```
Make sure [Git is installed](https://git-scm.com/downloads) on your machine and in your system's `PATH`.

## 用法
_Go MySQL Driver_ is an implementation of Go's `database/sql/driver` interface. You only need to import the driver and can use the full [`database/sql`](https://golang.org/pkg/database/sql/) API then.

Use `mysql` as `driverName` and a valid [DSN](#dsn-data-source-name)  as `dataSourceName`:
```go
import "database/sql"
import _ "github.com/go-sql-driver/mysql"

db, err := sql.Open("mysql", "user:password@/dbname")
```

[Examples are available in our Wiki](https://github.com/go-sql-driver/mysql/wiki/Examples "Go-MySQL-Driver Examples").


### DSN (数据源名称)

The Data Source Name has a common format, like e.g. [PEAR DB](http://pear.php.net/manual/en/package.database.db.intro-dsn.php) uses it, but without type-prefix (optional parts marked by squared brackets):
```
[username[:password]@][protocol[(address)]]/dbname[?param1=value1&...&paramN=valueN]
```

A DSN in its fullest form:
```
username:password@protocol(address)/dbname?param=value
```

Except for the databasename, all values are optional. So the minimal DSN is:
```
/dbname
```

If you do not want to preselect a database, leave `dbname` empty:
```
/
```
This has the same effect as an empty DSN string:
```

```

Alternatively, [Config.FormatDSN](https://godoc.org/github.com/go-sql-driver/mysql#Config.FormatDSN) can be used to create a DSN string by filling a struct.

#### 密码
Passwords can consist of any character. Escaping is **not** necessary.

#### 协议
See [net.Dial](https://golang.org/pkg/net/#Dial) for more information which networks are available.
In general you should use an Unix domain socket if available and TCP otherwise for best performance.

#### 地址
For TCP and UDP networks, addresses have the form `host[:port]`.
If `port` is omitted, the default port will be used.
If `host` is a literal IPv6 address, it must be enclosed in square brackets.
The functions [net.JoinHostPort](https://golang.org/pkg/net/#JoinHostPort) and [net.SplitHostPort](https://golang.org/pkg/net/#SplitHostPort) manipulate addresses in this form.

For Unix domain sockets the address is the absolute path to the MySQL-Server-socket, e.g. `/var/run/mysqld/mysqld.sock` or `/tmp/mysql.sock`.

#### 参数
*Parameters are case-sensitive!*

Notice that any of `true`, `TRUE`, `True` or `1` is accepted to stand for a true boolean value. Not surprisingly, false can be specified as any of: `false`, `FALSE`, `False` or `0`.

##### `allowAllFiles`

```
Type:           bool
Valid Values:   true, false
Default:        false
```

`allowAllFiles=true` disables the file Whitelist for `LOAD DATA LOCAL INFILE` and allows *all* files.
[*Might be insecure!*](http://dev.mysql.com/doc/refman/5.7/en/load-data-local.html)

##### `allowCleartextPasswords`

```
Type:           bool
Valid Values:   true, false
Default:        false
```

`allowCleartextPasswords=true` allows using the [cleartext client side plugin](http://dev.mysql.com/doc/en/cleartext-authentication-plugin.html) if required by an account, such as one defined with the [PAM authentication plugin](http://dev.mysql.com/doc/en/pam-authentication-plugin.html). Sending passwords in clear text may be a security problem in some configurations. To avoid problems if there is any possibility that the password would be intercepted, clients should connect to MySQL Server using a method that protects the password. Possibilities include [TLS / SSL](#tls), IPsec, or a private network.

##### `allowNativePasswords`

```
Type:           bool
Valid Values:   true, false
Default:        true
```
`allowNativePasswords=false` disallows the usage of MySQL native password method.

##### `allowOldPasswords`

```
Type:           bool
Valid Values:   true, false
Default:        false
```
`allowOldPasswords=true` allows the usage of the insecure old password method. This should be avoided, but is necessary in some cases. See also [the old_passwords wiki page](https://github.com/go-sql-driver/mysql/wiki/old_passwords).

##### `charset`

```
Type:           string
Valid Values:   <name>
Default:        none
```

Sets the charset used for client-server interaction (`"SET NAMES <value>"`). If multiple charsets are set (separated by a comma), the following charset is used if setting the charset failes. This enables for example support for `utf8mb4` ([introduced in MySQL 5.5.3](http://dev.mysql.com/doc/refman/5.5/en/charset-unicode-utf8mb4.html)) with fallback to `utf8` for older servers (`charset=utf8mb4,utf8`).

Usage of the `charset` parameter is discouraged because it issues additional queries to the server.
Unless you need the fallback behavior, please use `collation` instead.

##### `collation`

```
Type:           string
Valid Values:   <name>
Default:        utf8_general_ci
```

Sets the collation used for client-server interaction on connection. In contrast to `charset`, `collation` does not issue additional queries. If the specified collation is unavailable on the target server, the connection will fail.

A list of valid charsets for a server is retrievable with `SHOW COLLATION`.

##### `clientFoundRows`

```
Type:           bool
Valid Values:   true, false
Default:        false
```

`clientFoundRows=true` causes an UPDATE to return the number of matching rows instead of the number of rows changed.

##### `columnsWithAlias`

```
Type:           bool
Valid Values:   true, false
Default:        false
```

When `columnsWithAlias` is true, calls to `sql.Rows.Columns()` will return the table alias and the column name separated by a dot. For example:

```
SELECT u.id FROM users as u
```

will return `u.id` instead of just `id` if `columnsWithAlias=true`.

##### `interpolateParams`

```
Type:           bool
Valid Values:   true, false
Default:        false
```

If `interpolateParams` is true, placeholders (`?`) in calls to `db.Query()` and `db.Exec()` are interpolated into a single query string with given parameters. This reduces the number of roundtrips, since the driver has to prepare a statement, execute it with given parameters and close the statement again with `interpolateParams=false`.

*This can not be used together with the multibyte encodings BIG5, CP932, GB2312, GBK or SJIS. These are blacklisted as they may [introduce a SQL injection vulnerability](http://stackoverflow.com/a/12118602/3430118)!*

##### `loc`

```
Type:           string
Valid Values:   <escaped name>
Default:        UTC
```

Sets the location for time.Time values (when using `parseTime=true`). *"Local"* sets the system's location. See [time.LoadLocation](https://golang.org/pkg/time/#LoadLocation) for details.

Note that this sets the location for time.Time values but does not change MySQL's [time_zone setting](https://dev.mysql.com/doc/refman/5.5/en/time-zone-support.html). For that see the [time_zone system variable](#system-variables), which can also be set as a DSN parameter.

Please keep in mind, that param values must be [url.QueryEscape](https://golang.org/pkg/net/url/#QueryEscape)'ed. Alternatively you can manually replace the `/` with `%2F`. For example `US/Pacific` would be `loc=US%2FPacific`.

##### `maxAllowedPacket`
```
Type:          decimal number
Default:       4194304
```

Max packet size allowed in bytes. The default value is 4 MiB and should be adjusted to match the server settings. `maxAllowedPacket=0` can be used to automatically fetch the `max_allowed_packet` variable from server *on every connection*.

##### `multiStatements`

```
Type:           bool
Valid Values:   true, false
Default:        false
```

Allow multiple statements in one query. While this allows batch queries, it also greatly increases the risk of SQL injections. Only the result of the first query is returned, all other results are silently discarded.

When `multiStatements` is used, `?` parameters must only be used in the first statement.

##### `parseTime`

```
Type:           bool
Valid Values:   true, false
Default:        false
```

`parseTime=true` changes the output type of `DATE` and `DATETIME` values to `time.Time` instead of `[]byte` / `string`


##### `readTimeout`

```
Type:           duration
Default:        0
```

I/O read timeout. The value must be a decimal number with a unit suffix (*"ms"*, *"s"*, *"m"*, *"h"*), such as *"30s"*, *"0.5m"* or *"1m30s"*.

##### `rejectReadOnly`

```
Type:           bool
Valid Values:   true, false
Default:        false
```


`rejectReadOnly=true` causes the driver to reject read-only connections. This
is for a possible race condition during an automatic failover, where the mysql
client gets connected to a read-only replica after the failover.

Note that this should be a fairly rare case, as an automatic failover normally
happens when the primary is down, and the race condition shouldn't happen
unless it comes back up online as soon as the failover is kicked off. On the
other hand, when this happens, a MySQL application can get stuck on a
read-only connection until restarted. It is however fairly easy to reproduce,
for example, using a manual failover on AWS Aurora's MySQL-compatible cluster.

If you are not relying on read-only transactions to reject writes that aren't
supposed to happen, setting this on some MySQL providers (such as AWS Aurora)
is safer for failovers.

Note that ERROR 1290 can be returned for a `read-only` server and this option will
cause a retry for that error. However the same error number is used for some
other cases. You should ensure your application will never cause an ERROR 1290
except for `read-only` mode when enabling this option.


##### `timeout`

```
Type:           duration
Default:        OS default
```

Timeout for establishing connections, aka dial timeout. The value must be a decimal number with a unit suffix (*"ms"*, *"s"*, *"m"*, *"h"*), such as *"30s"*, *"0.5m"* or *"1m30s"*.


##### `tls`

```
Type:           bool / string
Valid Values:   true, false, skip-verify, <name>
Default:        false
```

`tls=true` enables TLS / SSL encrypted connection to the server. Use `skip-verify` if you want to use a self-signed or invalid certificate (server side). Use a custom value registered with [`mysql.RegisterTLSConfig`](https://godoc.org/github.com/go-sql-driver/mysql#RegisterTLSConfig).


##### `writeTimeout`

```
Type:           duration
Default:        0
```

I/O write timeout. The value must be a decimal number with a unit suffix (*"ms"*, *"s"*, *"m"*, *"h"*), such as *"30s"*, *"0.5m"* or *"1m30s"*.


##### 系统变量

其它参数可解析为系统变量:
  * `<boolean_var>=<value>`: `SET <boolean_var>=<value>`
  * `<enum_var>=<value>`: `SET <enum_var>=<value>`
  * `<string_var>=%27<value>%27`: `SET <string_var>='<value>'`

规则:
* 字符串变量的值必须引用 `'`.
* 值必须是 [url.QueryEscape](http://golang.org/pkg/net/url/#QueryEscape)'ed!
 (which implies values of string variables must be wrapped with `%27`).

例子:
  * `autocommit=1`: `SET autocommit=1`
  * [`time_zone=%27Europe%2FParis%27`](https://dev.mysql.com/doc/refman/5.5/en/time-zone-support.html): `SET time_zone='Europe/Paris'`
  * [`tx_isolation=%27REPEATABLE-READ%27`](https://dev.mysql.com/doc/refman/5.5/en/server-system-variables.html#sysvar_tx_isolation): `SET tx_isolation='REPEATABLE-READ'`


#### 例子
```
user@unix(/path/to/socket)/dbname
```

```
root:pw@unix(/tmp/mysql.sock)/myDatabase?loc=Local
```

```
user:password@tcp(localhost:5555)/dbname?tls=skip-verify&autocommit=true
```

Treat warnings as errors by setting the system variable [`sql_mode`](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html):
```
user:password@/dbname?sql_mode=TRADITIONAL
```

TCP via IPv6:
```
user:password@tcp([de:ad:be:ef::ca:fe]:80)/dbname?timeout=90s&collation=utf8mb4_unicode_ci
```

TCP on a remote host, e.g. Amazon RDS:
```
id:password@tcp(your-amazonaws-uri.com:3306)/dbname
```

Google Cloud SQL on App Engine (First Generation MySQL Server):
```
user@cloudsql(project-id:instance-name)/dbname
```

Google Cloud SQL on App Engine (Second Generation MySQL Server):
```
user@cloudsql(project-id:regionname:instance-name)/dbname
```

TCP using default port (3306) on localhost:
```
user:password@tcp/dbname?charset=utf8mb4,utf8&sys_var=esc%40ped
```

Use the default protocol (tcp) and host (localhost:3306):
```
user:password@/dbname
```

No Database preselected:
```
user:password@/
```


### 连接池和超时
The connection pool is managed by Go's database/sql package. For details on how to configure the size of the pool and how long connections stay in the pool see `*DB.SetMaxOpenConns`, `*DB.SetMaxIdleConns`, and `*DB.SetConnMaxLifetime` in the [database/sql documentation](https://golang.org/pkg/database/sql/). The read, write, and dial timeouts for each individual connection are configured with the DSN parameters [`readTimeout`](#readtimeout), [`writeTimeout`](#writetimeout), and [`timeout`](#timeout), respectively.

## `ColumnType` 支持
This driver supports the [`ColumnType` interface](https://golang.org/pkg/database/sql/#ColumnType) introduced in Go 1.8, with the exception of [`ColumnType.Length()`](https://golang.org/pkg/database/sql/#ColumnType.Length), which is currently not supported.

## `context.Context` 支持
Go 1.8 added `database/sql` support for `context.Context`. This driver supports query timeouts and cancellation via contexts.
See [context support in the database/sql package](https://golang.org/doc/go1.8#database_sql) for more details.


### `LOAD DATA LOCAL INFILE` 支持
For this feature you need direct access to the package. Therefore you must change the import path (no `_`):
```go
import "github.com/go-sql-driver/mysql"
```

Files must be whitelisted by registering them with `mysql.RegisterLocalFile(filepath)` (recommended) or the Whitelist check must be deactivated by using the DSN parameter `allowAllFiles=true` ([*Might be insecure!*](http://dev.mysql.com/doc/refman/5.7/en/load-data-local.html)).

To use a `io.Reader` a handler function must be registered with `mysql.RegisterReaderHandler(name, handler)` which returns a `io.Reader` or `io.ReadCloser`. The Reader is available with the filepath `Reader::<name>` then. Choose different names for different handlers and `DeregisterReaderHandler` when you don't need it anymore.

See the [godoc of Go-MySQL-Driver](https://godoc.org/github.com/go-sql-driver/mysql "golang mysql driver documentation") for details.


### `time.Time` 支持
The default internal output type of MySQL `DATE` and `DATETIME` values is `[]byte` which allows you to scan the value into a `[]byte`, `string` or `sql.RawBytes` variable in your program.

However, many want to scan MySQL `DATE` and `DATETIME` values into `time.Time` variables, which is the logical opposite in Go to `DATE` and `DATETIME` in MySQL. You can do that by changing the internal output type from `[]byte` to `time.Time` with the DSN parameter `parseTime=true`. You can set the default [`time.Time` location](https://golang.org/pkg/time/#Location) with the `loc` DSN parameter.

**Caution:** As of Go 1.1, this makes `time.Time` the only variable type you can scan `DATE` and `DATETIME` values into. This breaks for example [`sql.RawBytes` support](https://github.com/go-sql-driver/mysql/wiki/Examples#rawbytes).

Alternatively you can use the [`NullTime`](https://godoc.org/github.com/go-sql-driver/mysql#NullTime) type as the scan destination, which works with both `time.Time` and `string` / `[]byte`.


### 支持 unicode
Since version 1.1 Go-MySQL-Driver automatically uses the collation `utf8_general_ci` by default.

Other collations / charsets can be set using the [`collation`](#collation) DSN parameter.

Version 1.0 of the driver recommended adding `&charset=utf8` (alias for `SET NAMES utf8`) to the DSN to enable proper UTF-8 support. This is not necessary anymore. The [`collation`](#collation) parameter should be preferred to set another collation / charset than the default.

See http://dev.mysql.com/doc/refman/5.7/en/charset-unicode.html for more details on MySQL's Unicode support.

## 测试 / 发展
To run the driver tests you may need to adjust the configuration. See the [Testing Wiki-Page](https://github.com/go-sql-driver/mysql/wiki/Testing "Testing") for details.

Go-MySQL-Driver is not feature-complete yet. Your help is very appreciated.
If you want to contribute, you can work on an [open issue](https://github.com/go-sql-driver/mysql/issues?state=open) or review a [pull request](https://github.com/go-sql-driver/mysql/pulls).

See the [Contribution Guidelines](https://github.com/go-sql-driver/mysql/blob/master/CONTRIBUTING.md) for details.

---------------------------------------

## 许可证
Go-MySQL-Driver 许可遵循 [Mozilla Public License Version 2.0](https://raw.github.com/go-sql-driver/mysql/master/LICENSE)

Mozilla 许可范围:
> MPL: The copyleft applies to any files containing MPLed code.


That means:
  * You can **use** the **unchanged** source code both in private and commercially.
  * When distributing, you **must publish** the source code of any **changed files** licensed under the MPL 2.0 under a) the MPL 2.0 itself or b) a compatible license (e.g. GPL 3.0 or Apache License 2.0).
  * You **needn't publish** the source code of your library as long as the files licensed under the MPL 2.0 are **unchanged**.

Please read the [MPL 2.0 FAQ](https://www.mozilla.org/en-US/MPL/2.0/FAQ/) if you have further questions regarding the license.

You can read the full terms here: [LICENSE](https://raw.github.com/go-sql-driver/mysql/master/LICENSE).

![Go Gopher and MySQL Dolphin](https://raw.github.com/wiki/go-sql-driver/mysql/go-mysql-driver_m.jpg "Golang Gopher transporting the MySQL Dolphin in a wheelbarrow")

