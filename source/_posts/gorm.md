---
title: gorm一些笔记
date: 2017-12-02 21:09:07
tags:
 - golang
 - gorm
categories:
 - 学习笔记
---

官网 <https://github.com/jinzhu/gorm>

## 数据表名

gorm默认将golang基于`帕斯卡命名法`的model的类名转换成`下划线命名法`格式作为数据表名字

若不一致，可以实现类方法`TableName`方法
``` go
func (StoreLite) TableName() string {
	return "stores"
}
```

这个转换的方法有放开
``` go
// ToDBName convert string to db name
func ToDBName(name string) string
```

写个工具函数通过一个对象获取它的表名
``` go
import (
	"github.com/jinzhu/gorm"
	"github.com/jinzhu/inflection"
)
func DBName(obj interface{}) string {
	t := reflect.TypeOf(obj)
	_, ok := t.MethodByName("TableName")
	if !ok { //没找到
		if t.Kind() == reflect.Ptr {
			t = t.Elem()
		}
		return inflection.Plural(gorm.ToDBName(t.Name()))
	} else {
		v := reflect.ValueOf(obj).MethodByName("TableName").Call([]reflect.Value{})
		return v[0].Interface().(string)
	}
}
```

通过外键关联查询时，会需要使用类名，参见<http://jinzhu.me/gorm/associations.html#has-one>`Related`
``` go
// User has one CreditCard, UserID is the foreign key
type User struct {
    gorm.Model
    CreditCard   CreditCard
}

type CreditCard struct {
    gorm.Model
    UserID   uint
    Number   string
}

var card CreditCard
db.Model(&user).Related(&card, "CreditCard")
```

go默认的反射的类名具有完整的包路径
``` go
func DBClassName(obj interface{}) string {
	t := reflect.TypeOf(obj)
	if t.Kind() != reflect.Struct {
		t = t.Elem()
	}
	name := filepath.Ext(t.String())
	return name[1:]
}
```

## Select
gorm默认是`select(*)`,有些情况可能会导致较多的不必要数据传输和性能损耗。解决方案是`Select`
``` go
db.Select("name, age").Find(&users)
//// SELECT name, age FROM users;

db.Select([]string{"name", "age"}).Find(&users)
//// SELECT name, age FROM users;

db.Table("users").Select("COALESCE(age,?)", 42).Rows()
//// SELECT COALESCE(age,'42') FROM users;
```

在代码中写死列名不是个好办法
``` go
// 过滤出需要查询的字段，若json:"-"，则忽略之
// 背景： 默认情况下grom总是查询所有字段select(*)，而实际情况并不需要那么多
func FieldsFilter(obj interface{}) []string {
	dbName := DBName(obj)
	value := reflect.TypeOf(obj)
	if value.Kind() == reflect.Ptr {
		value = value.Elem()
	}

	if value.Kind() != reflect.Struct {
		panic("must be struct object")
	}

	var res []string
	return fieldFilterImp(value, res, dbName)
}

func fieldFilterImp(t reflect.Type, res []string, dbName string) []string {
	count := t.NumField()
	for i := 0; i < count; i++ {
		field := t.Field(i)
		if field.Anonymous {
			res = fieldFilterImp(field.Type, res, dbName)
			continue
		}

		tag := field.Tag.Get("json")
		name := parseTag(tag)
		if name == "" {
			name = field.Name
		}
		if name == "-" {
			continue
		}

		tag = field.Tag.Get("field")
		if tag == "-" {
			continue
		} else if tag != "" {
			res = append(res, tag)
		} else {
			// 考虑到join之后可能有列的重名，所以加上db name
			res = append(res, fmt.Sprintf("%s.%s", dbName, name))
		}

	}
	return res
}
```

## 记录不存在的错误
当查询不存在时，同样返回错误，通过函数`RecordNotFound`判断是否属于这种情况

## Count
gorm默认使用`select count(*)`，相比select count('id')效率低那么一点，可以优化

