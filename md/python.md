### **获取文件夹内所有文件信息**

#### **核心方法**

- **`os.walk()`**：递归遍历文件夹，返回 `(root, dirs, files)`
- **`os.listdir()`**：仅获取当前目录下的文件和子目录（不递归）
- **`pathlib.Path`**（Python 3.4+）：面向对象的路径操作，支持 `rglob('*')` 递归查找

##### 过滤临时文件

```python
def is_temp_file(filename):
    return filename.startswith('~$') or filename.endswith('.tmp')
```



### **计算文件MD5哈希值**

#### **核心模块**

- **`hashlib.md5()`**：生成MD5哈希对象
- **`hexdigest()`**：返回16进制字符串形式的哈希值

##### 计算文件hash值（可用于大文件）

```python
import hashlib

def get_file_md5(file_path, chunk_size=8192):
    md5 = hashlib.md5()
    with open(file_path, 'rb') as f:
        while chunk := f.read(chunk_size):
            md5.update(chunk)
    return md5.hexdigest()
```



### `yield` 的基本作用

### 核心功能

- 将普通函数转换为**生成器函数**
- 每次执行到 `yield` 时：
  - **暂停**函数执行
  - **返回** `yield` 后面的值
  - **保留**函数当前状态（局部变量、指令指针等）
- 下次调用时从暂停处**继续执行**
- 实现**惰性计算**和**协程**的核心机制，合理使用可以大幅提升代码效率和可读性。

##### 无限序列

```python
def simple_generator():
    count = 0
    while True:
        yield count
        count += 1
        
gen = simple_generator()
print(next(gen))  # 输出: 1
print(next(gen))  # 输出: 2
print(next(gen))  # 输出: 3
```



