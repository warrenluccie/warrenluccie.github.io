# Python 面向对象编程 (OOP) 速查表及最佳实践

主要内容：
 1. 类与对象
 2. 继承与多态
 3. 封装与属性
 4. 特殊方法（魔术方法）
 5. 抽象基类
 6. 类与静态方法
 7. 属性装饰器
 8. 最佳实践



## 🚀 快速概览

面向对象编程是 Python 的核心编程范式，通过类和对象来组织代码，实现封装、继承和多态。掌握 OOP 是构建复杂、可维护应用程序的关键。

---

## 📝 类和对象基础

### 类定义和实例化
```python
# 基础类定义
class Dog:
    """狗类的简单示例"""
    
    # 类属性（所有实例共享）
    species = "Canis familiaris"
    
    def __init__(self, name, age):
        """初始化方法"""
        # 实例属性（每个实例独有）
        self.name = name
        self.age = age
    
    def bark(self):
        """实例方法"""
        return f"{self.name} says woof!"
    
    def get_human_age(self):
        """另一个实例方法"""
        return self.age * 7

# 创建实例
my_dog = Dog("Buddy", 3)
print(my_dog.name)           # Buddy
print(my_dog.bark())         # Buddy says woof!
print(my_dog.species)        # Canis familiaris
```

### 完整的类示例
```python
class BankAccount:
    """银行账户类"""
    
    # 类常量
    BANK_NAME = "Python Bank"
    INTEREST_RATE = 0.01
    
    def __init__(self, account_holder, initial_balance=0):
        self.account_holder = account_holder
        self._balance = initial_balance  # 保护属性
        self._transaction_history = []
    
    def deposit(self, amount):
        """存款"""
        if amount <= 0:
            raise ValueError("存款金额必须为正数")
        
        self._balance += amount
        self._transaction_history.append(f"存款: +${amount}")
        return self._balance
    
    def withdraw(self, amount):
        """取款"""
        if amount <= 0:
            raise ValueError("取款金额必须为正数")
        if amount > self._balance:
            raise ValueError("余额不足")
        
        self._balance -= amount
        self._transaction_history.append(f"取款: -${amount}")
        return self._balance
    
    def get_balance(self):
        """获取余额"""
        return self._balance
    
    def get_transaction_history(self):
        """获取交易历史"""
        return self._transaction_history.copy()  # 返回副本保护原始数据
    
    def apply_interest(self):
        """应用利息"""
        interest = self._balance * self.INTEREST_RATE
        self._balance += interest
        self._transaction_history.append(f"利息: +${interest:.2f}")
        return interest

# 使用
account = BankAccount("Alice", 1000)
account.deposit(500)
account.withdraw(200)
print(f"余额: ${account.get_balance()}")  # 余额: $1300
```

---

## 🎯 封装和属性控制

### 访问控制
```python
class SecureData:
    """展示封装和访问控制的类"""
    
    def __init__(self, public_data, protected_data, private_data):
        self.public_data = public_data           # 公共属性
        self._protected_data = protected_data    # 保护属性（约定）
        self.__private_data = private_data       # 私有属性（名称修饰）
    
    # 公共方法
    def get_private_data(self):
        """获取私有数据的公共接口"""
        return self.__private_data
    
    def set_private_data(self, value):
        """设置私有数据的公共接口"""
        if self._validate_data(value):
            self.__private_data = value
        else:
            raise ValueError("数据验证失败")
    
    # 保护方法
    def _validate_data(self, data):
        """内部验证方法"""
        return data is not None and len(str(data)) > 0
    
    # 私有方法
    def __internal_process(self):
        """内部处理方法"""
        return f"处理: {self.__private_data}"

# 使用
obj = SecureData("public", "protected", "private")

print(obj.public_data)           # 可以直接访问
print(obj._protected_data)       # 可以访问但不推荐
# print(obj.__private_data)      # 错误！AttributeError
print(obj._SecureData__private_data)  # 可以访问但不应该！

print(obj.get_private_data())    # 通过公共接口访问
```