``` go
func GetObjectCount(obj interface{}, db *gorm.DB) (int, error) {
	var cnt int
	temp_db := db.Model(obj).Select("count(\"id\") as count").Count(&cnt)
	if temp_db.Error != nil {
		return 0, temp_db.Error
	} else {
		return cnt, nil
	}
}
```

## 更新部分
可以使用`Select/Omit`来选择/排除更新的列

``` go
db.Model(&user).Select("name").Updates(map[string]interface{}{
	"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET name='hello',
//  updated_at='2013-11-17 21:34:10' WHERE id=111;

db.Model(&user).Omit("name").Updates(map[string]interface{}{
	"name": "hello", "age": 18, "actived": false})
//// UPDATE users SET age=18, actived=false,
//  updated_at='2013-11-17 21:34:10' WHERE id=111;
```

同样利用上面的工具函数，可以过滤可更新的列 数据


## 批量更新
当需要一并插入多条数据，sql批量插入的写法比较省
``` sql
INSERT INTO tbl_name
    (a,b,c)
VALUES
    (1,2,3),
    (4,5,6),
    (7,8,9);
```

不过gorm目前不支持<https://github.com/jinzhu/gorm/issues/255>

## last_insert_id 

gorm自动实现了`last_insert_id`,即便在事务模式下，后面的sql都可以直接取ID

``` go
tmpDB := gd.DB.Begin()
if err := service.CreateObject(apartment, tmpDB); err != nil {
	tmpDB.Rollback()
	util.ResponseDbError(http.StatusOK, c, err)
	return
}

for k, v := range param.ImageUrls {
	img := &model.ApartmentImage{
		ApartmentID: apartment.ID,
		Sequence:    uint8(k + 1),
		ImageUrl:    v,
	}
	if err := service.CreateObject(img, tmpDB); err != nil {
		tmpDB.Rollback()
		util.ResponseDbError(http.StatusOK, c, err)
		return
	}
}

tmpDB.Commit()
```

## 软删除
``` go
type User struct {
    gorm.Model
    Birthday     time.Time
}

// gorm源码
type Model struct {
	ID        uint `gorm:"primary_key"`
	CreatedAt time.Time
	UpdatedAt time.Time
	DeletedAt *time.Time `sql:"index"`
}
```

grom确保自动更新CreatedAt/UpdatedAt，删除只会设置DeletedAt而不删除实际的数据

## 打印sql
打印gorm最终执行的sql对性能优化很有帮助
``` go
// LogMode set log mode, `true` for detailed logs,
//  `false` for no log, default, will only print error logs
func (s *DB) LogMode(enable bool) *DB {
	if enable {
		s.logMode = 2
	} else {
		s.logMode = 1
	}
	return s
}
```

## 支持自定义对象

对于自定义类型变量，只需要实现`database/sql.Scanner`接口，参见<https://github.com/jinzhu/gorm/issues/47>

``` go
type FmtDate struct {
	Tm *time.Time
}

const (
	ctLayout = "2006-01-02 15:04:05"
	cdLayout = "2006-01-02"
)

// sql.Scanner implementation to convert a time.Time column to a LocalDate
func (ct *FmtDate) Scan(value interface{}) error {
	if tm, ok := value.(time.Time); ok {
		ct.Tm = &tm
		return nil
	} else {
		return fmt.Errorf("invalid time.Time format")
	}
}

// sql/driver.Valuer implementation to go from LocalDate -> time.Time
func (ct FmtDate) Value() (driver.Value, error) {
	if ct.Tm != nil {
		return ct.Tm.Format(cdLayout), nil
	} else {
		return "", nil
	}
}

// 为了支持json(序列化、反序列化)，可以实现
func (ct *FmtDate) UnmarshalJSON(b []byte) (err error) {
	s := strings.Trim(string(b), "\"")
	if s == "null" {
		return fmt.Errorf("invalid date format")
	}
	tm, err := time.Parse(cdLayout, s)
	if err == nil {
		ct.Tm = &tm
	}
	return err
}

func (ct FmtDate) MarshalJSON() ([]byte, error) {
	if ct.Tm != nil {
		return []byte(fmt.Sprintf("\"%s\"", ct.Tm.Format(cdLayout))), nil
	} else {
		return []byte("\"\""), nil
	}
}
```

