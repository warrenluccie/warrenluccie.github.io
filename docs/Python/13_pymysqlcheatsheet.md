# pymysql库速查表及最佳工程实践



## 概述

pymysql是Python中连接MySQL数据库的纯Python驱动，兼容MySQLdb API，支持Python 3.5+。

通常我们的Python脚本是与需要后台数据库进行交互操作(CRUD):增删改查，pymysql就是Python连接MySQL数据库的一个库，它遵循Python DB-API 2.0规范。

- 使用Python操作MySQL数据库的一般步骤如下所示：

  > `连接数据库 -> 创建游标对象 -> 执行SQL语句 -> 提交事务 -> 关闭游标和连接`

- 常用操作

  > - 查询：使用execute()执行SQL，然后使用fetchone()、fetchall()或fetchmany()获取结果。
  >
  > - 插入、更新、删除：同样使用execute()执行SQL，然后commit()提交。

- 防止SQL注入

  > - 使用参数化查询，不要直接拼接SQL字符串。

  

- 使用上下文管理器

  > - 使用上下文管理器（with语句）自动管理资源（连接和游标）。

- 使用连接池（如DBUtils）提高性能。



- 注意事务处理，确保数据一致性。



---

## 一、核心特性与安装

### 1.1 核心特性
- ✅ **纯Python实现**：无需编译C扩展
- ✅ **兼容MySQLdb**：API与MySQLdb兼容
- ✅ **支持Python 3.5+**：原生支持Python 3
- ✅ **SSL连接**：支持安全的数据库连接
- ✅ **连接池**：可通过第三方库实现
- ✅ **异步支持**：通过aiomysql支持异步操作
- ✅ **类型转换**：自动进行Python与MySQL类型转换

### 1.2 安装与版本
```bash
# 基础安装
pip install pymysql

# 安装异步版本
pip install aiomysql

# 安装连接池支持
pip install dbutils

# 安装带SSL支持的版本
pip install pymysql[rsa]

# 指定版本
pip install pymysql==1.0.2
```



### 1.3 版本兼容性

| pymysql版本 | Python版本 | MySQL版本 | 主要特性 |
|------------|-----------|-----------|----------|
| 1.0.x | 3.7+ | 5.5+ | 支持Python 3.7+, MySQL 8.0认证 |
| 0.10.x | 3.5+ | 5.5+ | 稳定版本，生产环境推荐 |
| 0.9.x | 2.7/3.4+ | 5.0+ | 旧版本支持 |

---



## 二、基础连接与配置

### 2.1 基本连接

```python
import pymysql
import logging
from datetime import datetime
from typing import Optional, Dict, Any, List, Tuple
import ssl
import warnings

# 设置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class MySQLConnection:
    """MySQL连接管理器"""
    
    @staticmethod
    def create_connection(
        host: str = 'localhost',
        port: int = 3306,
        user: str = 'root',
        password: str = '',
        database: str = None,
        charset: str = 'utf8mb4',
        autocommit: bool = False,
        connect_timeout: int = 10,
        read_timeout: int = 30,
        write_timeout: int = 30,
        ssl_enabled: bool = False,
        ssl_ca: str = None,
        ssl_cert: str = None,
        ssl_key: str = None,
        cursorclass: Any = pymysql.cursors.DictCursor,
        **kwargs
    ) -> pymysql.Connection:
        """
        创建数据库连接
        
        Args:
            host: 数据库主机
            port: 端口
            user: 用户名
            password: 密码
            database: 数据库名
            charset: 字符集
            autocommit: 是否自动提交
            connect_timeout: 连接超时(秒)
            read_timeout: 读取超时(秒)
            write_timeout: 写入超时(秒)
            ssl_enabled: 是否启用SSL
            ssl_ca: CA证书路径
            ssl_cert: 客户端证书路径
            ssl_key: 客户端密钥路径
            cursorclass: 游标类
            **kwargs: 其他参数
            
        Returns:
            pymysql.Connection 对象
        """
        try:
            # SSL配置
            ssl_config = None
            if ssl_enabled:
                ssl_config = {
                    'ca': ssl_ca,
                    'cert': ssl_cert,
                    'key': ssl_key,
                }
                # 清理None值
                ssl_config = {k: v for k, v in ssl_config.items() if v}
            
            connection = pymysql.connect(
                host=host,
                port=port,
                user=user,
                password=password,
                database=database,
                charset=charset,
                autocommit=autocommit,
                connect_timeout=connect_timeout,
                read_timeout=read_timeout,
                write_timeout=write_timeout,
                ssl=ssl_config,
                cursorclass=cursorclass,
                **kwargs
            )
            
            logger.info(f"成功连接到数据库: {host}:{port}/{database}")
            return connection
            
        except pymysql.err.OperationalError as e:
            logger.error(f"数据库连接失败: {e}")
            raise
        except Exception as e:
            logger.error(f"未知连接错误: {e}")
            raise
    
    @staticmethod
    def test_connection(connection: pymysql.Connection) -> bool:
        """测试连接是否有效"""
        try:
            with connection.cursor() as cursor:
                cursor.execute("SELECT 1")
                result = cursor.fetchone()
                return result[0] == 1
        except Exception as e:
            logger.error(f"连接测试失败: {e}")
            return False
    
    @staticmethod
    def get_connection_info(connection: pymysql.Connection) -> Dict[str, Any]:
        """获取连接信息"""
        return {
            'host': connection.host,
            'port': connection.port,
            'user': connection.user,
            'database': connection.db,
            'charset': connection.charset,
            'autocommit': connection.autocommit,
            'server_version': connection.get_server_info(),
            'protocol_version': connection.get_proto_info(),
            'thread_id': connection.thread_id()
        }
```

### 2.2 连接池实现
```python
from dbutils.pooled_db import PooledDB
import threading
from contextlib import contextmanager
from typing import Generator

class MySQLConnectionPool:
    """MySQL连接池管理器"""
    
    def __init__(
        self,
        host: str = 'localhost',
        port: int = 3306,
        user: str = 'root',
        password: str = '',
        database: str = None,
        charset: str = 'utf8mb4',
        mincached: int = 1,
        maxcached: int = 10,
        maxshared: int = 20,
        maxconnections: int = 30,
        blocking: bool = True,
        maxusage: int = 1000,
        setsession: list = None,
        reset: bool = True,
        failures: Any = None,
        ping: int = 1,
        **kwargs
    ):
        """
        初始化连接池
        
        Args:
            host: 数据库主机
            port: 端口
            user: 用户名
            password: 密码
            database: 数据库名
            charset: 字符集
            mincached: 最小空闲连接数
            maxcached: 最大空闲连接数
            maxshared: 最大共享连接数
            maxconnections: 最大连接数
            blocking: 是否阻塞等待连接
            maxusage: 连接最大使用次数
            setsession: 会话设置SQL列表
            reset: 是否重置连接
            failures: 错误重试处理
            ping: 连接检查频率
            **kwargs: 其他参数
        """
        self.pool = PooledDB(
            creator=pymysql,
            mincached=mincached,
            maxcached=maxcached,
            maxshared=maxshared,
            maxconnections=maxconnections,
            blocking=blocking,
            maxusage=maxusage,
            setsession=setsession,
            reset=reset,
            failures=failures,
            ping=ping,
            host=host,
            port=port,
            user=user,
            password=password,
            database=database,
            charset=charset,
            cursorclass=pymysql.cursors.DictCursor,
            **kwargs
        )
        
        self._stats = {
            'total_connections': 0,
            'active_connections': 0,
            'idle_connections': 0,
            'waiting_requests': 0
        }
        self._lock = threading.Lock()
        
        logger.info(f"连接池初始化完成: {host}:{port}/{database}")
    
    @contextmanager
    def get_connection(self) -> Generator[pymysql.Connection, None, None]:
        """
        从连接池获取连接（上下文管理器）
        
        Yields:
            数据库连接
        """
        connection = None
        try:
            with self._lock:
                self._stats['waiting_requests'] += 1
            
            connection = self.pool.connection()
            
            with self._lock:
                self._stats['total_connections'] += 1
                self._stats['active_connections'] += 1
                self._stats['waiting_requests'] -= 1
            
            yield connection
            
        except pymysql.Error as e:
            logger.error(f"获取连接失败: {e}")
            raise
        finally:
            if connection:
                connection.close()
                with self._lock:
                    self._stats['active_connections'] -= 1
                    self._stats['idle_connections'] += 1
    
    def get_pool_stats(self) -> Dict[str, Any]:
        """获取连接池统计信息"""
        with self._lock:
            return self._stats.copy()
    
    def close_all(self):
        """关闭所有连接"""
        self.pool.close()
        logger.info("连接池已关闭")
```



