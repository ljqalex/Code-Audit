## 浅谈sql注入

**sql注入的危害：**

从1998乃至几年前，sql注入都一直名列网站安全、web安全的头把交椅，主要就是因为其攻击简单、攻击手法多样、危害巨大（拿数据库、什么拿shell、拿系统权限)。
近几年，因为规范化的框架使用、orm的普及，基本在框架层面就解决了大部分普通sql注入的问题, sql注入越来越难，所以表现就是sql注入不那么受关注了。但需要注意的是，sql注入仍然是危害巨大的攻击手法。

**sql注入的基础代码示例**：

**php：**

```php
<?php
// 假设用户通过表单提交了一个名为username的参数
$username = $_POST['username'];

// 构造SQL查询语句
$sql = "SELECT * FROM users WHERE username = '$username'";// input: qiangqiang' and 1=1 #

// 执行查询
$result = mysqli_query($conn, $sql);

// 处理查询结果
while ($row = mysqli_fetch_assoc($result)) {
    echo "Username: " . $row['username'] . "<br>";
}

// 关闭数据库连接
mysqli_close($conn);
?>
```

**java：**

```java
import java.sql.*;
public class SQLInjectionExample {
    public static void main(String[] args) {
        try {
            // 建立数据库连接
            String url = "jdbc:mysql://localhost:3306/mydatabase";
            String username = "root";
            String password = "password";
            Connection con = DriverManager.getConnection(url, username, password);

            // 用户输入
            String userInput = "admin' OR '1'='1";

            // 使用Statement执行SQL查询
            Statement stmt = con.createStatement();
            String sql = "SELECT * FROM users WHERE username = '" + userInput + "'";
            ResultSet rs = stmt.executeQuery(sql);

            // 输出查询结果
            while (rs.next()) {
                String username = rs.getString("username");
                String password = rs.getString("password");
                System.out.println("Username: " + username + ", Password: " + password);
            }

            // 关闭连接
            con.close();
        } catch (SQLException e) {
            System.err.println("SQL Exception: " + e.getMessage());
        }
    }
}
```
**python：**

```python
import sqlite3

conn = sqlite3.connect('example.db')
cursor = conn.cursor()

cursor.execute('''CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, password TEXT)''')

cursor.execute("INSERT INTO users (username, password) VALUES ('admin', 'password')")
cursor.execute("INSERT INTO users (username, password) VALUES ('alice', '123456')")

user_input = "admin' OR '1'='1"
sql = "SELECT * FROM users WHERE username = '{}'".format(user_input)

cursor.execute(sql)

for row in cursor.fetchall():
    print(row)

conn.close()
```

以上全都是不同语言在不调用orm框架，直接调用原生数据库操作函数时的用例。(其实orm框架的底层也是调用了原生数据库操作函数，只是orm帮开发者做了封装和对象映射的步骤)

如何理解这个漏洞:

```
1.仅仅抓住输入
2.当数据流入侵到控制流时，漏洞就产生了
3.“数据流入侵控制流”产生的风险点，在于不同层面组件的交汇处（如:代码层与数据库层)
```

