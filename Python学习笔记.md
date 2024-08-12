## 一、Python的内置类型

### 1、Python中的类型分类



### 2、简单类型

#### 2.1、布尔类型



#### 2.2、整数类型



#### 2.3、浮点类型



#### 2.4、复数类型



#### 2.5、空类型



#### 2.6、简单类型的运算



### 3、常量类型



### 4、序列类型



### 5、列表类型

#### 5.1、列表切片



#### 5.2、列表的操作运算



#### 5.3、列表的方法

##### 5.3.1、append()

**描述：**append()方法用于在列表末尾添加新的对象，

**语法：**list.append(obj)

**返回值：**该方法无返回值，但是会修改原来的列表

**实例：**

```python
#结果为[1,2,3,4,5]
test = [1,2,3,4]
test.append(5)
print(test)
```

##### 5.3.2、insert()

**描述：**insert()方法用于在列表的指定位置插入对象

**语法：**insert(index,obj)

**返回值：**无返回值

**实例：**

```python
#结果为[1,2,3,4,'插在末尾']
test = [1,2,3,4]
test.insert(4,'插在末尾')
print(test)
```

##### 5.3.3、pop()

**描述：**pop()方法用于移除列表中的某个元素(默认是列表的最后一个元素)，并返回该元素的值

**语法：**list.pop(index)

**返回值：**返回列表中被移除的元素的值

**实例：**

```python
#结果为[1,2,3]
test = [1,2,3,4]
test.pop()
print(test)

#结果为[1, 3, 4]
test = [1,2,3,4]
test.pop(1)
print(test)

#结果为2
test = [1,2,3,4]
print(test.pop(1))
```

##### 5.3.4、remove()

**描述：**remove()用于移除列表中某个值的第一个匹配项

**语法：**remove(obj)

**返回值：**无返回值

**实例：**

```python
#结果为[1, 3, 4, 2, 2, 2]
test = [1,2,3,4,2,2,2]
test.remove(2)
print(test)
```

##### 5.3.5、extend()

**描述：**extend()方法用于在列表末尾追加另一个可迭代对象(iterable)的值，可迭代对象可以是任何可遍历的数据类型，通常是列表、元组、字符串等

**语法：**list.extend(iterable)

**返回值：**无返回值

**实例：**

```python
#结果为[1, 2, 3, 4, 2, 2, 2, 'hello', 'world']
test = [1,2,3,4,2,2,2]
test1 = ['hello','world']
test.extend(test1)
print(test)

#结果为[1, 2, 3, 4, 2, 2, 2, 'a', 'b', 'c', 'd', 'e', 'f', 'g']
test = [1,2,3,4,2,2,2]
test1 = 'abcdefg'
test.extend(test1)
print(test)
```

##### 5.3.6、count()

**描述：**count() 方法用于统计某个元素在列表中出现的次数

**语法：**list.count(obj)

**返回值：**返回元素在列表中出现的次数

**实例：**

```python
#结果为4
test = [1,2,3,4,2,2,2]
print(test.count(2))
```

##### 5.3.7、index()

**描述：**index() 函数用于从列表指定的索引范围中找出索引最靠前且值相等的那个值的索引

**语法：**list.index(x,index_start,index_end)

**返回值：**返回查找对象的索引位置，如果没有找到会抛出异常

**实例：**

```python
#结果为1
test = [1,2,3,4,2,2,2]
print(test.index(2))

#结果为4
test = [1,2,3,4,2,2,2]
print(test.index(2,2))
```

##### 5.3.8、reverse()

**描述：**reverse() 方法用于反转列表中元素

**语法：**list.reverse()

**返回值：**无返回值

**实例：**

```python
#结果为[2, 2, 2, 4, 3, 2, 1]
test = [1,2,3,4,2,2,2]
test.reverse()
print(test)
```

##### 5.3.9、sort()

**描述：**sort()用于将列表中的函数进行排序

**语法：**list.sort(key=None, reverse=False)

key参数可以指定一个函数，然后再每个元素上调用，来决定排序顺序

reverse=False表示以升序排列(默认)，反之reverse=True为降序排列

**返回值：**无返回值

**实例：**

```python
#结果为[1, 2, 2, 2, 2, 3, 4]
test = [1,2,3,4,2,2,2]
test.sort()
print(test)

#结果为['a', 'sda', 'asdsadas']
test = ['sda','asdsadas','a']
test.sort(key=len)
print(test)
```

##### 5.3.10、clear()

**描述：**clear() 方法用于清空列表，类似于del list[:]

**语法：**list.clear()

**返回值：**无返回值

**实例：**

```python
#结果为空列表[]
test = ['sda','asdsadas','a']
test.clear()
print(test)
```

##### 5.3.11、copy()

**描述：**copy()方法用于复制列表

**语法：**list.copy()

**返回值：**返回复制之后的新列表

**实例：**

```python
#结果为['sda','asdsadas','a']
test = ['sda','asdsadas','a']
b = test.copy()
print(b)

#结果为['sda', 'asdsadas', 'a', 'go'] ['sda', 'asdsadas', 'a'] ['sda', 'asdsadas', 'a', 'go']
test = ['sda','asdsadas','a']
a = test.copy()
b = test
test.append("go")
print(test,a,b)
```

### 6、元组类型





### 7、字符串类型

#### 7.1、字符串的方法

##### 7.1.1、split()

**描述：**split()方法通过指定分隔符对字符串进行切片

**语法：**string.split(separator, maxsplit)，separator是分割符，maxsplit是最大分割次数

**返回值：**返回分割完成后生成的字符串列表

**实例：**

```python
#结果为['hello', 'world']
test = "hello  world"
print(test.split())

#结果为['hell', '  w', 'rld']
test = "hello  world"
print(test.split('o'))

#结果为['hell', '  world']
test = "hello  world"
print(test.split('o',1))
```

##### 7.1.2、splitlines()

**描述：**

**语法：**

**返回值：**

**实例：**

7.1.3、

**描述：**

**语法：**

**返回值：**

**实例：**

7.1.4、

**描述：**

**语法：**

**返回值：**

**实例：**

7.1.5、



7.1.6、



7.1.7、



7.1.8、



### 8、字典类型



### 9、集合类型



## 二、流程控制和函数



三、