### 属性装饰器
```python
class Person:
    """使用属性装饰器的类"""
    
    def __init__(self, name, age):
        self._name = name
        self._age = age
    
    @property
    def name(self):
        """name 属性的 getter"""
        print("获取 name")
        return self._name
    
    @name.setter
    def name(self, value):
        """name 属性的 setter"""
        print(f"设置 name: {value}")
        if not value or not value.strip():
            raise ValueError("姓名不能为空")
        self._name = value.strip()
    
    @property
    def age(self):
        """age 属性的 getter"""
        return self._age
    
    @age.setter
    def age(self, value):
        """age 属性的 setter"""
        if not isinstance(value, int) or value < 0 or value > 150:
            raise ValueError("年龄必须在 0-150 之间")
        self._age = value
    
    @property
    def is_adult(self):
        """只读计算属性"""
        return self._age >= 18
    
    @property
    def info(self):
        """只读组合属性"""
        return f"{self._name}, {self._age}岁, {'成人' if self.is_adult else '未成年'}"

# 使用
person = Person("Alice", 25)
print(person.name)        # 调用 getter
person.name = "Bob"       # 调用 setter
print(person.is_adult)    # True
print(person.info)        # Bob, 25岁, 成人

# person.is_adult = False  # 错误！只读属性
```

---

## 🔄 继承和多态

### 基础继承
```python
class Animal:
    """动物基类"""
    
    def __init__(self, name, age):
        self.name = name
        self.age = age
    
    def speak(self):
        """动物叫声（需要在子类中重写）"""
        raise NotImplementedError("子类必须实现 speak 方法")
    
    def move(self):
        """动物移动"""
        return f"{self.name} 在移动"
    
    def __str__(self):
        return f"{self.name} ({self.__class__.__name__}), {self.age}岁"

class Dog(Animal):
    """狗类，继承自动物"""
    
    def __init__(self, name, age, breed):
        super().__init__(name, age)  # 调用父类初始化
        self.breed = breed
    
    def speak(self):
        """重写父类方法"""
        return f"{self.name} says Woof!"
    
    def fetch(self):
        """狗特有的方法"""
        return f"{self.name} 正在接球"

class Cat(Animal):
    """猫类，继承自动物"""
    
    def __init__(self, name, age, lives=9):
        super().__init__(name, age)
        self.lives = lives
    
    def speak(self):
        """重写父类方法"""
        return f"{self.name} says Meow!"
    
    def climb(self):
        """猫特有的方法"""
        return f"{self.name} 在爬树"

# 使用多态
animals = [
    Dog("Buddy", 3, "Golden Retriever"),
    Cat("Whiskers", 2)
]

for animal in animals:
    print(animal)           # 调用 __str__
    print(animal.speak())   # 多态：调用各自的方法
    print(animal.move())    # 继承的方法
```

### 多重继承和方法解析顺序 (MRO)
```python
class Flyable:
    """可飞行混合类"""
    
    def __init__(self, max_altitude=1000):
        self.max_altitude = max_altitude
    
    def fly(self):
        return "飞行中..."
    
    def take_off(self):
        return "起飞"

class Swimmable:
    """可游泳混合类"""
    
    def swim(self):
        return "游泳中..."
    
    def dive(self):
        return "下潜"

class Duck(Animal, Flyable, Swimmable):
    """鸭子类，多重继承"""
    
    def __init__(self, name, age, max_altitude=500):
        Animal.__init__(self, name, age)
        Flyable.__init__(self, max_altitude)
        # Swimmable 没有 __init__，不需要调用
    
    def speak(self):
        return f"{self.name} says Quack!"
    
    def all_abilities(self):
        """使用所有混合类的方法"""
        return [
            self.move(),
            self.fly(),
            self.swim()
        ]

# 使用
duck = Duck("Donald", 2)
print(duck.speak())         # Donald says Quack!
print(duck.take_off())      # 起飞
print(duck.dive())          # 下潜

# 查看方法解析顺序
print(Duck.__mro__)
# (<class '__main__.Duck'>, <class '__main__.Animal'>, 
#  <class '__main__.Flyable'>, <class '__main__.Swimmable'>, <class 'object'>)
```

---

## 🎭 特殊方法（魔术方法）

