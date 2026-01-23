# SQLAlchemy库速查表及最佳实践



## 1.SQLAlchemy库是什么?

-  The Python SQL Toolkit and Object Relational Mapper
-  Python SQL 工具包和对象关系映射器
-  SQLAlchemy is the Python SQL toolkit and Object Relational Mapper that gives application developers the full power and flexibility of SQL.
-  SQLAlchemy 是 Python 的 SQL 工具包和对象关系映射器，它为应用程序开发人员提供了 SQL 的全部强大功能和灵活性。
-  It provides a full suite of well known enterprise-level persistence patterns, designed for efficient and high-performing database access, adapted into a simple and Pythonic domain language.
- 它提供了一整套广为人知的企业级持久化模式，专为高效且高性能的数据库访问而设计，并被改编成一种简单且符合 Python 风格的领域语言。

SQLAlchemy SQL工具包和对象关系映射器是一组用于处理数据库和Python的全面工具。

它具有几个不同的功能区域，可以单独使用或组合在一起使用。其主要组件如下图所示，组件依赖性分为几个层次：

![sqla_arch_small](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/sqla_arch_small.png)



- 一句话总结：把我们平时操作数据库的CRUD(SQL)转换成标准Python语言范式来达到对数据库的增删改查的目的。
- **以上，SQLAlchemy的两个最重要的前端部分是对象关系映射器和 SQL表达式语言。SQL表达式可以独立于ORM使用。在使用ORM时，SQL表达式语言仍然是面向公众的API的一部分，因为它在对象关系配置和查询中使用。**





## 2.使用SQLAlchemy来创建数据库表



### 2.1 安装sqlalchemy模块

```bash
pip install sqlalchemy
```



### 2.2 配置引擎

```python
from sqlalchemy.engine import create_engine
conn_url = 'mysql://root:123456@127.0.0.1:3306/testlogin?charset=utf8'
engine = create_engine(conn_url,encoding='utf-8',echo=True)
```



### 2.3 声明ORM基类

- **这个基类的子类会自动和数据库表进行关联**

```python
from sqlalchemy.ext.declarative import declarative_base
Base = declarative_base(bind=engine)
```



### 2.4 导入列字段类和数据类型类

- Column类和导入的数据类型类目的是为了与表中字段名和相应的数据类型相关联

```python
from sqlalchemy import Column
from sqlalchemy.types import Integer,String,Date,DateTime,Float,Text
```



### 2.5 创建ORM类

- ORM类将数据库中表以及表中相关字段名和相应的字段数据类型全部转换为Python中的类对象以及类中的attributes属性
- 建议：一张表对应一个Python class object.

```python
class User(Base):
    __tablename__='t_cuser'  # MySQL数据库表名：假设表名为t_user

    id = Column(Integer,primary_key=True,autoincrement=True)  # t_user表中id字段，数据类型为：int,主键,自增长
    account = Column(String(length=8),unique=True) # t_user表中account字段，数据类型为：varchar(8)，唯一字段
    pwd = Column(String(length=3))  # t_user表中的pwd字段，数据类型为varchar(3)
    birth = Column(Date)  # t_user表中的birth字段，数据类型为：date (日期类型)
    score = Column(Float(decimal_return_scale=2))  # t_user表中的分数字段，数据类型为：float(浮点型)

    def __repr__(self):
        """字符串表示"""
        return '[User:%s,%s]'%(self.id,self.account)

class Address(Base):
    __tablename__='t_addr'   # MySQL数据库中表：表名为t_addr

    id = Column(Integer,primary_key=True,autoincrement=True) # id字段,PK,autoincrement自增长字段
    aname = Column(String(30),unique=True) # t_addr表中aname字段，数据类型为：varchar(30)
```



### 2.6 利用基类创建数据库表

- 也可以利用SQLAlchemy中基类来创建数据库表

```python
    #如果表已经存在，则不执行当前存在表的创建操作
    Base.metadata.create_all()
    #7.利用基类删除所有的数据库表
    Base.metadata.drop_all()
```





## 3.利用SQLAlchemy对数据库中的单表进行CRUD操作

- 利用sqlalchemy库的内置方法对一张表进行增删改查操作
-  MySQL数据库表：t_user表，一共有五个字段：分别是id,account,pwd,birth,score
- id字段是PK且自增长字段，新增加记录时该字段可以不填充值

```python
# 数据表中的单表CRUD操作
from sqlalchemy.orm import sessionmaker

# 定义一个函数方法addUser用来向数据库中增加一条记录
def addUser(account,pwd):
    # 建立数据库连接
    db_session = sessionmaker(bind=engine)  # 相当于创建数据库连接池（默认有5个连接）
    session = db_session()  # 获取连接池中的一个连接

    user = User(account=account,pwd=pwd)
    session.add(user)
    session.commit()
    session.refresh(user)
    session.close()#将连接放回连接池中
    return user

# addUser('zhangsan',123)

 # 定义一个函数方法addManyUser向数据库中增加多条记录
def addManyUser(users=[]):
    # 2.建立数据库连接
    db_session = sessionmaker(bind=engine)  # 相当于创建数据库连接池（默认有5个连接）
    session = db_session()  # 获取连接池中的一个连接

    us =[]
    import datetime
    for ac,pwd,sc in users:
        us.append(User(account=ac,pwd=pwd,score=sc,birth=datetime.datetime.today()))
    session.add_all(us)
    session.commit()#提交事务
    [session.refresh(u) for u in us]#刷新属性值
    session.close()
    return us

# addManyUser(users=[('xiaoming','123',88.9),('xiaohong','123',100)])


  #8.3:增加不同对象数据
def addDiffObj(*args):
    db_session = sessionmaker(bind=engine)
    session = db_session()

    session.add_all(args)
    session.commit()
    [session.refresh(a) for a in args]
    session.close()
    return args

# addDiffObj(User(account='李四',pwd='123',score=66.6),Address(aname='上海市'))


```



















































