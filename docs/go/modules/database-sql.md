# database/sql

## 必读说明

数据库接口：

* `database/sql` 是 Go 语言标准库中的一个包，提供了一组通用的接口、类型和方法等

* `sqlx` 是一个在 `database/sql` 包的基础上封装的库，提供了一些更高级的功能和便利的接口

数据库驱动：

* `go-sql-driver/mysql` 是一个第三方的`MySQL`驱动程序，实现了 `database/sql` 包所定义的接口

<br />

## 参考资料

Go SQL接口说明：[https://github.com/golang/go/wiki/SQLInterface](https://github.com/golang/go/wiki/SQLInterface)

Go SQL驱动列表：[https://github.com/golang/go/wiki/SQLDrivers](https://github.com/golang/go/wiki/SQLDrivers)

**database/sql**

* 文档：[https://pkg.go.dev/database/sql](https://pkg.go.dev/database/sql)

**sqlx**

* 文档：[https://pkg.go.dev/github.com/jmoiron/sqlx](https://pkg.go.dev/github.com/jmoiron/sqlx)
* Github：[https://github.com/jmoiron/sqlx](https://github.com/jmoiron/sqlx)

**go-sql-driver/mysql**

* 文档：[https://pkg.go.dev/github.com/go-sql-driver/mysql](https://pkg.go.dev/github.com/go-sql-driver/mysql)
* Github：[https://github.com/go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)

<br />

## 安装依赖

```bash
go get github.com/jmoiron/sqlx
go get github.com/go-sql-driver/mysql
```

<br />

## 连接

### 连接示例

::: details （1）database/sql连接MySQL

```go
package main

import (
	"database/sql"
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sql.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库
	db, err := sql.Open("mysql", mysqlConfig.FormatDSN())
	if err != nil {
		return nil, err
	}

	// 验证连接是否有效
	if err := db.Ping(); err != nil {
		return nil, err
	}

	return db, nil
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 查看数据库版本
	var version string
	err = db.QueryRow("SELECT CONCAT_WS(' ', @@version, @@version_comment)").Scan(&version)
	if err != nil {
		panic(err)
	}
	fmt.Println(version)
}
```

:::

::: details （2）sqlx连接MySQL

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库
	// sqlx.Connect = sqlx.Open + db.Ping,也可以使用 sql.MustConnect, 连接不成功就panic
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 查看数据库版本 - 兼容database/sql
	{
		var version string
		err = db.QueryRow("SELECT CONCAT_WS(' ', @@version, @@version_comment)").Scan(&version)
		if err != nil {
			panic(err)
		}
		fmt.Println(version)
	}

	// 查看数据库版本 - sqlx特有的查询方式,其内部本质也是 QueryRow + Scan 的方式
	{
		var version string
		err = db.Get(&version, "SELECT CONCAT_WS(' ', @@version, @@version_comment)")
		if err != nil {
			panic(err)
		}
		fmt.Println(version)
	}
}
```

输出结果

```bash
8.0.30 MySQL Community Server - GPL
8.0.30 MySQL Community Server - GPL
```

:::

<br />

### 简单分析

::: details （1）MySQL 驱动注册逻辑

```go
// github.com/go-sql-driver/mysql init方法用于注册驱动
func init() {
	sql.Register("mysql", &MySQLDriver{})
}

// database/sql包提供的注册逻辑
var (
	driversMu sync.RWMutex
	drivers   = make(map[string]driver.Driver)
)

func Register(name string, driver driver.Driver) {
	driversMu.Lock()
	defer driversMu.Unlock()
	if driver == nil {
		panic("sql: Register driver is nil")
	}
	if _, dup := drivers[name]; dup {
		panic("sql: Register called twice for driver " + name)
	}
	drivers[name] = driver
}

// driver.Driver接口, name就是注册时的name
type Driver interface {
	Open(name string) (Conn, error)
}
```

:::

::: details （2）sql.Open函数

```go
func Open(driverName, dataSourceName string) (*DB, error) {
    // 查找驱动
	driversMu.RLock()
	driveri, ok := drivers[driverName]
	driversMu.RUnlock()
	if !ok {
		return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
	}

    // 调用 OpenConnector，然后将返回值作为OpenDB的参数，最后返回一个*DB结构体
	if driverCtx, ok := driveri.(driver.DriverContext); ok {
		connector, err := driverCtx.OpenConnector(dataSourceName)
		if err != nil {
			return nil, err
		}
		return OpenDB(connector), nil
	}

	return OpenDB(dsnConnector{dsn: dataSourceName, driver: driveri}), nil
}
```

:::

::: details （3）sqlx.Connect函数

```go
// Connect内部调用了Open和Ping函数
// 注意这里的Open并不是sql.Open
func Connect(driverName, dataSourceName string) (*DB, error) {
	db, err := Open(driverName, dataSourceName)
	if err != nil {
		return nil, err
	}
	err = db.Ping()
	if err != nil {
		db.Close()
		return nil, err
	}
	return db, nil
}

// Open内部调用了sql.Open,然后对sql.DB做了一个简单的封装
// Open is the same as sql.Open, but returns an *sqlx.DB instead.
func Open(driverName, dataSourceName string) (*DB, error) {
	db, err := sql.Open(driverName, dataSourceName)
	if err != nil {
		return nil, err
	}
	return &DB{DB: db, driverName: driverName, Mapper: mapper()}, err
}
```

:::

<br />

### 连接池设置

::: details （1）设置最大打开的连接数：SetMaxOpenConns

```go
package main

import (
	"log"
	"sync"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库
	// sqlx.Connect = sqlx.Open + db.Ping,也可以使用 sql.MustConnect, 连接不成功就panic
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 设置最大打开的连接数
	// 当 n <= 0时表示不限制打开的连接数,默认值为 0
	db.SetMaxOpenConns(2)

	// 当达到最大连接后，再去打开新连接，会阻塞，直到一段时间后仍旧获取不到连接则报超时错误
	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_, err := db.Exec("SELECT SLEEP(60);")
			if err != nil {
				log.Printf("Exec error: %#v\n", err)
			}
		}()
		time.Sleep(time.Second)
		log.Printf("%#v\n\n", db.Stats())
	}
	wg.Wait()
}
```

输出结果

```bash
2023/04/01 10:45:23 sql.DBStats{MaxOpenConnections:2, OpenConnections:1, InUse:1, Idle:0, WaitCount:0, WaitDuration:0, MaxIdleClosed:0, MaxIdleTimeClosed:0, MaxLifetimeClosed:0}

2023/04/01 10:45:24 sql.DBStats{MaxOpenConnections:2, OpenConnections:2, InUse:2, Idle:0, WaitCount:0, WaitDuration:0, MaxIdleClosed:0, MaxIdleTimeClosed:0, MaxLifetimeClosed:0}

2023/04/01 10:45:25 sql.DBStats{MaxOpenConnections:2, OpenConnections:2, InUse:2, Idle:0, WaitCount:1, WaitDuration:0, MaxIdleClosed:0, MaxIdleTimeClosed:0, MaxLifetimeClosed:0}

[mysql] 2023/04/01 10:45:52 packets.go:37: read tcp 192.168.48.1:55526->192.168.48.151:3306: i/o timeout
2023/04/01 10:45:52 Exec error: &errors.errorString{s:"invalid connection"}
[mysql] 2023/04/01 10:45:53 packets.go:37: read tcp 192.168.48.1:55527->192.168.48.151:3306: i/o timeout
2023/04/01 10:45:53 Exec error: &errors.errorString{s:"invalid connection"}
[mysql] 2023/04/01 10:46:22 packets.go:37: read tcp 192.168.48.1:55528->192.168.48.151:3306: i/o timeout
2023/04/01 10:46:22 Exec error: &errors.errorString{s:"invalid connection"}

# 分析
# 1、当达到最大连接后，再去打开新连接，会阻塞，直到一段时间后仍旧获取不到连接则报错 mysql.ErrInvalidConn
# 2、当我们将 SELECT SLEEP(60); 修改为 SELECT SLEEP(30); 甚至更低时，可以及时释放连接，不会报错
# 3、注意上面我们并没有考虑到MySQL可接收的最大连接数，可以通过 SHOW VARIABLES LIKE 'max_connections'; 查询
# 4、当达到MySQL最大连接数后会报错 &mysql.MySQLError{Number:0x410, SQLState:[5]uint8{0x0, 0x0, 0x0, 0x0, 0x0}, Message:"Too many connections"}

# 总结：为了提高性能，这个值应该尽量大一点，但并不是越大越好
```

:::

::: details （2）设置最大空闲的连接数：SetMaxIdleConns

```go
package main

import (
	"log"
	"sync"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库
	// sqlx.Connect = sqlx.Open + db.Ping,也可以使用 sql.MustConnect, 连接不成功就panic
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 设置最大空闲的连接数
	// 如果 n <= 0，则不保留任何空闲连接
	// 如果 n >0，并且大于 MaxOpenConns，则 MaxIdleConns 减少到和MaxOpenConns一致
	// 默认值是2
	db.SetMaxIdleConns(10)

	// 开1000个连接，然后等待查询执行完成后，再查看剩余(空闲)的连接个数
	var wg sync.WaitGroup
	for i := 0; i < 1000; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_, err := db.Exec("SELECT SLEEP(3)")
			if err != nil {
				log.Printf("Exec error: %#v\n", err)
			}
		}()
		log.Printf("%#v\n\n", db.Stats())
	}
	wg.Wait()
	log.Printf("%#v\n\n", db.Stats())
}
```

输出结果

```bash
# 重点看最后一行, OpenConnections:10, MaxIdleClosed:990
...
2023/04/01 11:29:33 sql.DBStats{MaxOpenConnections:0, OpenConnections:997, InUse:997, Idle:0, WaitCount:0, WaitDuration:0, MaxIdleClosed:0, MaxIdleTimeClosed:0, MaxLifetimeClosed:0}  

2023/04/01 11:29:33 sql.DBStats{MaxOpenConnections:0, OpenConnections:999, InUse:999, Idle:0, WaitCount:0, WaitDuration:0, MaxIdleClosed:0, MaxIdleTimeClosed:0, MaxLifetimeClosed:0}  

2023/04/01 11:29:33 sql.DBStats{MaxOpenConnections:0, OpenConnections:999, InUse:999, Idle:0, WaitCount:0, WaitDuration:0, MaxIdleClosed:0, MaxIdleTimeClosed:0, MaxLifetimeClosed:0}  

2023/04/01 11:29:37 sql.DBStats{MaxOpenConnections:0, OpenConnections:10, InUse:0, Idle:10, WaitCount:0, WaitDuration:0, MaxIdleClosed:990, MaxIdleTimeClosed:0, MaxLifetimeClosed:0}

# 总结：为了提高性能，这个值应该尽量大一点，但并不是越大越好
```

:::

::: details （3）设置连接最长的空闲时间：SetConnMaxIdleTime

```go
package main

import (
	"log"
	"sync"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库
	// sqlx.Connect = sqlx.Open + db.Ping,也可以使用 sql.MustConnect, 连接不成功就panic
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 设置最大空闲的连接数
	db.SetMaxIdleConns(20)

	// 设置连接最长的空闲时间
	// 如果连接空闲达到该时长将会被关闭
    // 如果 d <= 0，则连接不会因连接空闲时间而关闭，默认为0
	db.SetConnMaxIdleTime(time.Second * 15)

	// 开100个连接，然后让他一直空闲
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_, err := db.Exec("SELECT SLEEP(1)")
			if err != nil {
				log.Printf("Exec error: %#v\n", err)
			}
		}()
	}
	wg.Wait()

	// 实时查看连接状态
	for {
		time.Sleep(time.Second)
		log.Printf("%#v\n\n", db.Stats())
	}
}
```

输出结果

```bash
# 重点看 MaxIdleTimeClosed
...
2023/04/01 11:44:44 sql.DBStats{MaxOpenConnections:0, OpenConnections:0, InUse:0, Idle:0, WaitCount:0, WaitDuration:0, MaxIdleClosed:80, MaxIdleTimeClosed:20, MaxLifetimeClosed:0}
```

:::

::: details （4）设置连接的最大生命周期：SetConnMaxLifetime

```go
	// 设置连接的最大生命周期
	// SetConnMaxLifetime 方法接受一个 time.Duration 类型的参数，表示连接的最大生存时间
	// 如果一个连接的生存时间超过了该时间，那么该连接将被关闭并从连接池中移除
	// 在下次需要连接时，将会创建一个新的连接

	// 作用是
	// 当数据库连接被重复利用时，连接的状态可能会变得不可预测，例如会话状态可能会发生变化
	// 因此为了确保应用程序的稳定性和性能，建议对连接池中的连接进行适当地管理和维护，以确保连接的安全和有效性
	db.SetConnMaxLifetime(time.Hour * 6)
