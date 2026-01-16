# Python 回调函数 (Callback) 完整指南

## 什么是回调函数？

回调函数是作为参数传递给另一个函数的函数，在特定事件发生或条件满足时被"回调"执行。

## 基础用法

### 1. 简单回调示例
```python
def callback_function(message):
    print(f"回调收到消息: {message}")

def main_function(data, callback):
    print("处理数据中...")
    result = data * 2
    callback(f"处理完成，结果是: {result}")

# 使用回调
main_function(5, callback_function)
```

### 2. 带多个参数的回调
```python
def success_callback(result, execution_time):
    print(f"操作成功! 结果: {result}, 耗时: {execution_time}ms")

def error_callback(error_msg, error_code):
    print(f"操作失败! 错误: {error_msg}, 代码: {error_code}")

def process_data(data, on_success, on_error):
    try:
        import time
        start_time = time.time()
        
        # 模拟处理
        if data < 0:
            raise ValueError("数据不能为负数")
            
        result = data ** 2
        execution_time = (time.time() - start_time) * 1000
        on_success(result, execution_time)
        
    except Exception as e:
        on_error(str(e), 500)

# 使用
process_data(10, success_callback, error_callback)
process_data(-5, success_callback, error_callback)
```

## 回调在不同场景的应用

### 1. 事件驱动编程
```python
class Button:
    def __init__(self):
        self.click_handlers = []
    
    def on_click(self, callback):
        self.click_handlers.append(callback)
    
    def click(self):
        print("按钮被点击了!")
        for handler in self.click_handlers:
            handler()

# 使用
def show_message():
    print("显示消息!")

def log_click():
    print("点击事件已记录")

button = Button()
button.on_click(show_message)
button.on_click(log_click)
button.click()
```

### 2. 异步操作回调
```python
import threading
import time

def long_running_operation(data, callback):
    def worker():
        print(f"开始处理数据: {data}")
        time.sleep(2)  # 模拟耗时操作
        result = f"处理后的 {data}"
        callback(result)
    
    thread = threading.Thread(target=worker)
    thread.start()
    return thread

def operation_complete(result):
    print(f"操作完成: {result}")

# 使用
long_running_operation("测试数据", operation_complete)
print("主线程继续执行...")
```

### 3. 数据处理管道
```python
def data_processor(data, *callbacks):
    result = data
    for callback in callbacks:
        result = callback(result)
    return result

# 定义处理函数
def filter_even(numbers):
    return [x for x in numbers if x % 2 == 0]

def square_numbers(numbers):
    return [x ** 2 for x in numbers]

def sort_numbers(numbers):
    return sorted(numbers)

# 使用回调链处理数据
data = [1, 2, 3, 4, 5, 6, 7, 8, 9]
result = data_processor(data, filter_even, square_numbers, sort_numbers)
print(f"处理结果: {result}")
```

## 高级回调模式

### 1. 带状态的回调（使用类）
```python
class ProgressTracker:
    def __init__(self, total_steps):
        self.total_steps = total_steps
        self.current_step = 0
    
    def update_progress(self, step_name):
        self.current_step += 1
        progress = (self.current_step / self.total_steps) * 100
        print(f"[{progress:.1f}%] 完成: {step_name}")

def complex_operation(steps, progress_callback):
    for i, step in enumerate(steps, 1):
        print(f"执行步骤: {step}")
        progress_callback(step)

# 使用
steps = ["数据加载", "数据清洗", "特征工程", "模型训练", "结果评估"]
tracker = ProgressTracker(len(steps))
complex_operation(steps, tracker.update_progress)
```

### 2. 装饰器实现回调注册
```python
class EventManager:
    def __init__(self):
        self._events = {}
    
    def register(self, event_name):
        def decorator(callback):
            if event_name not in self._events:
                self._events[event_name] = []
            self._events[event_name].append(callback)
            return callback
        return decorator
    
    def trigger(self, event_name, *args, **kwargs):
        if event_name in self._events:
            for callback in self._events[event_name]:
                callback(*args, **kwargs)

# 使用
event_manager = EventManager()

@event_manager.register('user_login')
def send_welcome_email(user):
    print(f"发送欢迎邮件给: {user}")

@event_manager.register('user_login')
def update_login_stats(user):
    print(f"更新登录统计: {user}")

@event_manager.register('user_logout')
def cleanup_session(user):
    print(f"清理会话: {user}")

# 触发事件
event_manager.trigger('user_login', 'Alice')
event_manager.trigger('user_logout', 'Bob')
```