## 自动解析时间
<https://stackoverflow.com/questions/29341590/go-parse-time-from-database>

<https://github.com/go-sql-driver/mysql#timetime-support>

The default internal output type of MySQL `DATE` and `DATETIME` values is `[]byte` which allows you to scan the value into a `[]byte`, `string` or `sql.RawBytes` variable in your program.

However, many want to scan MySQL `DATE` and `DATETIME` values into `time.Time` variables, which is the logical opposite in Go to `DATE` and `DATETIME` in MySQL. You can do that by changing the internal output type from `[]byte` to `time.Time` with the DSN parameter `parseTime=true`. You can set the default [`time.Time` location](https://golang.org/pkg/time/#Location) with the `loc` DSN parameter.

**Caution:** As of Go 1.1, this makes `time.Time` the only variable type you can scan `DATE` and `DATETIME` values into. This breaks for example [`sql.RawBytes` support](https://github.com/go-sql-driver/mysql/wiki/Examples#rawbytes).

Alternatively you can use the [`NullTime`](https://godoc.org/github.com/go-sql-driver/mysql#NullTime) type as the scan destination, which works with both `time.Time` and `string` / `[]byte`.

## 时区
gorm（应该是go-sql-driver/mysql）会使用utc时间，结果就是明明用9点写入数据库，但数据库里面存的是1点（东八区）

解决办法有几种

设置driver的loc为local
``` go
db, err := gorm.Open("mysql", 
	"db:dbadmin@tcp(127.0.0.1:3306)/foo?charset=utf8&parseTime=true&loc=Local")
```

这样方案的好处是省事，不过在数据库中存当地时间，某些情况下（比如跨国家）会造成疑惑和bug，另一种变通的办法时，使用时间转成本地时间

``` go
type FmtTime struct {
	Tm *time.Time
}

const (
	ctLayout = "2006-01-02 15:04:05"
	cdLayout = "2006-01-02"
)

func (ct *FmtTime) UnmarshalJSON(b []byte) (err error) {
	s := strings.Trim(string(b), "\"")
	if s == "null" {
		return fmt.Errorf("invalid date format")
	}
	tm, err := time.Parse(ctLayout, s)
	if err == nil {
		ct.Tm = &tm
	}
	return err
}

func (ct *FmtTime) MarshalJSON() ([]byte, error) {
	if ct.Tm != nil {
		// 用本地时区输出
		return []byte(fmt.Sprintf("\"%s\"", ct.Tm.In(time.Local).Format(ctLayout))), nil
	} else {
		return []byte("\"\""), nil
	}
}
```

## 初始化
``` go
func getMysqlUrl(c *config.Config) string {
	return fmt.Sprintf("%s:%s@tcp(%s:%d)/%s?charset=utf8&parseTime=true",
		c.MysqlUser,
		c.MysqlPassword,
		c.MysqlHost,
		c.MysqlPort,
		c.MysqlDb)
}

func OpenDB(c *config.Config) (db *sql.DB, driver string) {
	driver = "mysql"
	var url string = getMysqlUrl(c)
	db, err := sql.Open(driver, url)
	if err != nil {
		panic(err)
	}
	if driver == "mysql" {
		// per issue https://github.com/go-sql-driver/mysql/issues/257
		db.SetMaxIdleConns(0)
	}

	if err := pingDatabase(db); err != nil {
		log.Fatal(err)
		log.Fatalln("ping 数据库" + driver + "失败")
	}
	return
}

func OpenGorm(c *config.Config, driver string) (db *gorm.DB) {
	url := getMysqlUrl(c)
	db, err := gorm.Open(driver, url)
	if err != nil {
		log.Fatal(err)
	}
	return
}

func pingDatabase(db *sql.DB) (err error) {
	for i := 0; i < 10; i++ {
		err = db.Ping()
		if err == nil {
			return
		}
		log.Print("ping 数据库失败, 1s后重试")
		time.Sleep(time.Second)
	}
	return
}
```