```

:::

<br />

### 身份认证插件

::: details （1）身份认证插件介绍

**身份认证插件介绍**

* mysql_native_password

  MySQL 8.0之前的默认身份插件，使用用户名和密码进行验证，使用SHA1算法进行密码加密，认证过程中不会缓存密码，不支持SSL/TLS

* caching_sha2_password

  MySQL 8.0之后的默认身份插件，使用用户名和密码进行验证，使用SHA256算法进行密码加密，在认证成功后将哈希值缓存到内存中，支持SSL/TLS

**go-sql-driver/mysql支持的身份认证插件**

* 默认情况下使用caching_sha2_password插件，不支持mysql_native_password插件
* 将AllowNativePasswords设置为true，可以同时支持caching_sha2_password和mysql_native_password

<br />

**查询当前数据库默认的身份认证插件**

```bash
mysql> SELECT @@default_authentication_plugin;
+---------------------------------+
| @@default_authentication_plugin |
+---------------------------------+
| caching_sha2_password           |
+---------------------------------+
1 row in set, 1 warning (0.00 sec)
```

**修改当前数据库默认的身份认证插件**

```bash
# 1、修改MySQL配置文件
[mysqld]
default_authentication_plugin = mysql_native_password

# 2、重启MySQL

# 3、查询验证
mysql> SELECT @@default_authentication_plugin;
+---------------------------------+
| @@default_authentication_plugin |
+---------------------------------+
| mysql_native_password           |
+---------------------------------+
1 row in set, 1 warning (0.00 sec)
```

:::

::: details （2）AllowNativePasswords测试

```bash
# 依据上面的文档和代码做一些简单的调整即可
#   1、修改MySQL默认的身份认证插件为 mysql_native_password
#   2、修改代码中的MySQL配置参数 AllowNativePasswords: false,

