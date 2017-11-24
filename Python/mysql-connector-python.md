# Python连接MySQL数据库（mysql-connector-python）

本文介绍的库是mysql-connector-python，它是MySQL的Python驱动，使用它你可以连接MySQL数据库，访问、操作表和数据。MySQL的Python驱动也不少，但是mysql-connector-python是最流行、最稳定的。

很久以前，我做过使用MySQL的C API操作数据库，相比之下，Python更加简洁、方便。

#### 1 安装 MySQL connector

mysql-connector-python支持Linux、Mac OS X和Windows系统。

mysql-connector-python的下载地址http://dev.mysql.com/downloads/connector/python/。

我以Ubuntu系统Python2.7为例：

**方法一：**使用apt安装：

$ sudo apt install python-mysql.connector

**方法二：**使用pip安装：

 $ sudo pip install mysql-connector-python 

**方法三**：安装最新版

```shell
$ wget http://cdn.mysql.com//Downloads/Connector-Python/mysql-connector-python-cext_2.1.3-1ubuntu15.04_amd64.deb$ sudo dpkg -i mysql-connector-python-cext_2.1.3-1ubuntu15.04_amd64.deb 
```

查看安装的mysql-connector-python版本：

![Python连接MySQL数据库（mysql-connector-python）](http://blog.topspeedsnail.com/wp-content/uploads/2016/06/Screen-Shot-2016-06-15-at-19.38.18.png)

#### 2 连接MySQL

连接本地MySQL数据库：

```python
import mysql.connector
 
conn = mysql.connector.connect(
         user='root',
         password='test1234',
         host='127.0.0.1',
         database='test')
 
# 关闭数据库
conn.close()
```

如果提供的用户、数据库不对，会输出如下错误信息：

```python
Traceback (most recent call last):
  File "test.py", line 7, in <module>
    database='test')
  File "/usr/lib/python2.7/dist-packages/mysql/connector/__init__.py", line 162, in connect
    return MySQLConnection(*args, **kwargs)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 129, in __init__
    self.connect(**kwargs)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 454, in connect
    self._open_connection()
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 421, in _open_connection
    self._ssl)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 204, in _do_auth
    self._auth_switch_request(username, password)
  File "/usr/lib/python2.7/dist-packages/mysql/connector/connection.py", line 240, in _auth_switch_request
    raise errors.get_exception(packet)
mysql.connector.errors.ProgrammingError: 1045 (28000): Access denied for user 'root'@'localhost' (using password: YES)

```

#### 3 执行SQL语句

```python
import mysql.connector
 
conn = mysql.connector.connect(
         user='root',
         password='test1234',
         host='127.0.0.1',
         database='test')
 
cur = conn.cursor()
 
# 要执行的SQL语句
query = ("SELECT * FROM sometable")
 
# 执行查询
cur.execute(query)
 
for (id, name, class, score) in cur:
    print("{}, {}, {}, {}".format(id, name,class,score))
 
cur.close()
conn.close()
```