### 2.3 配置文件管理

```python
import yaml
import os
from pathlib import Path
from dataclasses import dataclass
from typing import Optional

@dataclass
class DatabaseConfig:
    """数据库配置数据类"""
    
    host: str = 'localhost'
    port: int = 3306
    user: str = 'root'
    password: str = ''
    database: str = ''
    charset: str = 'utf8mb4'
    autocommit: bool = False
    connect_timeout: int = 10
    read_timeout: int = 30
    write_timeout: int = 30
    
    # SSL配置
    ssl_enabled: bool = False
    ssl_ca: Optional[str] = None
    ssl_cert: Optional[str] = None
    ssl_key: Optional[str] = None
    
    # 连接池配置
    pool_mincached: int = 1
    pool_maxcached: int = 10
    pool_maxconnections: int = 30
    
    @classmethod
    def from_yaml(cls, config_file: str) -> 'DatabaseConfig':
        """从YAML文件加载配置"""
        try:
            with open(config_file, 'r', encoding='utf-8') as f:
                config_data = yaml.safe_load(f)
            
            # 支持环境变量覆盖
            env_mapping = {
                'DB_HOST': 'host',
                'DB_PORT': 'port',
                'DB_USER': 'user',
                'DB_PASSWORD': 'password',
                'DB_NAME': 'database'
            }
            
            for env_var, config_key in env_mapping.items():
                if env_var in os.environ:
                    config_data[config_key] = os.environ[env_var]
            
            return cls(**config_data)
        
        except FileNotFoundError:
            logger.warning(f"配置文件不存在: {config_file}")
            return cls()
        except Exception as e:
            logger.error(f"加载配置失败: {e}")
            raise
    
    def to_dict(self) -> Dict[str, Any]:
        """转换为字典"""
        return {
            'host': self.host,
            'port': self.port,
            'user': self.user,
            'password': self.password,
            'database': self.database,
            'charset': self.charset,
            'autocommit': self.autocommit,
            'connect_timeout': self.connect_timeout,
            'read_timeout': self.read_timeout,
            'write_timeout': self.write_timeout,
            'ssl': {
                'enabled': self.ssl_enabled,
                'ca': self.ssl_ca,
                'cert': self.ssl_cert,
                'key': self.ssl_key
            } if self.ssl_enabled else None
        }

```



### 2.4 配置文件示例 (database_config.yaml)

```yaml
# 配置文件示例 (database_config.yaml)
# 数据库连接配置
host: "localhost"
port: 3306
user: "app_user"
password: "secure_password_123"
database: "app_database"
charset: "utf8mb4"
autocommit: false

# 超时设置
connect_timeout: 10
read_timeout: 30
write_timeout: 30

# SSL配置
ssl_enabled: true
ssl_ca: "/path/to/ca.pem"
ssl_cert: "/path/to/client-cert.pem"
ssl_key: "/path/to/client-key.pem"

# 连接池配置
pool_mincached: 2
pool_maxcached: 10
pool_maxconnections: 50
```







---

## 三、游标与查询操作

### 3.1 游标类型
```python
import pymysql.cursors

class CursorManager:
    """游标管理器"""
    
    @staticmethod
    def create_cursor(connection: pymysql.Connection, cursor_type: str = 'dict'):
        """
        创建游标
        
        Args:
            connection: 数据库连接
            cursor_type: 游标类型 ('dict', 'tuple', 'ssdict', 'sscursor', 'sdict')
            
        Returns:
            游标对象
        """
        cursor_map = {
            'dict': pymysql.cursors.DictCursor,          # 字典游标
            'tuple': pymysql.cursors.Cursor,             # 元组游标（默认）
            'ssdict': pymysql.cursors.SSDictCursor,      # 无缓冲字典游标
            'sscursor': pymysql.cursors.SSCursor,        # 无缓冲元组游标
            'sdict': pymysql.cursors.SSDictCursor        # 同ssdict
        }
        
        cursor_class = cursor_map.get(cursor_type.lower(), pymysql.cursors.DictCursor)
        return connection.cursor(cursor=cursor_class)
    
    @staticmethod
    def explain_query(cursor: pymysql.cursors.Cursor, sql: str, params: tuple = None):
        """
        解释查询执行计划
        
        Args:
            cursor: 游标
            sql: SQL查询语句
            params: 参数
            
        Returns:
            执行计划
        """
        explain_sql = f"EXPLAIN {sql}"
        cursor.execute(explain_sql, params)
        return cursor.fetchall()
```



### 3.2 查询操作基础

```python
class QueryExecutor:
    """查询执行器"""
    
    def __init__(self, connection: pymysql.Connection):
        self.connection = connection
    
    def execute_query(
        self,
        sql: str,
        params: tuple = None,
        fetch_all: bool = True,
        cursor_type: str = 'dict'
    ) -> List[Dict]:
        """
        执行查询
        
        Args:
            sql: SQL查询语句
            params: 参数
            fetch_all: 是否获取所有结果
            cursor_type: 游标类型
            
        Returns:
            查询结果列表
        """
        cursor = CursorManager.create_cursor(self.connection, cursor_type)
        
        try:
            start_time = datetime.now()
            cursor.execute(sql, params)
            
            if fetch_all:
                result = cursor.fetchall()
            else:
                result = cursor.fetchone()
            
            elapsed_time = (datetime.now() - start_time).total_seconds()
            
            logger.debug(f"查询执行时间: {elapsed_time:.3f}s, SQL: {sql[:100]}...")
            
            return result
            
        except pymysql.Error as e:
            logger.error(f"查询执行失败: {e}\nSQL: {sql}")
            raise
        finally:
            cursor.close()
    
    def execute_many(self, sql: str, params_list: List[tuple]) -> int:
        """
        批量执行
        
        Args:
            sql: SQL语句
            params_list: 参数列表
            
        Returns:
            影响的行数
        """
        cursor = self.connection.cursor()
        
        try:
            start_time = datetime.now()
            affected_rows = cursor.executemany(sql, params_list)
            self.connection.commit()
            
            elapsed_time = (datetime.now() - start_time).total_seconds()
            logger.info(f"批量执行完成: {len(params_list)}条, 耗时: {elapsed_time:.3f}s")
            
            return affected_rows
            
        except pymysql.Error as e:
            self.connection.rollback()
            logger.error(f"批量执行失败: {e}")
            raise
        finally:
            cursor.close()
    
    def paginate_query(
        self,
        sql: str,
        params: tuple = None,
        page: int = 1,
        per_page: int = 20,
        cursor_type: str = 'dict'
    ) -> Dict[str, Any]:
        """
        分页查询
        
        Args:
            sql: SQL查询语句
            params: 参数
            page: 页码（从1开始）
            per_page: 每页数量
            cursor_type: 游标类型
            
        Returns:
            分页结果
        """
        # 计算偏移量
        offset = (page - 1) * per_page
        
        # 构建分页SQL
        paginated_sql = f"{sql} LIMIT {per_page} OFFSET {offset}"
        
        # 执行分页查询
        data = self.execute_query(paginated_sql, params, cursor_type=cursor_type)
        
        # 获取总数
        count_sql = f"SELECT COUNT(*) as total FROM ({sql}) as subquery"
        total_result = self.execute_query(count_sql, params, cursor_type='tuple')
        total = total_result[0][0] if total_result else 0
        
        # 计算总页数
        total_pages = (total + per_page - 1) // per_page
        
        return {
            'data': data,
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': total,
                'total_pages': total_pages,
                'has_next': page < total_pages,
                'has_prev': page > 1
            }
        }
```