**调用orm的错误写法**
***php:***
php没有啥比较出名的orm框架，基本是各web框架或者cms自行去实现自己的orm (利用点,一套闭源cms的orm可能实现得并不规范，产生漏洞）

***java/mybatis：***

这是mybatis的详解：https://blog.csdn.net/qq_21561501/article/details/123649171（博主写的很完善）

mybatis需要有个xml配置文件来绑定映射关系

<select id="findUserByName" parameterType="java.lang.String"resultType="cn.itcast.mybatis.po.User">
<!--拼接MySQL,引起SQL注入-->
		SELECT *FROM table WHERE username = '${value}'
</select>

```java
@Test
public void testFindUserByName( )throws Exception{
	SqlSession sqlSession=sqlSessionFactory. openSession();//创建UserMapper代理对象
	UserMapper userMapper=sqlSession.getMapper(UserMapper.class);//调用userMapper的方法
	List<User> list=userMapper.findUserByName("adminqq' and 1=1#");
	sqlSession.close(;
	System.out.println(list);
	}
}
```

简单讲就是MyBatis有两种变量绑定方式，分别是:#{}和${}

- [ ] ```
  1 #是绑定变量的形式，底层会用#{}会被替换为?号，有参数映射,会在 DefaultParameterHandler 中进行设置占位符的操作-->预编译
  2 $也是绑定变量的形式，{value}是直接被替换为了对应的值，没有参数映射，不会进行设置占位符的操作-->拼接
  ```

***python:***

sqlalchemy是flask最经常配套的orm框架，在许多django项目中也常常看到身影。也是目前python上用得最火的orm框架。

这里小编给大家推荐1篇文章：

https://blog.csdn.net/weixin_46549605/article/details/123458430

来看下这两段代码：

```python
@app.route("/", methods=["GET"])
def test():
    username = request.args.get( 'username ')
    res = db.session. query(table).filter("username={}".format(username))
```

原因跟上面MyBatis类似，依然是用户输入其实是拼接后才导入sqlalchemy层的

正确的写法：

```python
@app.route("/", methods=["GET"])
def test():
    username = request.args.get( 'username ')
    res = db.session. query(table).filter(table.username=username)
```

**sql注入类型：**

- ```
  - sql联合查询注入
  - sql堆叠注入
  - sql报错注入-N个payload
  - sql时间盲注
  - sql布尔盲注
  - sql带外数据
  - sql注入执行命令--（os-shell）
  ```

  核心思维::执行sql语句，关键是数据库类型

  

所以最终都可以归纳成一个模型：

```
action(动作): select

object(对象) : table

subject(目标客体)︰*

condition(条件)∶

    key: username
    value: $username //用户输入
```

如果能进行堆叠注入我们就能跳出action动作，执行一条新的sql语句。

**爆数据的速度：**

报错 ~= 联合查询 > 带外数据 > 布尔盲注 > 时间盲注

我带大家深入解析下**宽字节注入**：

```php
<?php
$db = init_db();
$db->query("set SET NAMES 'gbk');//设置gbk字符集
$username = addslashes($_GET[ 'username ']);//input: adminqq' and 1=1#
$db->query("select * from table where username = '$username'");
?>
```

我们来还原下在数据里面的操作：

```sql
select * from table where username = 'adminqq/' and 1=1#'
```

在这种场景下，我们没办法进行sql注入，因为我们的单引号被转义了，我们没办法侵入到控制流去。

输入  ***adminqq%df' and 1=1#***
数据在存储的时候一定是以字节存储的，但是数据解读的时候，都是以字符的标准去解读的。所以，在连接数据库进行sql执行（执行就是一种对存储的解读)时，会按照字符集编码的规范去解读，所以遇到了\xDF\x5C的时候，会解读成一个字符)
此时我们来关注下**输入流转**

```
1 url输入-->php字符串变量-->addslashes-->sql语句-->数据库
2 adminqq%DF' and 1=1# --> adminqq\xDF' and 1=1# -->adminqq\xDF\' and 1=1#-->
3 select * from table where username = 'adminqq运' and 1=1#’//数据流成功入
侵到控制流
```

**无法预编译的输入点：**

**like关键字**

我们知道sql语句的模糊查找里面用的关键字like，而like关键字默认是不会预编译的(如果使用Mybatis则是预编译报错)。数据库方给出的原因好像是like预编译会造成慢查询和DOS

只能手动添加预编译：

```php
<?php
$username = $_GET['username'l;
$db = "mysql:host=127.0.0.1; dbname=test; ";
$dbname = "root";
$passwd = "root";
$conn = new PDO( $dbs，$dbname，$passwd);
$conn->query( 'SET NAMES GBK');
$stmt = $conn->prepare("select * from table where username like'%:username%'");//不生效
$stmt = $conn->prepare( "select * from table where username likeconcat('%', : username, '%' ");//生效
$stmt->bindParam(":username",$username);
$stmt->execute();
?>
```

可能很多开发会遗漏这个点，导致存在注入。
或者一些java的开发，Mybatis编译报错，然后他们自己去添加过滤(过滤没写好）或者不过滤（使用原生语句)，导致GG。
与之类似的还有IN关键字，该位置也默认不能预编译，需要在预编译语法中去for循环，有些程序员为了方便，可能也会在这边偷工减料。

**不能加引号的关键字：**

```php
$newInput = addslashes($input)//内容转义
sql = select * from table where column = '$newInput’//强制用单引号包裹
```

我们来看这个sql语句：

```sql
select $username$,password from $table$ where $username2$ = '$adminqq$'order by $username3$ $desc$ limit $O$, 1;
```

那是否有输入点是不可以加上单引号的呐？

所以我们就找到了：$username$，$table$，$username2$，$desc$，$0$

归纳就是 **表名，列名，limit子句，order by【desc/asc】**

这里还有一些介绍：https://forum.butian.net/share/1559