### 3. 带错误处理的回调系统
```python
def safe_callback_executor(callback, *args, **kwargs):
    """
    安全执行回调，处理异常
    """
    try:
        return callback(*args, **kwargs)
    except Exception as e:
        print(f"回调执行错误: {e}")
        return None

class RobustCallbackSystem:
    def __init__(self):
        self.callbacks = []
    
    def add_callback(self, callback, priority=0):
        self.callbacks.append((priority, callback))
        self.callbacks.sort(key=lambda x: x[0], reverse=True)
    
    def execute_all(self, *args, **kwargs):
        results = []
        for priority, callback in self.callbacks:
            result = safe_callback_executor(callback, *args, **kwargs)
            results.append((callback.__name__, result))
        return results

# 使用
system = RobustCallbackSystem()

def high_priority_task(data):
    return f"高优先级处理: {data.upper()}"

def medium_priority_task(data):
    return f"中优先级处理: {data * 2}"

def failing_task(data):
    raise ValueError("这个任务会失败")

system.add_callback(high_priority_task, priority=10)
system.add_callback(medium_priority_task, priority=5)
system.add_callback(failing_task, priority=1)

results = system.execute_all("hello")
print("执行结果:", results)
```

## 回调在 Web 开发中的应用

### 1. Flask 路由回调
```python
from flask import Flask, request, jsonify

app = Flask(__name__)

# 回调存储
user_actions = []

def register_user_action(action, user_id):
    user_actions.append({
        'action': action,
        'user_id': user_id,
        'timestamp': '2024-01-01 10:00:00'
    })

@app.route('/user/<user_id>/purchase', methods=['POST'])
def purchase_item(user_id):
    data = request.json
    item = data.get('item')
    
    # 业务逻辑
    print(f"用户 {user_id} 购买了 {item}")
    
    # 执行回调
    register_user_action(f'purchase_{item}', user_id)
    
    return jsonify({'status': 'success', 'user_actions': user_actions})
```

### 2. 请求处理管道
```python
class RequestProcessor:
    def __init__(self):
        self.pre_processors = []
        self.post_processors = []
    
    def add_pre_processor(self, callback):
        self.pre_processors.append(callback)
    
    def add_post_processor(self, callback):
        self.post_processors.append(callback)
    
    def process(self, request):
        # 预处理
        for pre_processor in self.pre_processors:
            request = pre_processor(request)
        
        # 主处理逻辑
        response = self._handle_request(request)
        
        # 后处理
        for post_processor in self.post_processors:
            response = post_processor(response)
        
        return response
    
    def _handle_request(self, request):
        return f"处理请求: {request}"

# 定义处理器
def auth_middleware(request):
    print("执行身份验证...")
    return f"{request} [已认证]"

def log_middleware(request):
    print("记录请求日志...")
    return request

def timing_middleware(response):
    print("添加计时信息...")
    return f"{response} [耗时: 100ms]"

# 使用
processor = RequestProcessor()
processor.add_pre_processor(auth_middleware)
processor.add_pre_processor(log_middleware)
processor.add_post_processor(timing_middleware)

result = processor.process("用户数据请求")
print(result)
```

## 回调最佳实践

### 1. 错误处理最佳实践
```python
from functools import wraps
import logging

logging.basicConfig(level=logging.INFO)

def callback_error_handler(callback):
    """
    回调错误处理装饰器
    """
    @wraps(callback)
    def wrapper(*args, **kwargs):
        try:
            return callback(*args, **kwargs)
        except Exception as e:
            logging.error(f"回调 {callback.__name__} 执行失败: {e}")
            # 可以选择重试、回退或其他错误处理逻辑
            return None
    return wrapper

@callback_error_handler
def risky_callback(data):
    if len(data) < 3:
        raise ValueError("数据太短")
    return data.upper()

# 使用
result1 = risky_callback("hello")  # 正常执行
result2 = risky_callback("hi")     # 错误但被捕获
```

### 2. 回调注册表模式
```python
class CallbackRegistry:
    def __init__(self):
        self._registry = {}
        self._default_callbacks = []
    
    def register(self, event_type, callback):
        if event_type not in self._registry:
            self._registry[event_type] = []
        self._registry[event_type].append(callback)
    
    def register_default(self, callback):
        self._default_callbacks.append(callback)
    
    def dispatch(self, event_type, *args, **kwargs):
        results = []
        
        # 执行特定事件类型的回调
        if event_type in self._registry:
            for callback in self._registry[event_type]:
                results.append(callback(*args, **kwargs))
        
        # 执行默认回调
        for callback in self._default_callbacks:
            results.append(callback(event_type, *args, **kwargs))
        
        return results

# 使用
registry = CallbackRegistry()

# 注册特定事件回调
registry.register('email_sent', lambda: print("邮件发送成功"))
registry.register('user_created', lambda user: print(f"用户创建: {user}"))

# 注册默认回调（处理所有事件）
registry.register_default(lambda event, *args: print(f"默认处理事件: {event}"))

# 分发事件
registry.dispatch('email_sent')
registry.dispatch('user_created', 'Alice')
registry.dispatch('unknown_event')
```

