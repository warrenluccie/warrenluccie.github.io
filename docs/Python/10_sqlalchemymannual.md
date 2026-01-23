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

### 增加一条或者多条记录

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
def addManyUser(users=None):
    if users is None:
        users = []
    
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


# :增加不同对象数据
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

```python
#解决输出中文乱码问题
import sys
reload(sys)
sys.setdefaultencoding('utf-8')
```





### 查询表中所有数据（全表查）

```python
#1.查询表中所有数据（全表查）
def query_all(cls):
    #建立一个连接池
    Session = sessionmaker(bind=engine)
    #从连接池中获取一个连接
    session = Session()
    #查询表中所有数据
    objs = session.query(cls).all()
    #将连接放回连接池中
    session.close()
    #方法返回
    return objs

# print query_all(User)


#2.排序查询
def orderbyCls(cls,column):
    from sqlalchemy.sql.expression import text
    db_session = sessionmaker(bind=engine)
    session = db_session()
    objs = session.query(cls).order_by(text(column)).all()
    #objs = session.query(cls).order_by(cls.id.desc()).all()
    session.close()
    return objs

# print orderbyCls(User,'id')


#3.获取表中记录数
def count(cls):
    db_session = sessionmaker(bind=engine)
    session = db_session()
    c = session.query(cls).count()
    session.close()
    return c

# print count(User)


#4.分页
def page(cls,num,size=2):
    db_session = sessionmaker(bind=engine)
    session = db_session()
    datas = session.query(cls).offset((num-1)*size).limit(size).all()
    session.close()
    return datas
# print page(User,2)


#5.通过主键查询记录
def getClsByPk(cls,pk):
    db_session = sessionmaker(bind=engine)
    session = db_session()
    data = session.query(cls).get(pk)
    session.close()
    return data
# print getClsByPk(User,3)


#6.将公共部分提取成装饰器
def wrapper_session(func):
    def _wrapper(*args,**kwargs):
        from sqlalchemy.orm.session import sessionmaker
        conn_pool = sessionmaker(bind=engine)
        conn = conn_pool()
        data = func(conn,*args,**kwargs)
        conn.close()
        return data
    return _wrapper

#通过某个字段删除一条记录
@wrapper_session
def deleteByCoumn(session,cls,id):
    session.query(cls).filter(cls.id==id).delete()
    session.commit()

# deleteByCoumn(cls=User,id=6)

#通过对象来更新属性
@wrapper_session
def updateUserByAttr(session,obj):
    session.add(obj)
    session.commit()

# u = getClsByPk(User,3)
# u.pwd = '111'
# updateUserByAttr(obj=u)


#filter(单条件查询)
@wrapper_session
def filter1(session,account):
    u = session.query(User).filter(User.account==account).all()
    return u

# print filter1('zhangsan')


#filter(多条件查询)(与的关系)
@wrapper_session
def filter2(session,account,pwd):
    from sqlalchemy import and_
    u = session.query(User).filter(and_(User.account == account, User.pwd == pwd)).all()
    # u = session.query(User).filter((User.account == account, User.pwd == pwd).all()
    return u

# print filter2('zhangsan','123')


#filter(多条件查询)(或的关系)
@wrapper_session
def filter3(session,account,pwd):
    from sqlalchemy import or_
    u = session.query(User).filter(or_(User.account==account,User.pwd==pwd)).all()
    return u

# print filter3('zhangsan','111')



#filter(多条件查询)(非的关系)
@wrapper_session
def filter4(session,account):
    from sqlalchemy import not_
    u = session.query(User).filter(not_(User.account==account)).all()
    return u

# print filter4('zhangsan')

#filter(多条件查询)(嵌套使用)
@wrapper_session
def filter5(session,account,pwd):
    from sqlalchemy import not_,or_
    u = session.query(User).filter(not_(or_(User.account==account,User.pwd==pwd))).all()
    return u

# print filter5('zhangsan','111')



#分组查询
@wrapper_session
def group_by_query(session):
    from sqlalchemy.sql.functions import func
    datas = session.query(func.count(User.id),User.pwd).group_by(User.pwd).all()
    return datas

# print group_by_query()


#查看部分字段的值
@wrapper_session
def query_part(session):
    # datas = session.query(User.id,User.account).all()
    datas = session.query(User.id.label(u'编号'),User.account.label(u'用户名')).all()
    return datas

# print query_part()
```





### 多表操作(创建多表)