### 3.3 高级查询示例

```python
class AdvancedQueries:
    """高级查询示例"""
    
    def __init__(self, connection: pymysql.Connection):
        self.connection = connection
    
    def find_by_id(self, table: str, id_value: Any, id_column: str = 'id') -> Optional[Dict]:
        """按ID查询"""
        sql = f"SELECT * FROM {table} WHERE {id_column} = %s"
        cursor = self.connection.cursor(pymysql.cursors.DictCursor)
        
        try:
            cursor.execute(sql, (id_value,))
            return cursor.fetchone()
        finally:
            cursor.close()
    
    def find_where(
        self,
        table: str,
        conditions: Dict[str, Any],
        operators: Dict[str, str] = None,
        order_by: str = None,
        limit: int = None
    ) -> List[Dict]:
        """
        条件查询
        
        Args:
            table: 表名
            conditions: 条件字典
            operators: 操作符字典
            order_by: 排序字段
            limit: 限制数量
            
        Returns:
            查询结果
        """
        if not conditions:
            sql = f"SELECT * FROM {table}"
            params = ()
        else:
            where_clauses = []
            params = []
            
            default_operators = {'=': '=', '>': '>', '<': '<', 'LIKE': 'LIKE', 'IN': 'IN'}
            operators = operators or {}
            
            for column, value in conditions.items():
                op = operators.get(column, '=')
                
                if op.upper() == 'IN':
                    if isinstance(value, (list, tuple)):
                        placeholders = ', '.join(['%s'] * len(value))
                        where_clauses.append(f"{column} IN ({placeholders})")
                        params.extend(value)
                    else:
                        where_clauses.append(f"{column} = %s")
                        params.append(value)
                elif op.upper() == 'BETWEEN':
                    if isinstance(value, (list, tuple)) and len(value) == 2:
                        where_clauses.append(f"{column} BETWEEN %s AND %s")
                        params.extend(value)
                    else:
                        raise ValueError("BETWEEN操作符需要两个值")
                else:
                    where_clauses.append(f"{column} {op} %s")
                    params.append(value)
            
            where_sql = " AND ".join(where_clauses)
            sql = f"SELECT * FROM {table} WHERE {where_sql}"
        
        # 添加排序
        if order_by:
            sql += f" ORDER BY {order_by}"
        
        # 添加限制
        if limit:
            sql += f" LIMIT {limit}"
        
        return self.execute_query(sql, tuple(params))
    
    def execute_raw_sql(self, sql: str, params: tuple = None) -> Any:
        """
        执行原始SQL
        
        Args:
            sql: SQL语句
            params: 参数
            
        Returns:
            执行结果
        """
        cursor = self.connection.cursor(pymysql.cursors.DictCursor)
        
        try:
            cursor.execute(sql, params)
            
            # 判断是否为查询语句
            if sql.strip().upper().startswith('SELECT'):
                result = cursor.fetchall()
            else:
                self.connection.commit()
                result = cursor.rowcount
            
            return result
            
        except pymysql.Error as e:
            self.connection.rollback()
            logger.error(f"原始SQL执行失败: {e}")
            raise
        finally:
            cursor.close()
```

---



## 四、数据操作（CRUD）

### 4.1 基础CRUD操作
```python
class CRUDOperations:
    """CRUD操作类"""
    
    def __init__(self, connection: pymysql.Connection):
        self.connection = connection
    
    def insert(self, table: str, data: Dict[str, Any]) -> int:
        """
        插入单条记录
        
        Args:
            table: 表名
            data: 数据字典
            
        Returns:
            插入的记录ID
        """
        if not data:
            raise ValueError("插入数据不能为空")
        
        columns = ', '.join(data.keys())
        placeholders = ', '.join(['%s'] * len(data))
        values = tuple(data.values())
        
        sql = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
        
        cursor = self.connection.cursor()
        
        try:
            cursor.execute(sql, values)
            self.connection.commit()
            
            # 返回自增ID
            if cursor.lastrowid:
                return cursor.lastrowid
            else:
                return cursor.rowcount
                
        except pymysql.Error as e:
            self.connection.rollback()
            logger.error(f"插入失败: {e}")
            raise
        finally:
            cursor.close()
    
    def bulk_insert(self, table: str, data_list: List[Dict[str, Any]]) -> int:
        """
        批量插入
        
        Args:
            table: 表名
            data_list: 数据字典列表
            
        Returns:
            影响的行数
        """
        if not data_list:
            return 0
        
        # 确保所有字典有相同的键
        first_dict = data_list[0]
        columns = ', '.join(first_dict.keys())
        placeholders = ', '.join(['%s'] * len(first_dict))
        
        # 构建值列表
        values = []
        for data in data_list:
            if set(data.keys()) != set(first_dict.keys()):
                raise ValueError("批量插入的数据字典必须有相同的键")
            values.append(tuple(data.values()))
        
        sql = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
        
        cursor = self.connection.cursor()
        
        try:
            affected_rows = cursor.executemany(sql, values)
            self.connection.commit()
            
            logger.info(f"批量插入完成: {affected_rows} 行")
            return affected_rows
            
        except pymysql.Error as e:
            self.connection.rollback()
            logger.error(f"批量插入失败: {e}")
            raise
        finally:
            cursor.close()
    
    def update(
        self,
        table: str,
        data: Dict[str, Any],
        where_conditions: Dict[str, Any]
    ) -> int:
        """
        更新记录
        
        Args:
            table: 表名
            data: 要更新的数据
            where_conditions: 更新条件
            
        Returns:
            影响的行数
        """
        if not data:
            raise ValueError("更新数据不能为空")
        
        if not where_conditions:
            raise ValueError("更新条件不能为空")
        
        # 构建SET子句
        set_clauses = []
        values = []
        
        for column, value in data.items():
            set_clauses.append(f"{column} = %s")
            values.append(value)
        
        # 构建WHERE子句
        where_clauses = []
        for column, value in where_conditions.items():
            where_clauses.append(f"{column} = %s")
            values.append(value)
        
        set_sql = ', '.join(set_clauses)
        where_sql = ' AND '.join(where_clauses)
        
        sql = f"UPDATE {table} SET {set_sql} WHERE {where_sql}"
        
        cursor = self.connection.cursor()
        
        try:
            cursor.execute(sql, tuple(values))
            self.connection.commit()
            
            affected_rows = cursor.rowcount
            logger.debug(f"更新完成: {affected_rows} 行")
            
            return affected_rows
            
        except pymysql.Error as e:
            self.connection.rollback()
            logger.error(f"更新失败: {e}")
            raise
        finally:
            cursor.close()
    
    def delete(self, table: str, where_conditions: Dict[str, Any]) -> int:
        """
        删除记录
        
        Args:
            table: 表名
            where_conditions: 删除条件
            
        Returns:
            影响的行数
        """
        if not where_conditions:
            raise ValueError("删除条件不能为空")
        
        # 构建WHERE子句
        where_clauses = []
        values = []
        
        for column, value in where_conditions.items():
            where_clauses.append(f"{column} = %s")
            values.append(value)
        
        where_sql = ' AND '.join(where_clauses)
        sql = f"DELETE FROM {table} WHERE {where_sql}"
        
        cursor = self.connection.cursor()
        
        try:
            cursor.execute(sql, tuple(values))
            self.connection.commit()
            
            affected_rows = cursor.rowcount
            logger.debug(f"删除完成: {affected_rows} 行")
            
            return affected_rows
            
        except pymysql.Error as e:
            self.connection.rollback()
            logger.error(f"删除失败: {e}")
            raise
        finally:
            cursor.close()
    
    def upsert(
        self,
        table: str,
        data: Dict[str, Any],
        unique_columns: List[str]
    ) -> Tuple[bool, int]:
        """
        UPSERT操作（存在则更新，不存在则插入）
        
        Args:
            table: 表名
            data: 数据
            unique_columns: 唯一约束列
            
        Returns:
            (是否新建, 记录ID)
        """
        # 尝试查找现有记录
        where_conditions = {col: data[col] for col in unique_columns if col in data}
        
        existing = self.find_where(table, where_conditions)
        
        if existing:
            # 更新现有记录
            row_id = existing[0]['id'] if 'id' in existing[0] else None
            self.update(table, data, where_conditions)
            return False, row_id
        else:
            # 插入新记录
            new_id = self.insert(table, data)
            return True, new_id
```