# 执行代码后会报错，设置AllowNativePasswords: true后就可以正常连接MySQL
[mysql] 2023/04/03 11:07:09 connector.go:95: could not use requested auth plugin 'mysql_native_password': this user requires mysql native password authentication.
panic: this user requires mysql native password authentication.
```

:::

<br />

## 日志

### 使用 zap

::: details （1）使用zap.Logger替换go-sql-driver/mysql内部的Logger

```go
package main

import (
	"fmt"
	"sync"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库
	// sqlx.Connect = sqlx.Open + db.Ping,也可以使用 sql.MustConnect, 连接不成功就panic
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

type mysqlLogger struct {
	logger *zap.Logger
}

func (l *mysqlLogger) Print(v ...any) {
	l.logger.Error(fmt.Sprint(v...))
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 初始化 zap.Logger
	logger, err := zap.NewProduction()
	if err != nil {
		panic(err)
	}
	logger = logger.
		WithOptions(zap.AddStacktrace(zapcore.FatalLevel)).
		WithOptions(zap.WithCaller(false)).
		With(zap.String("driver", "go-sql-driver/mysql"))

	// 使用zap.Logger替换go-sql-driver/mysql内部的Logger
	err = mysql.SetLogger(&mysqlLogger{logger: logger})
	if err != nil {
		panic(err)
	}

	// 人为制造一个go-sql-driver/mysql内部错误
	var wg sync.WaitGroup
	db.SetMaxOpenConns(2)
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_, _ = db.Exec("SELECT sleep(60)")
		}()
	}
	wg.Wait()
}
```

输出结果

```bash
{"level":"error","ts":1680347876.7655728,"msg":"read tcp 192.168.48.1:59902->192.168.48.151:3306: i/o timeout","driver":"go-sql-driver/mysql"}
{"level":"error","ts":1680347876.7655728,"msg":"read tcp 192.168.48.1:59901->192.168.48.151:3306: i/o timeout","driver":"go-sql-driver/mysql"}
{"level":"error","ts":1680347906.7748804,"msg":"read tcp 192.168.48.1:59903->192.168.48.151:3306: i/o timeout","driver":"go-sql-driver/mysql"}

