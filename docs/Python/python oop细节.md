





# 静态方法

Python中的`@staticmethod`（静态方法）通过在类中定义时添加此装饰器来创建。

它不需要**实例化类**即可通过类名调用，也不需要强制传递`self`或`cls`参数。静态方法适用于逻辑上归属于类、但不需要访问类属性或实例属性的工具型函数。 

**用法总结：**

- **定义**：在方法上方添加 `@staticmethod`。
- **参数**：不需要 `self` 或 `cls`。
- **调用**：可以通过类名 `ClassName.method()` 或实例对象 `instance.method()` 调用。
- **场景**：通常用于实现工具类逻辑，即与类本身的状态无关的函数。 



**示例代码：**

```python
class MyClass:
    @staticmethod
    def add(x, y):
        return x + y

# 1. 直接通过类名调用（推荐）
result = MyClass.add(5, 3)
print(result)  # 输出 8

# 2. 通过实例化对象来调用
obj = MyClass()
print(obj.add(2, 4))  # 输出 6

```