### 4.2 事务处理

```python
class TransactionManager:
    """事务管理器"""
    
    def __init__(self, connection: pymysql.Connection):
        self.connection = connection
    
    @contextmanager
    def transaction(self, isolation_level: str = None):
        """
        事务上下文管理器
        
        Args:
            isolation_level: 隔离级别 (READ UNCOMMITTED, READ COMMITTED, 
                                 REPEATABLE READ, SERIALIZABLE)
        """
        cursor = None
        try:
            # 设置隔离级别
            if isolation_level:
                self.connection.begin()
                cursor = self.connection.cursor()
                cursor.execute(f"SET TRANSACTION ISOLATION LEVEL {isolation_level}")
            
            yield
            
            # 提交事务
            self.connection.commit()
            logger.debug("事务提交成功")
            
        except Exception as e:
            # 回滚事务
            self.connection.rollback()
            logger.error(f"事务回滚: {e}")
            raise
        finally:
            if cursor:
                cursor.close()
    
    def execute_in_transaction(self, operations: List[callable]) -> bool:
        """
        在事务中执行多个操作
        
        Args:
            operations: 操作函数列表
            
        Returns:
            是否全部成功
        """
        try:
            self.connection.begin()
            
            for operation in operations:
                operation()
            
            self.connection.commit()
            return True
            
        except Exception as e:
            self.connection.rollback()
            logger.error(f"事务执行失败: {e}")
            return False
    
    def savepoint_example(self):
        """保存点示例"""
        cursor = self.connection.cursor()
        
        try:
            # 开始事务
            self.connection.begin()
            
            # 操作1
            cursor.execute("INSERT INTO users (name) VALUES ('Alice')")
            
            # 创建保存点
            cursor.execute("SAVEPOINT sp1")
            
            try:
                # 操作2（可能失败）
                cursor.execute("INSERT INTO users (name) VALUES (NULL)")
            except pymysql.Error:
                # 回滚到保存点
                cursor.execute("ROLLBACK TO SAVEPOINT sp1")
                logger.warning("回滚到保存点 sp1")
            
            # 操作3
            cursor.execute("INSERT INTO users (name) VALUES ('Bob')")
            
            # 提交事务
            self.connection.commit()
            
        except Exception as e:
            self.connection.rollback()
            logger.error(f"事务失败: {e}")
            raise
        finally:
            cursor.close()
```

---



## 五、数据类型处理

### 5.1 类型转换
```python
class DataTypeHandler:
    """数据类型处理器"""
    
    @staticmethod
    def python_to_mysql(value: Any) -> Any:
        """
        Python类型转换为MySQL类型
        
        Args:
            value: Python值
            
        Returns:
            MySQL兼容的值
        """
        if value is None:
            return None
        elif isinstance(value, datetime):
            return value.strftime('%Y-%m-%d %H:%M:%S')
        elif isinstance(value, date):
            return value.strftime('%Y-%m-%d')
        elif isinstance(value, bool):
            return 1 if value else 0
        elif isinstance(value, (list, tuple, dict)):
            return json.dumps(value, ensure_ascii=False)
        elif isinstance(value, Decimal):
            return float(value)
        elif isinstance(value, bytes):
            return value.decode('utf-8', errors='ignore')
        else:
            return value
    
    @staticmethod
    def mysql_to_python(value: Any, column_type: str = None) -> Any:
        """
        MySQL类型转换为Python类型
        
        Args:
            value: MySQL值
            column_type: 列类型
            
        Returns:
            Python值
        """
        if value is None:
            return None
        
        if column_type:
            column_type = column_type.upper()
            
            # 处理JSON类型
            if 'JSON' in column_type and isinstance(value, str):
                try:
                    return json.loads(value)
                except json.JSONDecodeError:
                    return value
            
            # 处理SET类型
            if 'SET' in column_type and isinstance(value, str):
                return set(value.split(','))
            
            # 处理BIT类型
            if 'BIT' in column_type and isinstance(value, (bytes, bytearray)):
                return int.from_bytes(value, 'big')
        
        # 自动类型推断
        if isinstance(value, datetime):
            return value
        elif isinstance(value, date):
            return value
        elif isinstance(value, timedelta):
            return value
        elif isinstance(value, (bytes, bytearray)):
            try:
                return value.decode('utf-8')
            except UnicodeDecodeError:
                return value
        else:
            return value
    
    @staticmethod
    def sanitize_input(value: Any, max_length: int = None) -> Any:
        """
        清理输入数据
        
        Args:
            value: 输入值
            max_length: 最大长度限制
            
        Returns:
            清理后的值
        """
        if isinstance(value, str):
            # 移除首尾空格
            value = value.strip()
            
            # 限制长度
            if max_length and len(value) > max_length:
                value = value[:max_length]
            
            # 转义特殊字符（pymysql会自动处理，这里只是额外保护）
            value = value.replace('\\', '\\\\').replace("'", "\\'")
        
        elif isinstance(value, (list, dict)):
            # 递归处理嵌套结构
            if isinstance(value, list):
                return [DataTypeHandler.sanitize_input(item) for item in value]
            elif isinstance(value, dict):
                return {k: DataTypeHandler.sanitize_input(v) for k, v in value.items()}
        
        return value
```



### 5.2 二进制数据处理

```python
class BinaryDataHandler:
    """二进制数据处理器"""
    
    @staticmethod
    def insert_blob(connection: pymysql.Connection, table: str, column: str, 
                   id_value: Any, blob_data: bytes, id_column: str = 'id'):
        """插入BLOB数据"""
        sql = f"UPDATE {table} SET {column} = %s WHERE {id_column} = %s"
        
        cursor = connection.cursor()
        try:
            cursor.execute(sql, (blob_data, id_value))
            connection.commit()
        finally:
            cursor.close()
    
    @staticmethod
    def read_blob(connection: pymysql.Connection, table: str, column: str,
                 id_value: Any, id_column: str = 'id') -> bytes:
        """读取BLOB数据"""
        sql = f"SELECT {column} FROM {table} WHERE {id_column} = %s"
        
        cursor = connection.cursor()
        try:
            cursor.execute(sql, (id_value,))
            result = cursor.fetchone()
            return result[0] if result else None
        finally:
            cursor.close()
    
    @staticmethod
    def stream_blob(connection: pymysql.Connection, table: str, column: str,
                   id_value: Any, chunk_size: int = 8192, 
                   id_column: str = 'id') -> Generator[bytes, None, None]:
        """流式读取BLOB数据"""
        sql = f"SELECT {column} FROM {table} WHERE {id_column} = %s"
        
        cursor = connection.cursor()
        try:
            cursor.execute(sql, (id_value,))
            result = cursor.fetchone()
            
            if result and result[0]:
                blob_data = result[0]
                for i in range(0, len(blob_data), chunk_size):
                    yield blob_data[i:i + chunk_size]
        finally:
            cursor.close()
```

---



## 六、错误处理与重试