# 需要注意的是，我们给zap添加了一个固定字段，{"driver": "go-sql-driver/mysql"}，这样就能很方便的区分出该日志由MySQL驱动打印
```

:::

<br />

## 操作

### 对库操作

::: details 点击查看详情

```go
package main

import (
	"database/sql"
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库
	// sqlx.Connect = sqlx.Open + db.Ping,也可以使用 sql.MustConnect, 连接不成功就panic
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

// ShowDatabase 查看当前使用的库
func ShowDatabase(db *sqlx.DB) {
	// 因为查询的值有可能是Null，所以这里必须使用 sql.NullString
	var database sql.NullString
	err := db.QueryRow("SELECT DATABASE()").Scan(&database)
	if err != nil {
		panic(err)
	}
	if database.Valid {
		fmt.Printf("%-20s%s\n", "Current Database:", database.String)
	} else {
		fmt.Printf("%-20s%s\n", "Current Database:", "未选择")
	}
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 1、连接数据库时可以不填写具体的库名
	// 2、可以通过执行SQL语句创建、切换和删除某个库
	// 3、若连接时指定库名 或 执行SQL切换到库，则要求库必须存在，则否会报错
	//    Error 1049 (42000): Unknown database 'demo123'

	// 查看当前使用的库
	ShowDatabase(db)

	// 若库不存在则创建库
	_, err = db.Exec("CREATE DATABASE IF NOT EXISTS demo")
	if err != nil {
		panic(err)
	}

	// 使用库
	_, err = db.Exec("USE demo")
	if err != nil {
		panic(err)
	}

	// 查看当前使用的库
	ShowDatabase(db)

	// 查看所有的库
	var databaseList []string
	err = db.Select(&databaseList, "SHOW DATABASES ;")
	if err != nil {
		panic(err)
	}
	fmt.Printf("%-20s%s\n", "Database List:", databaseList)
}

```

输出结果

```bash
Current Database:   未选择
Current Database:   demo
Database List:      [demo information_schema mysql performance_schema sys]
```

:::

<br />

### 对表操作

::: details （1）表操作

```go
package main

import (
	"database/sql"
	"fmt"
	"strings"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// DescStruct 表结构
type DescStruct struct {
	Field   string         `db:"Field"`
	Type    string         `db:"Type"`
	Null    string         `db:"Null"`
	Key     string         `db:"Key"`
	Default sql.NullString `db:"Default"`
	Extra   string         `db:"Extra"`
}

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库
	// sqlx.Connect = sqlx.Open + db.Ping,也可以使用 sql.MustConnect, 连接不成功就panic
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 若users表存在则删除
	{
		var sqlString = `DROP TABLE IF EXISTS users;`
		_, err := db.Exec(sqlString)
		if err != nil {
			panic(err)
		}
	}

	// 若users表不存在则创建
	{
		var sqlString = `
		CREATE TABLE IF NOT EXISTS users (
			·id·         int auto_increment,
			·username·   varchar(128) not null,
			·password·   varchar(255) not null,
			·email·      varchar(128) not null,
			·created_at· datetime(6) not null,
			·updated_at· datetime(6) not null,
			·deleted_at· datetime(6) null default null,
			PRIMARY KEY (·id·),
			UNIQUE (·username·),
  			UNIQUE (·email·)
		)`
		sqlString = strings.ReplaceAll(sqlString, "·", "`")
		_, err := db.Exec(sqlString)
		if err != nil {
			panic(err)
		}
	}

	// 查看users表结构
	{
		// 执行SQL语句
		var desc []DescStruct
		err := db.Select(&desc, "DESC users")
		if err != nil {
			panic(err)
		}

		// 表结构可视化
		format := "%-20s%-20s%-10s%-10s%-20v%-20s\n"
		fmt.Printf(format, "Field", "Type", "Null", "Key", "Default", "Extra")
		fmt.Printf("%s\n", strings.Repeat("_", 100))
		for _, row := range desc {
			// Default字段的值可能为NULL,所以需要特殊处理
			defaultString := ""
			if row.Default.Valid {
				defaultString = row.Default.String
			} else {
				defaultString = "NULL"
			}
			fmt.Printf(format, row.Field, row.Type, row.Null, row.Key, defaultString, row.Extra)
		}
	}
}
```

输出结果

```bash
Field               Type                Null      Key       Default             Extra
____________________________________________________________________________________________________
id                  int                 NO        PRI       NULL                auto_increment
username            varchar(128)        NO        UNI       NULL
password            varchar(255)        NO                  NULL
email               varchar(128)        NO        UNI       NULL
created_at          datetime(6)         NO                  NULL
updated_at          datetime(6)         NO                  NULL
deleted_at          datetime(6)         YES                 NULL
```

:::

<br />

### 写入数据

::: details （1）写数据的几种方法

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// User 定义结构体
type User struct {
	ID        int       `db:"id"`
	Username  string    `db:"username"`
	Password  string    `db:"password"`
	Email     string    `db:"email"`
	CreatedAt time.Time `db:"created_at"`
	UpdatedAt time.Time `db:"updated_at"`
	DeletedAt time.Time `db:"deleted_at"`
}

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库: sqlx.Connect = sqlx.Open(不会真正连接数据库) + db.Ping(会真正连接数据库)
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 写入单条数据 - 方式1，使用?占位
	{
		user := User{Username: "alice", Password: "123456", Email: "alice@example.com", CreatedAt: time.Now(), UpdatedAt: time.Now()}
		sqlString := `INSERT INTO users (username, password, email, created_at, updated_at) VALUES (?, ?, ?,?,?)`
		_, err := db.Exec(sqlString, user.Username, user.Password, user.Email, user.CreatedAt, user.UpdatedAt)
		if err != nil {
			panic(err)
		}
	}

	// 写入单条数据 - 方式2，:username等写法需要对应结构体的tag: db
	{
		user := User{Username: "john", Password: "123456", Email: "john@example.com", CreatedAt: time.Now(), UpdatedAt: time.Now()}
		sqlString := `INSERT INTO users (username, password, email, created_at, updated_at) VALUES
			(:username, :password, :email, :created_at, :updated_at)`
		_, err := db.NamedExec(sqlString, user)
		if err != nil {
			panic(err)
		}
	}

	// 批量写入数据
	{
		users := []User{
			{Username: "bob1", Password: "123456", Email: "bob1@example.com", CreatedAt: time.Now(), UpdatedAt: time.Now()},
			{Username: "bob2", Password: "123456", Email: "bob2@example.com", CreatedAt: time.Now(), UpdatedAt: time.Now()},
			{Username: "bob3", Password: "123456", Email: "bob3@example.com", CreatedAt: time.Now(), UpdatedAt: time.Now()},
			{Username: "bob4", Password: "123456", Email: "bob4@example.com", CreatedAt: time.Now(), UpdatedAt: time.Now()},
			{Username: "bob5", Password: "123456", Email: "bob5@example.com", CreatedAt: time.Now(), UpdatedAt: time.Now()},
		}
		sqlString := `INSERT INTO users (username, password, email, created_at, updated_at) VALUES
			(:username, :password, :email, :created_at, :updated_at)`

		result, err := db.NamedExec(sqlString, users)
		if err != nil {
			panic(err)
		}

		// 返回受update, insert, or delete操作影响的行数
		// 不是每个数据库或数据库驱动程序都支持这一点
		n, err := result.RowsAffected()
		if err != nil {
			panic(err)
		}
		fmt.Printf("RowsAffected: %d\n", n)

		// 返回上一次(本次)插入操作生成的自增ID
		// 不是每个数据库或数据库驱动程序都支持这一点
		m, err := result.LastInsertId()
		if err != nil {
			panic(err)
		}
		fmt.Printf("LastInsertId: %d\n", m)
	}
}
```

输出结果

```bash
RowsAffected: 5    # 受影响的行数
LastInsertId: 3    # 上一次插入ID(这里的ID是自增主键)

# 我们来看一下数据库的信息
# 当插入多行数据时，LastInsertId是多行的第一行的ID
mysql> select * from users;
+----+----------+----------+-------------------+----------------------------+----------------------------+------------+
| id | username | password | email             | created_at                 | updated_at                 | deleted_at |
+----+----------+----------+-------------------+----------------------------+----------------------------+------------+
|  1 | alice    | 123456   | alice@example.com | 2023-04-03 18:00:37.703388 | 2023-04-03 18:00:37.703388 | NULL       |
|  2 | john     | 123456   | john@example.com  | 2023-04-03 18:00:37.710173 | 2023-04-03 18:00:37.710173 | NULL       |
|  3 | bob1     | 123456   | bob1@example.com  | 2023-04-03 18:00:37.713542 | 2023-04-03 18:00:37.713542 | NULL       |
|  4 | bob2     | 123456   | bob2@example.com  | 2023-04-03 18:00:37.713542 | 2023-04-03 18:00:37.713542 | NULL       |
|  5 | bob3     | 123456   | bob3@example.com  | 2023-04-03 18:00:37.713542 | 2023-04-03 18:00:37.713542 | NULL       |
|  6 | bob4     | 123456   | bob4@example.com  | 2023-04-03 18:00:37.713542 | 2023-04-03 18:00:37.713542 | NULL       |
|  7 | bob5     | 123456   | bob5@example.com  | 2023-04-03 18:00:37.713542 | 2023-04-03 18:00:37.713542 | NULL       |
|  8 | faker    | 123456   | faker@example.com | 2023-04-03 18:00:37.716145 | 2023-04-03 18:00:37.716145 | NULL       |
+----+----------+----------+-------------------+----------------------------+----------------------------+------------+
8 rows in set (0.00 sec)
```

:::

::: details （2）批量写入大量虚假数据

```go
package main

import (
	"fmt"
	"github.com/brianvoe/gofakeit/v6"
	"sync"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// User 定义结构体
type User struct {
	ID        int       `db:"id"`
	Username  string    `db:"username"`
	Password  string    `db:"password"`
	Email     string    `db:"email"`
	CreatedAt time.Time `db:"created_at"`
	UpdatedAt time.Time `db:"updated_at"`
	DeletedAt time.Time `db:"deleted_at"`
}

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库: sqlx.Connect = sqlx.Open(不会真正连接数据库) + db.Ping(会真正连接数据库)
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 统计时间
	start := time.Now()
	defer func() {
		fmt.Printf("Used %.0f seconds", time.Since(start).Seconds())
	}()

	// 生成数据，200万条
	ch := make(chan User, 1024)
	go func() {
		for i := 0; i < 200*10000; i++ {
			user := User{
				Username:  gofakeit.Username(),
				Password:  gofakeit.Password(true, true, true, false, false, 12),
				Email:     gofakeit.Email(),
				CreatedAt: time.Now(),
				UpdatedAt: time.Now(),
			}
			ch <- user
		}
		close(ch)
	}()

	// 写入数据库,并发100
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				user, ok := <-ch
				if !ok {
					break
				}
				sqlString := `INSERT INTO users (username, password, email, created_at, updated_at) VALUES
						(:username, :password, :email, :created_at, :updated_at)`
				_, _ = db.NamedExec(sqlString, user)
			}
		}()
	}
	wg.Wait()
}
```

输出结果

```bash
# 待补充
```

:::

<br />

### 修改数据

::: details 点击查看详情

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则, 不区分大小写
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库: sqlx.Connect = sqlx.Open(不会真正连接数据库) + db.Ping(会真正连接数据库)
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 总结
	// 1、在修改数据时候，时间最好不要使用数据库的now()函数 和 程序的now() 混合使用，这有可能会因为时区导致获取错误的时间
	// 2、SQL语句中的大小写是否敏感 和 排序规则有关

	// 修改数据: 将ID为8的用户名修改为 Alice
	{
		result, err := db.Exec("UPDATE users SET username=?, updated_at=? WHERE id = 8", "Alice", time.Now())
		if err != nil {
			panic(err)
		}
		n, err := result.RowsAffected()
		if err != nil {
			panic(err)
		}
		fmt.Printf("受影响的行数: %d\n", n)
	}

	// 修改数据1: utf8mb4_general_ci默认不区分大小写，所以这里的更改总会生效
	{
		result, err := db.Exec("UPDATE users SET username = ?, updated_at=? WHERE username = ?", "jack", time.Now(), "alice")
		if err != nil {
			panic(err)
		}
		n, err := result.RowsAffected()
		if err != nil {
			panic(err)
		}
		fmt.Printf("受影响的行数: %d\n", n)
	}

	// 修改数据2: 注释上面的代码，在不修改MySQL排序规则的情况下，添加 binary 关键字,这样就会区分大小写
	{
		result, err := db.Exec("UPDATE users SET username = ?, updated_at=? WHERE binary username = ?", "bob", time.Now(), "alice")
		if err != nil {
			panic(err)
		}
		n, err := result.RowsAffected()
		if err != nil {
			panic(err)
		}
		fmt.Printf("受影响的行数: %d\n", n)
	}

	// 修改数据3: 排除逻辑删除数据(deleted_at不是null的情况)
	{
		result, err := db.Exec("UPDATE users SET username=?, updated_at=? WHERE deleted_at is null and id = 9", "Alice", time.Now())
		if err != nil {
			panic(err)
		}
		n, err := result.RowsAffected()
		if err != nil {
			panic(err)
		}
		fmt.Printf("受影响的行数: %d\n", n)
	}
}
```

输出结果

```bash
受影响的行数: 1
受影响的行数: 1
受影响的行数: 0
受影响的行数: 1
```

:::

<br />

### 删除数据

::: details 点击查看详情

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库: sqlx.Connect = sqlx.Open(不会真正连接数据库) + db.Ping(会真正连接数据库)
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 删除数据：硬删除
	{
		result, err := db.Exec("DELETE FROM users WHERE id = ?", "8")
		if err != nil {
			panic(err)
		}
		n, err := result.RowsAffected()
		if err != nil {
			panic(err)
		}
		fmt.Printf("受影响的行数: %d\n", n)
	}

	// 删除数据：软删除
	{
		result, err := db.Exec("UPDATE users SET deleted_at = ? WHERE id = ?", time.Now(), "9")
		if err != nil {
			panic(err)
		}
		n, err := result.RowsAffected()
		if err != nil {
			panic(err)
		}
		fmt.Printf("受影响的行数: %d\n", n)
	}
}
```

