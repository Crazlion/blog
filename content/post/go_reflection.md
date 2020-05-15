+++
title='go的反射'
tags=["go"]
categories=["go"]
date="2020-04-27T11:10:52+08:00"
+++

最近在学习go的反射，做一下笔记  
## 基本概念 ##

```go
func reflectionDemo(a interface{})  {
    // 获取类型
    t0 := reflect.TypeOf(a)
    fmt.Printf("type of a is:%v\n", t0)
    // 获取值
    v := reflect.ValueOf(a)
    // 通过值获取类型
    t1 := v.Type()
    fmt.Printf("type of a is:%v\n", t1)
    // 通过类型获得分类
    kind := t1.Kind()
    // 通过分类来判断是什么类型的
    switch kind {
    // 结构体
    case reflect.Struct:
        for i := 0; i < v.NumField(); i++ {
            field := v.Field(i)
            //打印字段的名称、类型以及值
            fmt.Printf("name:%s type:%v value:%v\n",
            t1.Field(i).Name, field.Type().Kind(), field.Interface())
        }
    // int类型
    case reflect.Int:
        fmt.Printf("%d\n",v)
    // string类型
    case reflect.String:
        fmt.Printf("%s\n",v)
    // 其他补充
    default:
        fmt.Printf("other\n")
    }
}
```

## 实践 ##
因为之前都是写java，访问数据库select的话框架会直接反解成java对象  
写一个简单的查询数据库反解成对象  

数据库建表语句  

```sql
CREATE TABLE `mysql`.`student`  (
  `id` int(10) NOT NULL auto_increment,
  `name` varchar(255) NULL DEFAULT NULL,
  `age` int(10) NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB character set= utf8mb4 COMMENT='学生表'
```

go代码

```go
// Student对象 通过tag为数据库的列名，必须严格对应
type Student struct {
    Id   int    `db:"id"`
    Name string `db:"name"`
    Age  int    `db:"age"`
}

const (
    DBHostsIp  = "127.0.0.1:3306"
    DBUserName = "root"
    DBPassWord = "123456"
    DBName     = "mysql"
)

func main() {
    db, err := sql.Open("mysql", DBUserName+":" +DBPassWord+"@tcp("+DBHostsIp+")/"+DBName) // 设置连接数据库的参数
    CheckErr(err)
    defer db.Close() //关闭数据库
    query(db, &Student{})
}

//查询demo
func query(db *sql.DB, target interface{}) []interface{} {
    // 获取值
    v := reflect.ValueOf(target).Elem()
    // 获取类型
    t := v.Type()
    // rows：返回查询操作的结果集
    rows, err := db.Query("SELECT * FROM student")
    CheckErr(err)
    cols, _ := rows.Columns()

    // 用一个列名：索引的map保存信息
    colsMap := make(map[string]int)
    for i := 0; i < len(cols); i++ {
        colsMap[cols[i]] = i
    }

    // 这里表示一行所有列的值，用[]byte表示
    vals := make([][]byte, len(cols))
    // 这里表示一行填充数据
    line := make([]interface{}, len(cols))
    // 这里scans引用vals，把数据填充到[]byte里
    for k, _ := range vals {
        line[k] = &vals[k]
    }

    i := 0
    var resList []interface{}
    for rows.Next() {
        // 调用反射创建新对象
        newStruc := reflect.New(t)
        // 填充数据
        rows.Scan(line...)
        // 把vals中的数据复制到row中
        for index, v := range line {
            t := reflect.TypeOf(v)
            fmt.Printf("type of a is:%v\n", t)
            fmt.Println("---", cols[index])
        }

        // 默认接收的都是byte[],通过类的信息,将byte转换成对应的类型
        numField := v.NumField()
        for i := 0; i < numField; i++ {
            // 通过类型获取字段
            field := t.Field(i)
            // 字段的名字
            name := field.Name
            // 通过字段获取db的Tag
            colName := field.Tag.Get("db")
            // 获取新对象中对应的字段
            turlyField := newStruc.Elem().FieldByName(name)
            if index, ok := colsMap[colName]; ok {
                fmt.Println("print", colName)
                switch field.Type.Kind() {
                case reflect.String:
                    turlyField.SetString(string(vals[index]))
                case reflect.Int:
                    value, err := strconv.Atoi(string(vals[index]))
                    if err == nil {
                        turlyField.SetInt(int64(value))
                    }
                default:
                    fmt.Println("other")
                }
            }
        }
        resList = append(resList, newStruc)
        i++
    }

    /**
    * 结果如下
    * &{1 小明 20}
    * &{2 小红 18}
    * &{3 大明 50}
    */
    for _,value := range resList {
        fmt.Println(value)
    }
    return resList
}

```