### 常用魔术方法
```python
class Vector:
    """向量类，展示魔术方法的使用"""
    
    def __init__(self, x, y):
        self.x = x
        self.y = y
    
    # 表示方法
    def __str__(self):
        """str() 和 print() 时调用"""
        return f"Vector({self.x}, {self.y})"
    
    def __repr__(self):
        """repr() 和交互式环境显示时调用"""
        return f"Vector({self.x}, {self.y})"
    
    # 算术运算
    def __add__(self, other):
        """+ 运算符"""
        if isinstance(other, Vector):
            return Vector(self.x + other.x, self.y + other.y)
        return NotImplemented
    
    def __sub__(self, other):
        """- 运算符"""
        if isinstance(other, Vector):
            return Vector(self.x - other.x, self.y - other.y)
        return NotImplemented
    
    def __mul__(self, scalar):
        """* 运算符（向量数乘）"""
        if isinstance(scalar, (int, float)):
            return Vector(self.x * scalar, self.y * scalar)
        return NotImplemented
    
    def __rmul__(self, scalar):
        """反向 * 运算符"""
        return self.__mul__(scalar)
    
    # 比较运算
    def __eq__(self, other):
        """== 运算符"""
        if isinstance(other, Vector):
            return self.x == other.x and self.y == other.y
        return False
    
    def __lt__(self, other):
        """< 运算符（基于模长）"""
        if isinstance(other, Vector):
            return self.magnitude() < other.magnitude()
        return NotImplemented
    
    # 其他魔术方法
    def __len__(self):
        """len() 时调用"""
        return 2  # 二维向量
    
    def __getitem__(self, index):
        """[] 索引访问"""
        if index == 0:
            return self.x
        elif index == 1:
            return self.y
        else:
            raise IndexError("Vector 索引超出范围")
    
    def __contains__(self, value):
        """in 运算符"""
        return value == self.x or value == self.y
    
    # 普通方法
    def magnitude(self):
        """计算向量模长"""
        return (self.x ** 2 + self.y ** 2) ** 0.5
    
    def dot(self, other):
        """点积"""
        if isinstance(other, Vector):
            return self.x * other.x + self.y * other.y
        raise TypeError("参数必须是 Vector 类型")

# 使用
v1 = Vector(3, 4)
v2 = Vector(1, 2)

print(v1)                   # Vector(3, 4)
print(v1 + v2)             # Vector(4, 6)
print(v1 * 2)              # Vector(6, 8)
print(2 * v1)              # Vector(6, 8)
print(v1 == v2)            # False
print(v1[0])               # 3
print(3 in v1)             # True
print(v1.magnitude())      # 5.0
```

### 上下文管理器魔术方法
```python
class DatabaseConnection:
    """数据库连接上下文管理器"""
    
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.connection = None
    
    def __enter__(self):
        """进入上下文时调用"""
        print(f"连接到数据库: {self.connection_string}")
        self.connection = f"Connection to {self.connection_string}"
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        """退出上下文时调用"""
        print("关闭数据库连接")
        self.connection = None
        if exc_type is not None:
            print(f"发生错误: {exc_val}")
        return False  # 不抑制异常
    
    def execute_query(self, query):
        """执行查询"""
        if self.connection is None:
            raise RuntimeError("数据库未连接")
        print(f"执行查询: {query}")
        return f"结果: {query}"

# 使用
with DatabaseConnection("postgresql://localhost/mydb") as db:
    result = db.execute_query("SELECT * FROM users")
    print(result)
# 自动调用 __exit__，即使发生异常也会关闭连接
```

---

## 📊 类方法和静态方法

### 类方法 vs 静态方法 vs 实例方法
```python
class Date:
    """日期类，展示不同类型的方法"""
    
    def __init__(self, year, month, day):
        self.year = year
        self.month = month
        self.day = day
    
    # 实例方法 - 操作实例数据
    def format_date(self, separator="/"):
        """格式化日期"""
        return f"{self.year}{separator}{self.month:02d}{separator}{self.day:02d}"
    
    # 类方法 - 操作类本身，可以访问类属性
    @classmethod
    def from_string(cls, date_string):
        """从字符串创建 Date 实例"""
        year, month, day = map(int, date_string.split("-"))
        return cls(year, month, day)
    
    @classmethod
    def get_current_year(cls):
        """获取当前年份（示例）"""
        return 2024  # 实际应该用 datetime
    
    # 静态方法 - 不需要访问类或实例数据
    @staticmethod
    def is_leap_year(year):
        """判断是否为闰年"""
        return year % 4 == 0 and (year % 100 != 0 or year % 400 == 0)
    
    @staticmethod
    def validate_date(year, month, day):
        """验证日期是否有效"""
        if month < 1 or month > 12:
            return False
        if day < 1 or day > 31:
            return False
        # 简化的日期验证
        return True

# 使用
# 实例方法
date1 = Date(2024, 3, 15)
print(date1.format_date())  # 2024/03/15

# 类方法
date2 = Date.from_string("2024-03-20")
print(date2.format_date())  # 2024/03/20

# 静态方法
print(Date.is_leap_year(2024))  # True
print(Date.validate_date(2024, 2, 30))  # False

# 也可以通过实例调用
print(date1.is_leap_year(2024))  # True
```