输出结果

```bash
受影响的行数: 1
受影响的行数: 1
```

:::

<br />

### 查询数据

::: details （1）查询数据：低级接口：Query*

```go
package main

import (
	"database/sql"
	"fmt"
	"log"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// User 定义结构体
type User struct {
	ID       int    `db:"id"`
	Name     string `db:"name"`
	Password string `db:"password"`
	Email    string `db:"email"`
}

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库: sqlx.Connect = sqlx.Open(不会真正连接数据库) + db.Ping(会真正连接数据库)
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer db.Close()

	// 注意事项：
	// 目标位置的类型必须与查询结果的结构相匹配，否则会导致运行时错误

	// 查询多条数据: 使用 Query
	{
		rows, err := db.Query("SELECT id,name,password,email FROM users WHERE id > ?", "42")
		if err != nil {
			panic(err)
		}
		for rows.Next() {
			user := User{}
			err := rows.Scan(&user.ID, &user.Name, &user.Password, &user.Email)
			if err != nil {
				panic(err)
			}
			fmt.Printf("%#v\n", user)
		}
		fmt.Println()
	}

	// 查询多条数据: 使用 Queryx, 可以使用 StructScan、SliceScan、MapScan映射到不同的对象中
	{
		rows, err := db.Queryx("SELECT id,name,password,email FROM users WHERE id > ?", "4")
		if err != nil {
			panic(err)
		}
		for rows.Next() {
			user := User{}
			err := rows.StructScan(&user)
			if err != nil {
				log.Fatalln(err)
			}
			fmt.Printf("%#v\n", user)
		}
		fmt.Println()
	}

	// 查询单条数据: 使用 QueryRow
	{
		user := User{}
		row := db.QueryRow("SELECT id,name,password,email FROM users WHERE name = ?", "bob5")
		err := row.Scan(&user.ID, &user.Name, &user.Password, &user.Email)
		if err != sql.ErrNoRows {
			if err != nil {
				panic(err)
			}
			fmt.Printf("%#v\n", user)
		}
		fmt.Println()
	}

	// 查询单条数据: 使用 QueryRowx
	{
		user := User{}
		row := db.QueryRowx("SELECT id,name,password,email FROM users WHERE name = ?", "bob5")
		err := row.StructScan(&user)
		if err != nil {
			panic(err)
		}
		fmt.Printf("%#v\n", user)
		fmt.Println()
	}
}
```

