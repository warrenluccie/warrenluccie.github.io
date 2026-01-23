# Python最为常见的陷阱:可变的默认参数

- [mutable default arguments](https://docs.python-guide.org/writing/gotchas/#mutable-default-arguments)

## Common Gotchas

​	**For the most part, Python aims to be a clean and consistent language that avoids surprises. However, there are a few cases that can be confusing for newcomers.**

**在大多数情况下，Python 致力于成为一种简洁且一致的语言，避免出现意外情况。然而，对于初学者来说，还是有一些情况可能会令人感到困惑。**

**Some of these cases are intentional but can be potentially surprising. Some could arguably be considered language warts. In general, what follows is a collection of potentially tricky behavior that might seem strange at first glance, but are generally sensible, once you’re aware of the underlying cause for the surprise.**

**其中一些情况是故意为之，但可能会令人意外。有些情况或许可以被看作是语言上的瑕疵。总的来说，以下是一些乍一看可能显得奇怪，但一旦了解其背后的原因，就会觉得合情合理的潜在棘手行为的集合。**



## Mutable Default Arguments

**Seemingly the *most* common surprise new Python programmers encounter is Python’s treatment of mutable default arguments in function definitions.**

**似乎新接触 Python 的程序员最常遇到的意外情况是 Python 在函数定义中对可变默认参数的处理方式。**



### What You Wrote

```python
def append_to(element, to_list=[]):
    """
    将元素添加到列表中
    :param element:  要添加的元素
    :param to_list:  要添加到的列表，默认值为空列表
    :return:  添加元素后的列表
    """
    to_list.append(element)
    return to_list
```



### What You Might Have Expected to Happen

```python
my_list = append_to(12)
print(my_list)  # [12]
my_other_list = append_to(42)
print(my_other_list)  # [42]
# print(my_list)
```



**A new list is created each time the function is called if a second argument isn’t provided, so that the output is:**

**如果未提供第二个参数，每次调用该函数时都会创建一个新的列表，因此输出为：**

```python
[12]
[42]
```



### What Actually Happens

```python
[12]
[12, 42]
```



**A new list is created *once* when the function is defined, and the same list is used in each successive call.**

**在定义函数时会创建一个新的列表，且在每次后续调用中都使用同一个列表。**

**Python’s default arguments are evaluated *once* when the function is defined, not each time the function is called (like it is in say, Ruby). This means that if you use a mutable default argument and mutate it, you *will* and have mutated that object for all future calls to the function as well.**

**Python 的默认参数在函数定义时只求值一次，而不是每次调用函数时都求值（这与 Ruby 不同）。这意味着，如果您使用可变的默认参数并对其进行修改，那么在以后对该函数的所有调用中，您都已修改了该对象。**



### visually debug your code step-by-step 

- [pythontutor](https://pythontutor.com/python-compiler.html#mode=edit)

```python
def append_to(element, to_list=[]):
    """
    将元素添加到列表中
    :param element:  要添加的元素
    :param to_list:  要添加到的列表，默认值为空列表
    :return:  添加元素后的列表
    """
    to_list.append(element)
    return to_list


def main():
    my_list = append_to(12)
    print(my_list)  # [12]
    my_other_list = append_to(42)
    print(my_other_list)  # [12,42]
    print(my_list)  # [12, 42]


if __name__ == "__main__":
    main()
```



#### 函数append_to在定义时就已经在内存中创建了一个空列表

![1769137551967](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769137551967.png)

#### main函数的定义

![1769137721593](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769137721593.png)



#### main函数的调用

- 在main函数体中只做一件事情：调用之前定义的append_to函数方法

![1769137873881](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769137873881.png)





#### 调用append_to函数时只传递了第一个参数，并没有传递第二个参数

- **第二个参数：to_list有一个默认值，它是一个空列表。to_list也指向append_to函数定义时在内存中已经创建的empty list**

![1769138116314](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769138116314.png)



#### 把传递的第一个函数参数添加到空列表中

![1769138855895](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769138855895.png)



#### 将列表对象作为返回值返回

![1769138972089](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769138972089.png)

#### 将append_to函数的return value赋值给变量my_list

![1769146647271](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769146647271.png)



#### 打印输出my_list的值

![1769146715053](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769146715053.png)





#### 再次调用append_to函数，第二个参数仍未传递值，使用默认值传递参数

![1769147178550](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769147178550.png)



#### 第二次调用append_to函数时，解释器并未创建新的空列表，而是使用之前的列表对象

![1769148316078](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769148316078.png)



#### 将第一次调用函数后列表变为[12]和第二次调用函数后的值[12,42]一起作为返回值返回

![1769148406547](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769148406547.png)



#### 变量my_list和变量my_other_list都引用内存中同一个地址

![1769148557104](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/1769148557104.png)



![image-20260123141149793](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/image-20260123141149793.png)



![image-20260123141251931](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/image-20260123141251931.png)







### What You Should Do Instead

**Create a new object each time the function is called, by using a default arg to signal that no argument was provided ([`None`](https://docs.python.org/3/library/constants.html#None) is often a good choice).**

**每次调用函数时都创建一个新对象，通过使用默认参数来表明未提供参数（通常将默认参数设置为None是一个不错的选择）。**

```python
def append_to(element, to=None):
    if to is None:
        to = []
    to.append(element)
    return to

def append_to_using_immutable_default_arguments(element,to=None):
    """
    将元素添加到列表中，使用不可变默认参数
    :param element:  要添加的元素
    :param to:  要添加到的列表，默认值为空列表
    :return:  添加元素后的列表
    """
    if to is None:
        to = []
    to.append(element)
    return to

```



**Do not forget, you are passing a *list* object as the second argument.**

**别忘了，您传递的是一个列表对象作为第二个参数。**



### When the Gotcha Isn’t a Gotcha

**Sometimes you can specifically “exploit” (read: use as intended) this behavior to maintain state between calls of a function. This is often done when writing a caching function.**

**有时您可以特意利用（即：按预期使用）这种行为在函数调用之间保持状态。这在编写缓存函数时经常用到。**





## Late Binding Closures

-  延迟绑定闭包

**Another common source of confusion is the way Python binds its variables in closures (or in the surrounding global scope).**

**另一个常见的混淆来源是 Python 在闭包（或在周围的全局作用域）中绑定变量的方式。**



### What You Wrote

```python
def create_multipliers():
    return [lambda x : i * x for i in range(5)]
```



### What You Might Have Expected to Happen

```python
def create_multipliers():
    return [lambda x : i * x for i in range(5)]


def main():
    for multiplier in create_multipliers():
        print(multiplier(2))  # 0, 2, 4, 6, 8
```



**A list containing five functions that each have their own closed-over `i` variable that multiplies their argument, producing:**

**一个包含五个函数的列表，每个函数都有自己的闭包变量i，用于对其参数进行乘法运算，从而得到：**

```python
0
2
4
6
8
```



### What Actually Happens

```python
8
8
8
8
8
```



**Five functions are created; instead all of them just multiply `x` by 4.创建了五个函数；但实际上它们都只是乘以 4。**

**Python’s closures are *late binding*. This means that the values of variables used in closures are looked up at the time the inner function is called.**

**Python 的闭包是延迟绑定的。 这意味着在调用内部函数时，会查找闭包中所使用变量的值。**

**Here, whenever *any* of the returned functions are called, the value of `i` is looked up in the surrounding scope at call time. By then, the loop has completed and `i` is left with its final value of 4.在这里，每当调用任何返回的函数时，都会在调用时在周围作用域中查找 的值。到那时，循环已经完成， 被赋予了其最终值 4 。**

**What’s particularly nasty about this gotcha is the seemingly prevalent misinformation that this has something to do with [lambdas](https://docs.python.org/3/reference/expressions.html#lambda) in Python. Functions created with a `lambda` expression are in no way special, and in fact the same exact behavior is exhibited by just using an ordinary `def`:**

**这种陷阱特别令人讨厌的地方在于，似乎普遍存在一种错误信息，即认为这与 Python 中的 lambda 表达式有关。**

**用 lambda 表达式创建的函数在任何方面都没有特殊之处，实际上，仅使用普通函数也会表现出完全相同的行为。:**

```python
def create_multipliers_using_closure():
    """
    使用闭包创建一个乘法器列表
    :return:  包含5个乘法器的列表，每个乘法器将输入乘以其索引
    """
    multipliers = []  # 存储乘法器的列表

    for i in range(5):
        """
        创建一个乘法器，将输入乘以其索引
        :param x:  要乘以的输入值
        :return:  输入值乘以索引的结果
        """
        def multiplier(x):
            return i * x
        multipliers.append(multiplier)  # 将乘法器添加到列表中

    return multipliers
```





- create_multipliers函数返回值是一个列表对象（可迭代遍历），列表内部存储的是lambda匿名函数的返回值

![image-20260123144026252](https://raw.githubusercontent.com/warrenluccie/quantpictures/main/image-20260123144026252.png)



### What  You  Should Do Instead

**The most general solution is arguably a bit of a hack. Due to Python’s aforementioned behavior concerning evaluating default arguments to functions (see [Mutable Default Arguments](https://docs.python-guide.org/writing/gotchas/#default-args)), you can create a closure that binds immediately to its arguments by using a default arg like so:**

**最通用的解决方案可以说有点像变通方法。由于 Python 上述有关函数默认参数求值的行为（参见可变默认参数），您可以使用如下默认参数创建一个立即绑定到其参数的闭包：**

```python
def create_multipliers():
    return [lambda x, i=i : i * x for i in range(5)]
```



**Alternatively, you can use the functools.partial function:**

**或者，您可以使用 functools.partial 函数：**

```python
from functools import partial
from operator import mul

def create_multipliers():
    return [partial(mul, i) for i in range(5)]
```



### When the Gotcha Isn’t a Gotcha

**Sometimes you want your closures to behave this way. Late binding is good in lots of situations. Looping to create unique functions is unfortunately a case where they can cause hiccups.**

**有时您希望闭包能以这种方式运行。在很多情况下，延迟绑定都是不错的。但不幸的是，在循环中创建独特函数时，它们可能会导致问题。**



## Bytecode (.pyc) Files Everywhere!



**By default, when executing Python code from files, the Python interpreter will automatically write a bytecode version of that file to disk, e.g. `module.pyc`.默认情况下，当从文件执行 Python 代码时，Python 解释器会自动将该文件的字节码版本写入磁盘，例如.**

**These `.pyc` files should not be checked into your source code repositories.这些文件不应提交到您的源代码库中。**

**Theoretically, this behavior is on by default for performance reasons. Without these bytecode files, Python would re-generate the bytecode every time the file is loaded.从理论上讲，出于性能方面的考虑，这种行为默认是开启的。 如果没有这些字节码文件，Python 每次加载文件时都会重新生成字节码。**

### Disabling Bytecode (.pyc) Files

- **禁用字节码（.pyc）文件**

**Luckily, the process of generating the bytecode is extremely fast, and isn’t something you need to worry about while developing your code.幸运的是，生成字节码的过程非常迅速，在开发代码时您无需为此操心。**

**Those files are annoying, so let’s get rid of them!那些文件真烦人，咱们把它们删了吧！**

```bash
$ export PYTHONDONTWRITEBYTECODE=1
```

With the `$PYTHONDONTWRITEBYTECODE` environment variable set, Python will no longer write these files to disk, and your development environment will remain nice and clean.

设置好环境变量后，Python 将不再将这些文件写入磁盘，您的开发环境将保持整洁干净。

I recommend setting this environment variable in your `~/.profile`.我建议在您的.~/.profile中设置此环境变量。



### Removing Bytecode (.pyc) Files

- 删除字节码（.pyc）文件

Here’s nice trick for removing all of these files, if they already exist:

这里有一个不错的技巧，可以删除所有已存在的此类文件：

```bash
$ find . -type f -name "*.py[co]" -delete -or -type d -name "__pycache__" -delete
```

Run that from the root directory of your project, and all `.pyc` files will suddenly vanish. Much better.

在项目的根目录下运行该命令，所有文件`.pyc`都会突然消失。好多了。



### Version Control Ignores

- 版本控制忽略项

If you still need the `.pyc` files for performance reasons, you can always add them to the ignore files of your version control repositories. Popular version control systems have the ability to use wildcards defined in a file to apply special rules.

如果出于性能方面的原因您仍需这些文件，您始终可以将它们添加到版本控制库的忽略文件中。流行的版本控制系统具有使用文件中定义的通配符来应用特殊规则的能力.

An ignore file will make sure the matching files don’t get checked into the repository. 

[Git](https://git-scm.com/) uses `.gitignore` while [Mercurial](https://www.mercurial-scm.org/) uses `.hgignore`.

忽略文件可确保匹配的文件不会被检入到存储库中。 Git 使用 `.gitignore`，而 Mercurial 使用 `.hgignore`。

At the minimum your ignore files should look like this.至少您的忽略文件应该像这样。

```bash
syntax:glob   # This line is not needed for .gitignore files.
*.py[cod]     # Will match .pyc, .pyo and .pyd files.
__pycache__/  # Exclude the whole folder
```

You may wish to include more files and directories depending on your needs. The next time you commit to the repository, these files will not be included.

您可以根据需要包含更多文件和目录。 下次您向存储库提交时，这些文件将不会被包含。