---

## 🏗️ 抽象基类

### 使用 ABC 模块
```python
from abc import ABC, abstractmethod
from typing import List

class Shape(ABC):
    """形状抽象基类"""
    
    def __init__(self, name):
        self.name = name
    
    @abstractmethod
    def area(self):
        """计算面积 - 抽象方法"""
        pass
    
    @abstractmethod
    def perimeter(self):
        """计算周长 - 抽象方法"""
        pass
    
    def describe(self):
        """具体方法 - 所有子类共享"""
        return f"{self.name}: 面积={self.area():.2f}, 周长={self.perimeter():.2f}"

class Rectangle(Shape):
    """矩形类"""
    
    def __init__(self, width, height):
        super().__init__("矩形")
        self.width = width
        self.height = height
    
    def area(self):
        return self.width * self.height
    
    def perimeter(self):
        return 2 * (self.width + self.height)

class Circle(Shape):
    """圆形类"""
    
    def __init__(self, radius):
        super().__init__("圆形")
        self.radius = radius
    
    def area(self):
        return 3.14159 * self.radius ** 2
    
    def perimeter(self):
        return 2 * 3.14159 * self.radius

# 使用
shapes: List[Shape] = [
    Rectangle(5, 3),
    Circle(4)
]

for shape in shapes:
    print(shape.describe())

# 不能实例化抽象类
# shape = Shape("抽象形状")  # TypeError!
```

### 注册虚拟子类
```python
from abc import ABC

class Drawable(ABC):
    """可绘制对象抽象基类"""
    
    @abstractmethod
    def draw(self):
        pass
    
    @classmethod
    def __subclasshook__(cls, subclass):
        """检查是否实现了 draw 方法"""
        if cls is Drawable:
            if any("draw" in B.__dict__ for B in subclass.__mro__):
                return True
        return NotImplemented

# 第三方类（我们无法修改）
class ThirdPartyShape:
    def draw(self):
        return "绘制第三方形状"

# 注册为虚拟子类
Drawable.register(ThirdPartyShape)

# 现在 ThirdPartyShape 被认为是 Drawable 的子类
print(issubclass(ThirdPartyShape, Drawable))  # True
print(isinstance(ThirdPartyShape(), Drawable))  # True
```

---

## 🔧 属性描述符

### 自定义描述符
```python
class ValidatedAttribute:
    """验证属性描述符"""
    
    def __init__(self, name, expected_type, min_value=None, max_value=None):
        self.name = name
        self.expected_type = expected_type
        self.min_value = min_value
        self.max_value = max_value
    
    def __get__(self, instance, owner):
        if instance is None:
            return self
        return instance.__dict__.get(self.name)
    
    def __set__(self, instance, value):
        if not isinstance(value, self.expected_type):
            raise TypeError(f"{self.name} 必须是 {self.expected_type.__name__}")
        
        if self.min_value is not None and value < self.min_value:
            raise ValueError(f"{self.name} 不能小于 {self.min_value}")
        
        if self.max_value is not None and value > self.max_value:
            raise ValueError(f"{self.name} 不能大于 {self.max_value}")
        
        instance.__dict__[self.name] = value

class Person:
    """使用描述符的 Person 类"""
    
    # 使用描述符
    name = ValidatedAttribute("name", str)
    age = ValidatedAttribute("age", int, 0, 150)
    height = ValidatedAttribute("height", (int, float), 0, 300)
    
    def __init__(self, name, age, height):
        self.name = name
        self.age = age
        self.height = height

# 使用
person = Person("Alice", 25, 170)
print(person.name)  # Alice

try:
    person.age = -5  # 触发验证错误
except ValueError as e:
    print(f"错误: {e}")

try:
    person.name = 123  # 触发类型错误
except TypeError as e:
    print(f"错误: {e}")
```