输出结果

```bash
main.User{ID:5, Name:"bob3", Password:"123456", Email:"bob3@example.com"}
main.User{ID:6, Name:"bob4", Password:"123456", Email:"bob4@example.com"}
```

:::

::: details （2）查询数据：高级接口：Get / Select

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// User 定义结构体
type User struct {
	ID       int    `db:"id"`
	Name     string `db:"name"`
	Password string `db:"password"`
	Email    string `db:"email"`
}

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库: sqlx.Connect = sqlx.Open(不会真正连接数据库) + db.Ping(会真正连接数据库)
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer db.Close()

	// 注意事项：
	// 目标位置的类型必须与查询结果的结构相匹配，否则会导致运行时错误

	// 查询单条数据: Get，参数要求是一个结构体指针
	{
		user := User{}
		err := db.Get(&user, "SELECT id,name,password,email FROM users WHERE name=?", "bob4")
		if err != nil {
			panic(err)
		}
		fmt.Printf("%#v\n", user)
		fmt.Println()
	}

	// 查询多条数据: Select，参数要求是一个 结构体切片的指针
	{
		users := []User{}
		err := db.Select(&users, "SELECT id,name,password,email FROM users WHERE id > ?", "4")
		if err != nil {
			panic(err)
		}
		for _, user := range users {
			fmt.Printf("%#v\n", user)
		}
		fmt.Println()
	}
}
```

:::

<br />

### 使用事物

::: details 点击查看详情

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// User 定义结构体
type User struct {
	ID       int    `db:"id"`
	Name     string `db:"name"`
	Password string `db:"password"`
	Email    string `db:"email"`
}

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库: sqlx.Connect = sqlx.Open(不会真正连接数据库) + db.Ping(会真正连接数据库)
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func Rollback(tx *sqlx.Tx) {
	err := tx.Rollback()
	if err != nil {
		fmt.Println("事物回滚失败")
	} else {
		fmt.Println("事物回滚成功")
	}
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 开启事物
	// Begin开启事物 ：只能执行 Exec 和 Query 方法
	// Beginx开启事物: 支持Exec, Query, Get, Select 等方法
	tx, err := db.Beginx()
	if err != nil {
		panic(err)
	}

	// 执行操作
	user1 := User{Name: "John1", Password: "123456", Email: "john3@example.com"}
	_, err = tx.NamedExec("INSERT INTO users (name, password, email) VALUES (:name, :password, :email)", user1)
	if err != nil {
		Rollback(tx)
	}

	user2 := User{Name: "John2", Password: "123456", Email: "john4@example.com"}
	_, err = tx.NamedExec("INSERT INTO users (name, password, email) VALUES (:name, :password, :email)", user2)
	if err != nil {
		Rollback(tx)
	}

	// 提交
	err = tx.Commit()
	if err != nil {
		Rollback(tx)
	}
}
```