### 6.1 错误处理策略
```python
import time
from functools import wraps
from typing import Type, Callable

class MySQLErrorHandler:
    """MySQL错误处理器"""
    
    ERROR_CODES = {
        1045: "访问被拒绝，错误的用户名或密码",
        1049: "未知数据库",
        1054: "未知列",
        1062: "重复键错误",
        1146: "表不存在",
        1213: "死锁",
        1216: "外键约束失败",
        2002: "无法连接到MySQL服务器",
        2003: "无法连接到MySQL服务器",
        2006: "MySQL服务器已断开连接",
        2013: "查询期间丢失连接",
        1205: "锁等待超时",
        1217: "无法删除或更新父行",
    }
    
    @classmethod
    def get_error_message(cls, error_code: int, default_msg: str = None) -> str:
        """获取错误信息"""
        return cls.ERROR_CODES.get(error_code, default_msg or f"MySQL错误代码: {error_code}")
    
    @classmethod
    def is_connection_error(cls, error: pymysql.Error) -> bool:
        """判断是否为连接错误"""
        error_code = getattr(error, 'args', [0])[0]
        return error_code in [2002, 2003, 2006, 2013]
    
    @classmethod
    def is_deadlock_error(cls, error: pymysql.Error) -> bool:
        """判断是否为死锁错误"""
        error_code = getattr(error, 'args', [0])[0]
        return error_code in [1213, 1205]
    
    @classmethod
    def is_duplicate_key_error(cls, error: pymysql.Error) -> bool:
        """判断是否为重复键错误"""
        error_code = getattr(error, 'args', [0])[0]
        return error_code == 1062
    
    @staticmethod
    def retry_on_error(
        max_retries: int = 3,
        retry_delay: float = 1.0,
        exponential_backoff: bool = True,
        retry_on: List[Type[Exception]] = None
    ):
        """
        错误重试装饰器
        
        Args:
            max_retries: 最大重试次数
            retry_delay: 重试延迟（秒）
            exponential_backoff: 是否使用指数退避
            retry_on: 需要重试的异常类型列表
        """
        if retry_on is None:
            retry_on = [pymysql.OperationalError, pymysql.InterfaceError]
        
        def decorator(func: Callable):
            @wraps(func)
            def wrapper(*args, **kwargs):
                last_exception = None
                
                for attempt in range(max_retries + 1):
                    try:
                        return func(*args, **kwargs)
                    
                    except tuple(retry_on) as e:
                        last_exception = e
                        
                        # 检查是否应该重试
                        if attempt < max_retries:
                            # 计算等待时间
                            if exponential_backoff:
                                wait_time = retry_delay * (2 ** attempt)
                            else:
                                wait_time = retry_delay
                            
                            logger.warning(
                                f"操作失败，{wait_time:.1f}秒后重试 ({attempt+1}/{max_retries}): {e}"
                            )
                            time.sleep(wait_time)
                        else:
                            logger.error(f"操作失败，已达到最大重试次数: {e}")
                            raise
                    
                    except Exception as e:
                        # 其他异常直接抛出
                        logger.error(f"操作失败: {e}")
                        raise
                
                # 如果所有重试都失败了，抛出最后一个异常
                raise last_exception
            
            return wrapper
        return decorator
    
    @classmethod
    def handle_duplicate_key(cls, error: pymysql.Error, table: str, 
                           duplicate_value: Any, action: str = 'insert') -> Dict:
        """
        处理重复键错误
        
        Args:
            error: 异常对象
            table: 表名
            duplicate_value: 重复的值
            action: 操作类型
            
        Returns:
            处理结果
        """
        error_msg = str(error)
        
        # 提取重复的键名
        key_name = 'unknown'
        if "key '" in error_msg:
            # 从错误消息中提取键名
            start = error_msg.find("key '") + 5
            end = error_msg.find("'", start)
            key_name = error_msg[start:end]
        
        logger.warning(
            f"{action.capitalize()}失败: 表 '{table}' 的键 '{key_name}' 重复, "
            f"值: {duplicate_value}"
        )
        
        return {
            'success': False,
            'error': 'duplicate_key',
            'key': key_name,
            'value': duplicate_value,
            'message': f"重复的键值: {duplicate_value}"
        }
```



### 6.2 连接健康检查

```python
class ConnectionHealthChecker:
    """连接健康检查器"""
    
    def __init__(self, connection: pymysql.Connection, check_interval: int = 300):
        """
        初始化健康检查器
        
        Args:
            connection: 数据库连接
            check_interval: 检查间隔（秒）
        """
        self.connection = connection
        self.check_interval = check_interval
        self.last_check = 0
        self.is_healthy = True
        self.failure_count = 0
    
    def check_health(self) -> bool:
        """
        检查连接健康状态
        
        Returns:
            是否健康
        """
        current_time = time.time()
        
        # 检查是否达到检查间隔
        if current_time - self.last_check < self.check_interval:
            return self.is_healthy
        
        self.last_check = current_time
        
        try:
            cursor = self.connection.cursor()
            cursor.execute("SELECT 1")
            result = cursor.fetchone()
            cursor.close()
            
            if result and result[0] == 1:
                self.is_healthy = True
                self.failure_count = 0
                return True
            else:
                self.is_healthy = False
                self.failure_count += 1
                return False
                
        except Exception as e:
            self.is_healthy = False
            self.failure_count += 1
            logger.warning(f"连接健康检查失败: {e}")
            return False
    
    def reconnect_if_needed(self) -> bool:
        """
        如果需要则重新连接
        
        Returns:
            是否成功重新连接
        """
        if not self.check_health() and self.failure_count >= 3:
            try:
                # 尝试ping重新连接
                self.connection.ping(reconnect=True)
                self.is_healthy = True
                self.failure_count = 0
                logger.info("连接已恢复")
                return True
            except Exception as e:
                logger.error(f"重新连接失败: {e}")
                return False
        
        return self.is_healthy
    
    def get_health_status(self) -> Dict[str, Any]:
        """获取健康状态"""
        return {
            'is_healthy': self.is_healthy,
            'failure_count': self.failure_count,
            'last_check': self.last_check,
            'connection_info': {
                'host': self.connection.host,
                'database': self.connection.db,
                'server_version': self.connection.get_server_info()
            }
        }
```

---



## 七、性能优化与监控

### 7.1 查询性能优化
```python
class QueryOptimizer:
    """查询优化器"""
    
    def __init__(self, connection: pymysql.Connection):
        self.connection = connection
    
    def analyze_query_performance(self, sql: str, params: tuple = None) -> Dict:
        """
        分析查询性能
        
        Args:
            sql: SQL查询
            params: 参数
            
        Returns:
            性能分析结果
        """
        cursor = self.connection.cursor(pymysql.cursors.DictCursor)
        
        try:
            # 启用性能分析
            cursor.execute("SET profiling = 1")
            
            # 执行查询
            start_time = time.time()
            cursor.execute(sql, params)
            results = cursor.fetchall()
            execution_time = time.time() - start_time
            
            # 获取性能分析数据
            cursor.execute("SHOW PROFILES")
            profiles = cursor.fetchall()
            
            if profiles:
                profile_id = profiles[-1]['Query_ID']
                cursor.execute(f"SHOW PROFILE FOR QUERY {profile_id}")
                profile_details = cursor.fetchall()
            else:
                profile_details = []
            
            # 获取执行计划
            cursor.execute(f"EXPLAIN {sql}", params)
            explain_plan = cursor.fetchall()
            
            # 禁用性能分析
            cursor.execute("SET profiling = 0")
            
            return {
                'execution_time': execution_time,
                'row_count': len(results),
                'explain_plan': explain_plan,
                'profile_details': profile_details,
                'query_hash': hash(sql + str(params))
            }
            
        finally:
            cursor.close()
    
    def optimize_query(self, sql: str) -> str:
        """
        优化查询语句
        
        Args:
            sql: 原始SQL
            
        Returns:
            优化后的SQL
        """
        # 简单的查询优化规则
        optimizations = [
            # 移除不必要的空格
            (r'\s+', ' '),
            # 用INNER JOIN替换WHERE连接
            (r'FROM\s+(\w+)\s*,\s*(\w+)\s+WHERE\s+\1\.(\w+)\s*=\s*\2\.(\w+)',
             r'FROM \1 INNER JOIN \2 ON \1.\3 = \2.\4'),
            # 用EXISTS替换IN子查询
            (r'WHERE\s+(\w+)\s+IN\s*$\s*SELECT', r'WHERE EXISTS (SELECT'),
        ]
        
        optimized_sql = sql
        
        for pattern, replacement in optimizations:
            import re
            optimized_sql = re.sub(pattern, replacement, optimized_sql, flags=re.IGNORECASE)
        
        return optimized_sql.strip()
    
    def create_index_if_not_exists(self, table: str, columns: List[str], 
                                 index_name: str = None, index_type: str = 'INDEX'):
        """
        创建索引（如果不存在）
        
        Args:
            table: 表名
            columns: 列名列表
            index_name: 索引名称
            index_type: 索引类型 (INDEX, UNIQUE, FULLTEXT, SPATIAL)
        """
        if not index_name:
            index_name = f"idx_{table}_{'_'.join(columns)}"
        
        columns_str = ', '.join(columns)
        
        # 检查索引是否已存在
        check_sql = """
        SELECT COUNT(*) as index_exists 
        FROM information_schema.STATISTICS 
        WHERE table_schema = DATABASE() 
          AND table_name = %s 
          AND index_name = %s
        """
        
        cursor = self.connection.cursor()
        
        try:
            cursor.execute(check_sql, (table, index_name))
            result = cursor.fetchone()
            
            if result[0] == 0:
                # 创建索引
                create_sql = f"CREATE {index_type} {index_name} ON {table} ({columns_str})"
                cursor.execute(create_sql)
                self.connection.commit()
                logger.info(f"创建索引: {index_name} 在表 {table}({columns_str})")
            else:
                logger.debug(f"索引已存在: {index_name}")
                
        finally:
            cursor.close()
```