---

## 🎯 设计模式实现

### 单例模式
```python
class Singleton:
    """单例模式"""
    
    _instance = None
    
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
    
    def __init__(self, name):
        # 防止重复初始化
        if not hasattr(self, 'initialized'):
            self.name = name
            self.initialized = True

# 使用
s1 = Singleton("第一个实例")
s2 = Singleton("第二个实例")

print(s1 is s2)  # True
print(s1.name)   # 第一个实例
print(s2.name)   # 第一个实例（不会改变）
```

### 工厂模式
```python
from abc import ABC, abstractmethod

class Notification(ABC):
    """通知抽象类"""
    
    @abstractmethod
    def send(self, message):
        pass

class EmailNotification(Notification):
    def send(self, message):
        return f"发送邮件: {message}"

class SMSNotification(Notification):
    def send(self, message):
        return f"发送短信: {message}"

class PushNotification(Notification):
    def send(self, message):
        return f"发送推送: {message}"

class NotificationFactory:
    """通知工厂"""
    
    @staticmethod
    def create_notification(notification_type):
        if notification_type == "email":
            return EmailNotification()
        elif notification_type == "sms":
            return SMSNotification()
        elif notification_type == "push":
            return PushNotification()
        else:
            raise ValueError(f"未知的通知类型: {notification_type}")
    
    @classmethod
    def get_available_types(cls):
        return ["email", "sms", "push"]

# 使用
notification = NotificationFactory.create_notification("email")
print(notification.send("Hello!"))  # 发送邮件: Hello!
```

### 观察者模式
```python
class Subject:
    """主题（被观察者）"""
    
    def __init__(self):
        self._observers = []
    
    def attach(self, observer):
        """添加观察者"""
        if observer not in self._observers:
            self._observers.append(observer)
    
    def detach(self, observer):
        """移除观察者"""
        self._observers.remove(observer)
    
    def notify(self, message):
        """通知所有观察者"""
        for observer in self._observers:
            observer.update(message)

class Observer:
    """观察者基类"""
    
    def __init__(self, name):
        self.name = name
    
    def update(self, message):
        """接收更新"""
        print(f"{self.name} 收到消息: {message}")

# 使用
subject = Subject()

observer1 = Observer("观察者1")
observer2 = Observer("观察者2")

subject.attach(observer1)
subject.attach(observer2)

subject.notify("状态已更新")  # 两个观察者都会收到消息
```

---

## 📝 最佳实践

### 组合优于继承
```python
# ❌ 不推荐：过度使用继承
class Engine:
    def start(self):
        return "引擎启动"

class Car(Engine):  # Car 不是 Engine 的一种
    def drive(self):
        return f"汽车行驶 - {self.start()}"

# ✅ 推荐：使用组合
class Engine:
    def start(self):
        return "引擎启动"

class Wheels:
    def rotate(self):
        return "轮子转动"

class Car:
    def __init__(self):
        self.engine = Engine()
        self.wheels = Wheels()
    
    def drive(self):
        return f"汽车行驶 - {self.engine.start()} - {self.wheels.rotate()}"

# 使用组合
car = Car()
print(car.drive())
```

### 单一职责原则
```python
# ❌ 不推荐：一个类做太多事情
class UserManager:
    def __init__(self):
        self.users = []
    
    def add_user(self, user):
        self.users.append(user)
    
    def send_email(self, user, message):
        # 发送邮件逻辑
        pass
    
    def validate_user(self, user):
        # 验证用户逻辑
        pass
    
    def save_to_database(self, user):
        # 数据库保存逻辑
        pass

# ✅ 推荐：拆分为多个单一职责的类
class UserRepository:
    """负责用户数据存储"""
    def save(self, user):
        # 数据库保存逻辑
        pass

class EmailService:
    """负责发送邮件"""
    def send(self, recipient, message):
        # 发送邮件逻辑
        pass

class UserValidator:
    """负责用户验证"""
    def validate(self, user):
        # 验证用户逻辑
        pass

class UserManager:
    """协调各个服务"""
    def __init__(self):
        self.repository = UserRepository()
        self.email_service = EmailService()
        self.validator = UserValidator()
    
    def register_user(self, user):
        if self.validator.validate(user):
            self.repository.save(user)
            self.email_service.send(user.email, "欢迎!")
            return True
        return False
```