```python
#coding=utf-8

#多表操作（创建多表）
from sqlalchemy.engine import create_engine

#配置引擎
conn_url = 'mysql://root:123456@127.0.0.1:3306/tornado?charset=utf8'
engine = create_engine(conn_url,encoding='utf-8',echo=True)


#创建基表
from sqlalchemy.ext.declarative import declarative_base
Base = declarative_base(bind=engine)

#导入列和数据类型
from sqlalchemy import Column,ForeignKey
from sqlalchemy.types import Integer,Float,String,Text,Date,DateTime


#建表
#班级表 t_cls
#学生表 t_stu
#课程表 t_course

class Clazz(Base):
    __tablename__='t_cls'
    cno = Column(Integer,primary_key=True,autoincrement=True)
    cname = Column(String(20),unique=True,nullable=False)
    def __repr__(self):
        return u'<Clazz:%s,%s>'%(self.cno,self.cname)

import datetime
class Student(Base):
    __tablename__='t_stu'
    sno = Column(Integer,primary_key=True,autoincrement=True)
    sname = Column(String(20),unique=True,nullable=False)
    score = Column(Float(decimal_return_scale=2),default=10.00)
    birth = Column(Date,default=datetime.date.today())
    desc = Column(Text,nullable=True)
    cno = Column(Integer,ForeignKey(Clazz.cno,ondelete='CASCADE'))

    def __repr__(self):
        return u'<Student:%s,%s,%s>'%(self.sno,self.sname,self.cno)
    
class Course(Base):
    __tablename__='t_course'
    courseid = Column(Integer,primary_key=True,autoincrement=True)
    coursename = Column(String(20),nullable=False)
    def __repr__(self):
        return u'<Course:%s,%s>'%(self.courseid,self.coursename)


class Stu_Course(Base):
    __tablename__='t_sc'
    id = Column(Integer,primary_key=True,autoincrement=True)
    sno = Column(Integer,ForeignKey(Student.sno,ondelete='CASCADE'),nullable=False)
    courseid = Column(Integer,ForeignKey(Course.courseid,ondelete='CASCADE'),nullable=False)

    def __repr(self):
        return u'<Stu_Course:%s,%s>'%(self.sno,self.courseid)

#建表语句
Base.metadata.create_all()
```





### 插入操作(多表插入操作)

```python

#插入操作(多表插入操作)
def insert(cname,sname,score,content,coursenames=[]):
    from sqlalchemy.orm.session import sessionmaker
    db_session = sessionmaker(bind=engine)
    session = db_session()


    try:
        #创建班级
        session.begin_nested()  # 保存断点
        if session.query(Clazz.cno).filter(Clazz.cname==cname).count()==1:
            cls =session.query(Clazz.cno).filter(Clazz.cname==cname).first()
        else:
            cls = Clazz(cname=cname)
            session.add(cls)
            session.commit()  # 提交事务
            session.flush([cls,])#将改变提交到数据库
            print cls.cno

        #创建学生
        import datetime
        stu = Student(sname=sname,score=score,birth=datetime.datetime.strptime('1983-01-02','%Y-%m-%d'),desc=content,cno=cls.cno)
        session.add(stu)
        session.commit()
        session.flush([stu,])


        #创建课程
        cs = []
        print coursenames
        for courname in coursenames:
            if session.query(Course).filter(Course.coursename==courname).count()==1:

                course = session.query(Course).filter(Course.coursename==courname).first()


            else:
                course = Course(coursename=courname)
                session.add(course)
                session.commit()
                session.flush([course,])

            print course
            cs.append(course)

        #插入课程学生中间表内容
        for c in cs:
            sc = Stu_Course(sno=stu.sno,courseid=c.courseid)
            session.add(sc)
            session.commit()
            session.flush([sc,])

    except Exception as e:
        print e
        session.rollback()

    session.close()


# insert(cname,sname,score,content,coursenames=[])
# insert('B209Python班','wangwu',100,'Django学习',coursenames=['Python','D'])



#多表查询
def query():
    from sqlalchemy.orm.session import sessionmaker
    db_sesion = sessionmaker(bind=engine)
    session = db_sesion()

    #查询同班同学学生信息
    #笛卡尔积（交叉连接）
    # datas = session.query(Clazz,Student).all()

    #等值连接
    # datas = session.query(Clazz,Student).filter(Clazz.cno==Student.cno).all()

    #非等值连接
    # datas = session.query(Clazz,Student).filter(Clazz.cno>8,Clazz.cno==Student.cno).all()

    #内连接
    # datas = session.query(Clazz,Student).join(Student,Clazz.cno==Student.cno).all()

    #外连接（只有左连接）
    # datas = session.query(Clazz,Student).outerjoin(Student,Clazz.cno==Student.cno).all()

    #原生查询
    sql ='select * from t_cls'
    datas =session.execute(sql)

    session.close()
    return datas

# print query()
print query().fetchall()



```



### 配置连接池

```python
#配置连接池
from sqlalchemy.engine import create_engine
#配置引擎
conn_url = 'mysql://root:123456@127.0.0.1:3306/tornado?charset=utf8'
engine = create_engine(conn_url,encoding='utf-8',echo=False,pool_size=10,max_overflow=20,pool_recycle=3600,pool_timeout=3600)
#pool_size=10  核心连接数为10
#max_overflow=20 非核心连接数为20(超过核心连接数后最多创建20个连接)
#pool_recycle=3600 等待3600秒（1小时）后回收非核心连接数
#pool_timeout=3600 所有连接数都被占用时，剩余请求连接将等待3600秒获取连接接，如果获取不到，将连接失败~
```



