### 3. 异步回调模式
```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class AsyncCallbackManager:
    def __init__(self):
        self.executor = ThreadPoolExecutor(max_workers=5)
    
    async def execute_callbacks_async(self, callbacks, *args):
        """异步执行多个回调"""
        loop = asyncio.get_event_loop()
        
        tasks = []
        for callback in callbacks:
            # 在线程池中执行阻塞回调
            task = loop.run_in_executor(
                self.executor, 
                callback, 
                *args
            )
            tasks.append(task)
        
        # 等待所有回调完成
        results = await asyncio.gather(*tasks, return_exceptions=True)
        return results

# 异步回调示例
async def main():
    manager = AsyncCallbackManager()
    
    def slow_callback_1(data):
        import time
        time.sleep(1)
        return f"回调1处理: {data}"
    
    def slow_callback_2(data):
        import time
        time.sleep(2)
        return f"回调2处理: {data}"
    
    callbacks = [slow_callback_1, slow_callback_2]
    results = await manager.execute_callbacks_async(callbacks, "测试数据")
    
    print("所有回调完成:", results)

# 运行
# asyncio.run(main())
```

## 实际应用案例

### 1. 数据验证框架
```python
class DataValidator:
    def __init__(self):
        self.validators = {}
    
    def add_validator(self, field_name, validator_callback, error_message):
        if field_name not in self.validators:
            self.validators[field_name] = []
        self.validators[field_name].append((validator_callback, error_message))
    
    def validate(self, data):
        errors = []
        
        for field_name, validators in self.validators.items():
            field_value = data.get(field_name)
            
            for validator, error_msg in validators:
                if not validator(field_value):
                    errors.append(f"{field_name}: {error_msg}")
                    break  # 一个字段一个错误
        
        return len(errors) == 0, errors

# 定义验证器
def is_required(value):
    return value is not None and value != ""

def is_email(value):
    import re
    return re.match(r'^[\w\.-]+@[\w\.-]+\.\w+$', value) if value else False

def is_positive_number(value):
    return isinstance(value, (int, float)) and value > 0

# 使用验证框架
validator = DataValidator()
validator.add_validator('name', is_required, "姓名不能为空")
validator.add_validator('email', is_email, "邮箱格式不正确")
validator.add_validator('age', is_positive_number, "年龄必须是正数")

test_data = {'name': 'Alice', 'email': 'alice@example.com', 'age': 25}
is_valid, errors = validator.validate(test_data)
print(f"验证结果: {is_valid}, 错误: {errors}")
```

### 2. 插件系统
```python
class PluginSystem:
    def __init__(self):
        self.plugins = []
    
    def register_plugin(self, plugin):
        self.plugins.append(plugin)
    
    def execute_hook(self, hook_name, *args, **kwargs):
        results = []
        for plugin in self.plugins:
            if hasattr(plugin, hook_name):
                hook_method = getattr(plugin, hook_name)
                result = hook_method(*args, **kwargs)
                results.append(result)
        return results

class BasePlugin:
    def on_start(self):
        pass
    
    def on_data_received(self, data):
        pass
    
    def on_shutdown(self):
        pass

class LoggingPlugin(BasePlugin):
    def on_start(self):
        print("日志插件: 系统启动")
        return "logging_started"
    
    def on_data_received(self, data):
        print(f"日志插件: 收到数据 {data}")
        return "data_logged"

class AnalyticsPlugin(BasePlugin):
    def on_data_received(self, data):
        print(f"分析插件: 分析数据 {data}")
        return "data_analyzed"

# 使用插件系统
system = PluginSystem()
system.register_plugin(LoggingPlugin())
system.register_plugin(AnalyticsPlugin())

# 执行钩子
system.execute_hook('on_start')
system.execute_hook('on_data_received', 'sample data')
```

## 总结

回调函数是 Python 中强大的编程模式，适用于：

1. **事件处理** - GUI 编程、Web 框架
2. **异步编程** - 非阻塞操作完成后的处理
3. **插件系统** - 可扩展的应用程序架构
4. **数据处理管道** - 数据转换和验证
5. **中间件模式** - 请求/响应处理链

**最佳实践建议**：
- 始终处理回调中的异常
- 使用类型注解提高代码可读性
- 考虑使用装饰器模式管理回调
- 对于复杂系统，使用回调注册表
- 在性能敏感场景考虑异步回调

回调模式让代码更加模块化和可扩展，是现代 Python 应用程序开发中的重要工具。