### 7.2 监控与统计

```python
import threading
from collections import defaultdict
from datetime import datetime, timedelta

class DatabaseMonitor:
    """数据库监控器"""
    
    def __init__(self, connection: pymysql.Connection):
        self.connection = connection
        self.stats = {
            'queries': defaultdict(list),
            'errors': defaultdict(int),
            'connections': 0,
            'slow_queries': []
        }
        self.lock = threading.Lock()
        self.slow_query_threshold = 1.0  # 秒
        self.start_time = datetime.now()
    
    def record_query(self, sql: str, execution_time: float, success: bool = True):
        """记录查询"""
        with self.lock:
            # 记录查询
            query_key = sql.split()[0].upper() if sql else 'UNKNOWN'
            self.stats['queries'][query_key].append({
                'sql': sql[:100] + '...' if len(sql) > 100 else sql,
                'time': execution_time,
                'timestamp': datetime.now(),
                'success': success
            })
            
            # 记录慢查询
            if execution_time > self.slow_query_threshold:
                self.stats['slow_queries'].append({
                    'sql': sql,
                    'time': execution_time,
                    'timestamp': datetime.now()
                })
    
    def record_error(self, error_type: str):
        """记录错误"""
        with self.lock:
            self.stats['errors'][error_type] += 1
    
    def get_performance_report(self, time_window: int = 3600) -> Dict:
        """
        获取性能报告
        
        Args:
            time_window: 时间窗口（秒）
            
        Returns:
            性能报告
        """
        with self.lock:
            cutoff_time = datetime.now() - timedelta(seconds=time_window)
            
            # 统计查询
            query_stats = {}
            for query_type, queries in self.stats['queries'].items():
                recent_queries = [
                    q for q in queries 
                    if q['timestamp'] > cutoff_time
                ]
                
                if recent_queries:
                    query_stats[query_type] = {
                        'count': len(recent_queries),
                        'avg_time': sum(q['time'] for q in recent_queries) / len(recent_queries),
                        'max_time': max(q['time'] for q in recent_queries),
                        'success_rate': sum(1 for q in recent_queries if q['success']) / len(recent_queries)
                    }
            
            # 统计慢查询
            slow_queries = [
                q for q in self.stats['slow_queries']
                if q['timestamp'] > cutoff_time
            ]
            
            return {
                'time_window_seconds': time_window,
                'uptime_seconds': (datetime.now() - self.start_time).total_seconds(),
                'query_statistics': query_stats,
                'error_statistics': dict(self.stats['errors']),
                'slow_queries_count': len(slow_queries),
                'slow_queries': slow_queries[:10],  # 只显示前10个
                'timestamp': datetime.now().isoformat()
            }
    
    def reset_stats(self):
        """重置统计"""
        with self.lock:
            self.stats = {
                'queries': defaultdict(list),
                'errors': defaultdict(int),
                'connections': 0,
                'slow_queries': []
            }
            self.start_time = datetime.now()
```

---



## 八、安全最佳实践

### 8.1 SQL注入防护
```python
class SecurityManager:
    """安全管理器"""
    
    @staticmethod
    def sanitize_sql_identifier(identifier: str) -> str:
        """
        清理SQL标识符（表名、列名等）
        
        Args:
            identifier: 标识符
            
        Returns:
            清理后的标识符
        """
        # 移除危险字符
        dangerous_chars = [';', '--', '/*', '*/', "'", '"', '`', '\\']
        for char in dangerous_chars:
            identifier = identifier.replace(char, '')
        
        # 限制长度
        if len(identifier) > 64:
            identifier = identifier[:64]
        
        return identifier
    
    @staticmethod
    def validate_input(value: Any, expected_type: type, max_length: int = None) -> bool:
        """
        验证输入数据
        
        Args:
            value: 输入值
            expected_type: 期望类型
            max_length: 最大长度
            
        Returns:
            是否有效
        """
        if not isinstance(value, expected_type):
            return False
        
        if expected_type == str and max_length:
            if len(value) > max_length:
                return False
            
            # 检查是否包含SQL注入特征
            sql_injection_patterns = [
                r'\b(SELECT|INSERT|UPDATE|DELETE|DROP|UNION)\b',
                r'--\s',
                r'/\*.*\*/',
                r';.*$'
            ]
            
            import re
            for pattern in sql_injection_patterns:
                if re.search(pattern, value, re.IGNORECASE):
                    return False
        
        return True
    
    @staticmethod
    def safe_execute(cursor: pymysql.cursors.Cursor, sql: str, params: tuple = None):
        """
        安全执行SQL
        
        Args:
            cursor: 游标
            sql: SQL语句
            params: 参数
            
        Raises:
            ValueError: 如果检测到潜在SQL注入
        """
        # 检查SQL语句
        if SecurityManager._contains_sql_injection(sql):
            raise ValueError("检测到潜在SQL注入风险")
        
        # 使用参数化查询
        cursor.execute(sql, params)
    
    @staticmethod
    def _contains_sql_injection(sql: str) -> bool:
        """
        检查是否包含SQL注入特征
        
        Args:
            sql: SQL语句
            
        Returns:
            是否包含注入特征
        """
        import re
        
        # 危险模式
        dangerous_patterns = [
            r'\b(SELECT|INSERT|UPDATE|DELETE|DROP|TRUNCATE)\b.*\b(FROM|INTO|TABLE)\b',
            r'--\s',
            r'/\*.*\*/',
            r';.*(SELECT|INSERT|UPDATE|DELETE|DROP)',
            r'UNION\s+ALL\s+SELECT',
            r'EXEC\s*$|EXECUTE\s*$|sp_|xp_',
            r'WAITFOR\s+DELAY',
            r'BENCHMARK\s*$'
        ]
        
        for pattern in dangerous_patterns:
            if re.search(pattern, sql, re.IGNORECASE):
                return True
        
        return False
```



### 8.2 敏感数据保护

```python
import hashlib
import base64
from cryptography.fernet import Fernet