### 依赖注入
```python
from abc import ABC, abstractmethod

class Database(ABC):
    """数据库抽象"""
    
    @abstractmethod
    def save(self, data):
        pass

class MySQLDatabase(Database):
    def save(self, data):
        return f"MySQL 保存: {data}"

class PostgreSQLDatabase(Database):
    def save(self, data):
        return f"PostgreSQL 保存: {data}"

class UserService:
    """用户服务，依赖数据库抽象"""
    
    def __init__(self, database: Database):  # 依赖注入
        self.database = database
    
    def create_user(self, user_data):
        # 业务逻辑
        result = self.database.save(user_data)
        return result

# 使用
mysql_db = MySQLDatabase()
postgres_db = PostgreSQLDatabase()

user_service1 = UserService(mysql_db)
user_service2 = UserService(postgres_db)

print(user_service1.create_user({"name": "Alice"}))
print(user_service2.create_user({"name": "Bob"}))
```

### 不可变数据类
```python
from dataclasses import dataclass
from typing import List

@dataclass(frozen=True)  # 不可变
class ImmutablePoint:
    """不可变点类"""
    x: float
    y: float
    
    def distance_to(self, other):
        """计算到另一个点的距离"""
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

@dataclass
class Configuration:
    """配置类"""
    host: str
    port: int = 8080
    timeout: float = 30.0
    enabled_features: List[str] = None
    
    def __post_init__(self):
        """初始化后处理"""
        if self.enabled_features is None:
            self.enabled_features = []

# 使用
point1 = ImmutablePoint(3.0, 4.0)
point2 = ImmutablePoint(0.0, 0.0)

print(point1.distance_to(point2))  # 5.0
# point1.x = 5.0  # 错误！不可变对象

config = Configuration("localhost", 5432)
print(config)  # Configuration(host='localhost', port=5432, timeout=30.0, enabled_features=[])
```

---

## 🏆 高级技巧

### 元类
```python
class SingletonMeta(type):
    """单例元类"""
    
    _instances = {}
    
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class DatabaseConnection(metaclass=SingletonMeta):
    """使用元类实现的单例"""
    
    def __init__(self):
        print("创建数据库连接")

# 使用
db1 = DatabaseConnection()  # 输出: 创建数据库连接
db2 = DatabaseConnection()  # 没有输出
print(db1 is db2)  # True
```

### 动态创建类
```python
def create_class(class_name, base_classes, attributes):
    """动态创建类"""
    return type(class_name, base_classes, attributes)

# 动态创建类
DynamicClass = create_class(
    "DynamicClass",
    (object,),
    {
        'x': 10,
        'get_x': lambda self: self.x,
        'set_x': lambda self, value: setattr(self, 'x', value)
    }
)

obj = DynamicClass()
print(obj.get_x())  # 10
obj.set_x(20)
print(obj.get_x())  # 20
```

---

## 📚 总结

**核心原则：**
- ✅ **封装**：隐藏内部实现，提供清晰接口
- ✅ **继承**：建立 is-a 关系，实现代码复用
- ✅ **多态**：同一接口，不同实现
- ✅ **抽象**：定义接口，隐藏细节

**最佳实践：**
- 🎯 组合优于继承
- 🎯 单一职责原则
- 🎯 依赖倒置原则
- 🎯 使用属性装饰器进行访问控制
- 🎯 合理使用抽象基类定义接口

**设计模式：**
- 🔧 工厂模式：创建对象
- 🔧 单例模式：确保唯一实例
- 🔧 观察者模式：事件通知
- 🔧 策略模式：算法族封装

**性能提示：**
- ⚡ 使用 `__slots__` 减少内存使用
- ⚡ 避免深度继承层次
- ⚡ 合理使用属性描述符
- ⚡ 考虑使用数据类简化代码

掌握这些面向对象编程技巧将帮助你构建更健壮、更灵活、更易维护的 Python 应用程序！我们准备创建一个关于Python面向对象编程（OOP）的速查表，包括类与对象、继承、多态、封装、特殊方法、属性、抽象基类等，以及最佳实践。