:::

<br />

### 预处理

::: details 点击查看详情

```go
package main

import (
	"fmt"
	"time"

	"github.com/go-sql-driver/mysql"
	"github.com/jmoiron/sqlx"
)

// User 定义结构体
type User struct {
	ID       int    `db:"id"`
	Name     string `db:"name"`
	Password string `db:"password"`
	Email    string `db:"email"`
}

// ConnMySQL 连接数据库
func ConnMySQL() (*sqlx.DB, error) {
	// 定义MySQL配置
	mysqlConfig := mysql.Config{
		User:                 "root",
		Passwd:               "QiNqg[l.%;H>>rO9",
		Net:                  "tcp",
		Addr:                 "192.168.48.151:3306",
		DBName:               "demo",
		Collation:            "utf8mb4_general_ci", // 设置字符集和排序规则
		Loc:                  time.Local,           // 设置连接时使用的时区,默认为UTC时区
		ParseTime:            true,                 // 是否将数据库中的TIME或DATETIME字段解析为Go的时间类型（即time.Time)
		Timeout:              5 * time.Second,      // 连接超时时间
		ReadTimeout:          30 * time.Second,     // 读取超时时间
		WriteTimeout:         30 * time.Second,     // 写入超时时间
		CheckConnLiveness:    true,                 // 在使用连接之前检查其存活性
		AllowNativePasswords: true,                 // 允许MySQL身份认证插件mysql_native_password
	}

	// 连接数据库: sqlx.Connect = sqlx.Open(不会真正连接数据库) + db.Ping(会真正连接数据库)
	return sqlx.Connect("mysql", mysqlConfig.FormatDSN())
}

func main() {
	// 连接数据库
	db, err := ConnMySQL()
	if err != nil {
		panic(err)
	}
	defer func() { _ = db.Close() }()

	// 定义预处理语句
	stmp, err := db.Preparex("SELECT id,name,password,email FROM users WHERE id = ?")
	if err != nil {
		panic(err)
	}

	// 使用预处理语句
	{
		u := User{}
		err = stmp.Get(&u, "1")
		if err != nil {
			panic(err)
		}
		fmt.Printf("%#v\n", u)
	}

	{
		u := User{}
		err = stmp.Get(&u, "2")
		if err != nil {
			panic(err)
		}
		fmt.Printf("%#v\n", u)
	}
}
```

:::

<br />

## 统计信息

::: details （1）模拟MySQL客户端内置命令status

```go

```

:::