class DataProtector:
    """数据保护器"""
    
    def __init__(self, encryption_key: str = None):
        """
        初始化数据保护器
        
        Args:
            encryption_key: 加密密钥，如果为None则生成新密钥
        """
        if encryption_key:
            self.key = base64.urlsafe_b64encode(
                hashlib.sha256(encryption_key.encode()).digest()
            )
        else:
            self.key = Fernet.generate_key()
        
        self.cipher = Fernet(self.key)
    
    def encrypt_sensitive_data(self, data: str) -> str:
        """
        加密敏感数据
        
        Args:
            data: 敏感数据
            
        Returns:
            加密后的数据
        """
        if not data:
            return data
        
        encrypted = self.cipher.encrypt(data.encode())
        return base64.urlsafe_b64encode(encrypted).decode()
    
    def decrypt_sensitive_data(self, encrypted_data: str) -> str:
        """
        解密敏感数据
        
        Args:
            encrypted_data: 加密数据
            
        Returns:
            解密后的数据
        """
        if not encrypted_data:
            return encrypted_data
        
        try:
            encrypted_bytes = base64.urlsafe_b64decode(encrypted_data.encode())
            decrypted = self.cipher.decrypt(encrypted_bytes)
            return decrypted.decode()
        except Exception as e:
            logger.error(f"数据解密失败: {e}")
            return encrypted_data
    
    @staticmethod
    def hash_password(password: str, salt: str = None) -> Dict[str, str]:
        """
        哈希密码
        
        Args:
            password: 密码
            salt: 盐值，如果为None则生成新盐
            
        Returns:
            包含哈希值和盐的字典
        """
        import secrets
        
        if salt is None:
            salt = secrets.token_hex(16)
        
        # 使用PBKDF2哈希
        hashed = hashlib.pbkdf2_hmac(
            'sha256',
            password.encode('utf-8'),
            salt.encode('utf-8'),
            100000  # 迭代次数
        )
        
        return {
            'hash': hashed.hex(),
            'salt': salt
        }
    
    @staticmethod
    def verify_password(password: str, stored_hash: str, salt: str) -> bool:
        """
        验证密码
        
        Args:
            password: 待验证密码
            stored_hash: 存储的哈希值
            salt: 盐值
            
        Returns:
            是否匹配
        """
        new_hash = DataProtector.hash_password(password, salt)['hash']
        return new_hash == stored_hash
```

---



## 九、异步操作（aiomysql）

### 9.1 异步连接与查询
```python
import asyncio
import aiomysql
from typing import Optional, List, Dict, Any

class AsyncMySQLManager:
    """异步MySQL管理器"""
    
    @staticmethod
    async def create_async_pool(
        host: str = 'localhost',
        port: int = 3306,
        user: str = 'root',
        password: str = '',
        db: str = None,
        charset: str = 'utf8mb4',
        minsize: int = 1,
        maxsize: int = 10,
        pool_recycle: int = 3600,
        **kwargs
    ) -> aiomysql.Pool:
        """
        创建异步连接池
        
        Args:
            host: 数据库主机
            port: 端口
            user: 用户名
            password: 密码
            db: 数据库名
            charset: 字符集
            minsize: 最小连接数
            maxsize: 最大连接数
            pool_recycle: 连接回收时间（秒）
            **kwargs: 其他参数
            
        Returns:
            异步连接池
        """
        try:
            pool = await aiomysql.create_pool(
                host=host,
                port=port,
                user=user,
                password=password,
                db=db,
                charset=charset,
                minsize=minsize,
                maxsize=maxsize,
                pool_recycle=pool_recycle,
                autocommit=False,
                **kwargs
            )
            
            logger.info(f"异步连接池创建成功: {host}:{port}/{db}")
            return pool
            
        except Exception as e:
            logger.error(f"创建异步连接池失败: {e}")
            raise
    
    @staticmethod
    async def execute_async_query(
        pool: aiomysql.Pool,
        sql: str,
        params: tuple = None,
        fetch_all: bool = True
    ) -> List[Dict]:
        """
        执行异步查询
        
        Args:
            pool: 连接池
            sql: SQL语句
            params: 参数
            fetch_all: 是否获取所有结果
            
        Returns:
            查询结果
        """
        async with pool.acquire() as conn:
            async with conn.cursor(aiomysql.DictCursor) as cursor:
                await cursor.execute(sql, params)
                
                if fetch_all:
                    result = await cursor.fetchall()
                else:
                    result = await cursor.fetchone()
                
                return result
    
    @staticmethod
    async def execute_async_transaction(
        pool: aiomysql.Pool,
        operations: List[callable]
    ) -> bool:
        """
        执行异步事务
        
        Args:
            pool: 连接池
            operations: 操作函数列表
            
        Returns:
            是否成功
        """
        async with pool.acquire() as conn:
            try:
                await conn.begin()
                
                for operation in operations:
                    await operation(conn)
                
                await conn.commit()
                return True
                
            except Exception as e:
                await conn.rollback()
                logger.error(f"异步事务失败: {e}")
                return False
    
    @staticmethod
    async def batch_async_insert(
        pool: aiomysql.Pool,
        table: str,
        data_list: List[Dict]
    ) -> int:
        """
        批量异步插入
        
        Args:
            pool: 连接池
            table: 表名
            data_list: 数据列表
            
        Returns:
            影响的行数
        """
        if not data_list:
            return 0
        
        columns = ', '.join(data_list[0].keys())
        placeholders = ', '.join(['%s'] * len(data_list[0]))
        
        sql = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
        
        # 准备数据
        values = [tuple(item.values()) for item in data_list]
        
        async with pool.acquire() as conn:
            async with conn.cursor() as cursor:
                affected_rows = await cursor.executemany(sql, values)
                await conn.commit()
                
                return affected_rows
```

---



## 十、完整示例与最佳实践

### 10.1 生产环境配置示例
```python
import os
from dotenv import load_dotenv

# 加载环境变量
load_dotenv()

class ProductionMySQLConfig:
    """生产环境MySQL配置"""
    
    @staticmethod
    def get_config() -> Dict[str, Any]:
        """获取生产环境配置"""
        return {
            'host': os.getenv('DB_HOST', 'localhost'),
            'port': int(os.getenv('DB_PORT', '3306')),
            'user': os.getenv('DB_USER', 'root'),
            'password': os.getenv('DB_PASSWORD', ''),
            'database': os.getenv('DB_NAME', 'production_db'),
            'charset': 'utf8mb4',
            'autocommit': False,
            'connect_timeout': 10,
            'read_timeout': 30,
            'write_timeout': 30,
            
            # SSL配置
            'ssl': {
                'ca': os.getenv('DB_SSL_CA'),
                'cert': os.getenv('DB_SSL_CERT'),
                'key': os.getenv('DB_SSL_KEY')
            } if os.getenv('DB_SSL_ENABLED') == 'true' else None,
            
            # 连接池配置
            'pool': {
                'mincached': int(os.getenv('DB_POOL_MIN', '2')),
                'maxcached': int(os.getenv('DB_POOL_MAX', '10')),
                'maxconnections': int(os.getenv('DB_POOL_MAX_CONN', '30')),
                'blocking': True,
                'maxusage': 1000,
                'ping': 1
            }
        }
    
    @staticmethod
    def create_production_connection():
        """创建生产环境连接"""
        config = ProductionMySQLConfig.get_config()
        
        # 使用连接池
        pool_config = config.pop('pool', {})
        
        return MySQLConnectionPool(
            **config,
            **pool_config
        )
```



### 10.2 完整的CRUD服务示例

```python
class UserService:
    """用户服务示例"""
    
    def __init__(self, connection_pool: MySQLConnectionPool):
        self.pool = connection_pool
    
    def create_user(self, user_data: Dict[str, Any]) -> Dict[str, Any]:
        """创建用户"""
        with self.pool.get_connection() as conn:
            crud = CRUDOperations(conn)
            
            # 验证数据
            required_fields = ['username', 'email', 'password_hash']
            for field in required_fields:
                if field not in user_data:
                    raise ValueError(f"缺少必需字段: {field}")
            
            # 创建用户
            user_id = crud.insert('users', user_data)
            
            # 记录日志
            logger.info(f"创建用户成功: ID={user_id}, 用户名={user_data['username']}")
            
            return {
                'id': user_id,
                'username': user_data['username'],
                'email': user_data['email'],
                'created_at': datetime.now()
            }
    
    def get_user_by_id(self, user_id: int) -> Optional[Dict]:
        """根据ID获取用户"""
        with self.pool.get_connection() as conn:
            crud = CRUDOperations(conn)
            return crud.find_by_id('users', user_id)
    
    def update_user(self, user_id: int, update_data: Dict[str, Any]) -> bool:
        """更新用户"""
        with self.pool.get_connection() as conn:
            crud = CRUDOperations(conn)
            
            # 不允许更新ID
            if 'id' in update_data:
                del update_data['id']
            
            # 执行更新
            affected_rows = crud.update(
                table='users',
                data=update_data,
                where_conditions={'id': user_id}
            )
            
            return affected_rows > 0
    
    def delete_user(self, user_id: int, soft_delete: bool = True) -> bool:
        """删除用户"""
        with self.pool.get_connection() as conn:
            if soft_delete:
                # 软删除：标记为删除状态
                update_data = {
                    'is_deleted': True,
                    'deleted_at': datetime.now()
                }
                
                crud = CRUDOperations(conn)
                affected_rows = crud.update(
                    table='users',
                    data=update_data,
                    where_conditions={'id': user_id}
                )
            else:
                # 硬删除：从数据库删除
                crud = CRUDOperations(conn)
                affected_rows = crud.delete(
                    table='users',
                    where_conditions={'id': user_id}
                )
            
            return affected_rows > 0
    
    def search_users(
        self,
        filters: Dict[str, Any],
        page: int = 1,
        per_page: int = 20,
        order_by: str = 'created_at DESC'
    ) -> Dict[str, Any]:
        """搜索用户"""
        with self.pool.get_connection() as conn:
            executor = QueryExecutor(conn)
            
            # 构建查询
            where_clauses = []
            params = []
            
            for key, value in filters.items():
                if value is not None:
                    if key == 'username' and value:
                        where_clauses.append("username LIKE %s")
                        params.append(f"%{value}%")
                    elif key == 'email' and value:
                        where_clauses.append("email LIKE %s")
                        params.append(f"%{value}%")
                    elif key == 'is_active':
                        where_clauses.append("is_active = %s")
                        params.append(value)
            
            where_sql = " AND ".join(where_clauses) if where_clauses else "1=1"
            sql = f"SELECT * FROM users WHERE {where_sql} ORDER BY {order_by}"
            
            # 执行分页查询
            return executor.paginate_query(sql, tuple(params), page, per_page)
```

---



## 十一、总结与最佳实践清单

### ✅ 最佳实践清单

#### 1. **连接管理**
- ✅ **使用连接池**：避免频繁创建连接
- ✅ **设置合理的超时**：connect_timeout, read_timeout, write_timeout
- ✅ **启用SSL连接**：生产环境必须启用
- ✅ **连接健康检查**：定期检查连接状态
- ✅ **错误重试机制**：实现指数退避重试

#### 2. **查询安全**
- ✅ **使用参数化查询**：防止SQL注入
- ✅ **验证输入数据**：类型、长度、格式检查
- ✅ **清理标识符**：表名、列名特殊处理
- ✅ **最小权限原则**：应用使用最小必要权限

#### 3. **性能优化**
- ✅ **使用索引**：合理创建和使用索引
- ✅ **批量操作**：使用executemany进行批量插入
- ✅ **游标选择**：根据场景选择合适游标类型
- ✅ **查询优化**：分析执行计划，优化慢查询
- ✅ **连接复用**：减少连接创建开销

#### 4. **错误处理**
- ✅ **异常分类处理**：区分连接错误、死锁、重复键等
- ✅ **事务回滚**：确保数据一致性
- ✅ **错误日志记录**：记录详细错误信息
- ✅ **优雅降级**：提供备选方案

#### 5. **数据类型**
- ✅ **类型转换**：正确处理Python与MySQL类型
- ✅ **日期时间处理**：统一时区，格式化输出
- ✅ **二进制数据处理**：使用BLOB类型处理
- ✅ **JSON数据**：使用JSON类型存储结构化数据

#### 6. **监控与维护**
- ✅ **性能监控**：记录查询性能，识别慢查询
- ✅ **连接统计**：监控连接池使用情况
- ✅ **错误统计**：统计错误类型和频率
- ✅ **定期维护**：检查索引、优化表结构

### ❌ 常见错误

#### 1. **SQL注入风险**
```python
# ❌ 错误：字符串拼接
username = "admin'; DROP TABLE users; --"
sql = f"SELECT * FROM users WHERE username = '{username}'"

# ✅ 正确：参数化查询
sql = "SELECT * FROM users WHERE username = %s"
cursor.execute(sql, (username,))
```

#### 2. **连接泄漏**
```python
# ❌ 错误：未关闭连接
def get_data():
    conn = pymysql.connect(...)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM table")
    return cursor.fetchall()
    # 连接未关闭！

# ✅ 正确：使用上下文管理器
def get_data():
    with pymysql.connect(...) as conn:
        with conn.cursor() as cursor:
            cursor.execute("SELECT * FROM table")
            return cursor.fetchall()
```

#### 3. **事务处理不当**
```python
# ❌ 错误：部分提交
try:
    cursor.execute("INSERT INTO table1 ...")
    conn.commit()  # 过早提交
    
    cursor.execute("INSERT INTO table2 ...")
    # 如果这里失败，table1已提交，table2未提交
except:
    conn.rollback()

# ✅ 正确：原子性事务
try:
    conn.begin()
    cursor.execute("INSERT INTO table1 ...")
    cursor.execute("INSERT INTO table2 ...")
    conn.commit()
except:
    conn.rollback()
```



### 📊 性能调优参数

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `connect_timeout` | 10-30秒 | 连接超时时间 |
| `read_timeout` | 30-60秒 | 读取超时时间 |
| `write_timeout` | 30-60秒 | 写入超时时间 |
| `pool_mincached` | 2-5 | 最小空闲连接数 |
| `pool_maxcached` | 10-20 | 最大空闲连接数 |
| `pool_maxconnections` | 20-100 | 最大连接数 |
| `max_allowed_packet` | 16M-64M | 最大数据包大小 |
| `charset` | utf8mb4 | 支持4字节UTF-8 |



### 🔧 调试工具

```python
class MySQLDebugger:
    """MySQL调试工具"""
    
    @staticmethod
    def enable_slow_query_log(connection: pymysql.Connection, 
                             threshold_seconds: float = 1.0):
        """启用慢查询日志"""
        cursor = connection.cursor()
        try:
            cursor.execute("SET GLOBAL slow_query_log = 'ON'")
            cursor.execute(f"SET GLOBAL long_query_time = {threshold_seconds}")
            cursor.execute("SET GLOBAL log_queries_not_using_indexes = 'ON'")
            connection.commit()
            logger.info(f"慢查询日志已启用，阈值: {threshold_seconds}秒")
        finally:
            cursor.close()
    
    @staticmethod
    def get_database_size(connection: pymysql.Connection) -> Dict[str, float]:
        """获取数据库大小"""
        sql = """
        SELECT 
            table_schema as database_name,
            ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as size_mb
        FROM information_schema.tables 
        GROUP BY table_schema
        ORDER BY size_mb DESC
        """
        
        cursor = connection.cursor(pymysql.cursors.DictCursor)
        try:
            cursor.execute(sql)
            return {row['database_name']: row['size_mb'] for row in cursor.fetchall()}
        finally:
            cursor.close()
    
    @staticmethod
    def analyze_table(connection: pymysql.Connection, table: str):
        """分析表状态"""
        cursor = connection.cursor(pymysql.cursors.DictCursor)
        try:
            # 获取表信息
            cursor.execute(f"SHOW TABLE STATUS LIKE '{table}'")
            table_status = cursor.fetchone()
            
            # 获取列信息
            cursor.execute(f"DESCRIBE {table}")
            columns = cursor.fetchall()
            
            # 获取索引信息
            cursor.execute(f"SHOW INDEX FROM {table}")
            indexes = cursor.fetchall()
            
            return {
                'table_status': table_status,
                'columns': columns,
                'indexes': indexes
            }
        finally:
            cursor.close()
```

此速查表提供了pymysql库的全面指南，涵盖了从基础连接到高级优化的各个方面，适用于不同场景下的MySQL数据库操作需求。
