## Atomic

> 解决并发的线程安全问题有以下两种：
> 1、等待唤醒机制
> 	如果抢不到锁，就将线程挂起，当锁释放的时候，然后将其唤醒重新抢锁。
> 2、自旋CAS
> 	自旋就是设置循环CAS抢锁的意思，当CAS成功的时候才会退出循环

**Atomic原子类就是利用自旋CAS来保证线程安全的。**

```java
public class TestAtomic {

    int age = 0;
    Lock lock = new ReentrantLock();

    AtomicInteger ageAtic = new AtomicInteger(0);

    //加锁方式
    public void setAgeByLock(){
        lock.lock();
        age++;
        lock.unlock();
    }

    //原子类方式
    public void setAgeByAtomic(){
        ageAtic.getAndIncrement();
    }
}
```

### Atomic原理

```java
public final int getAndIncrement() {
    //this对应着当前对应
    //valueOffset对应着当前属性在对象中的内存偏移地址
    //1代表着增加的数量
    return unsafe.getAndAddInt(this, valueOffset, 1);
}

public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        //获取在内存中的值
        var5 = this.getIntVolatile(var1, var2);

    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));//自旋CAS，var5是希望的旧值，var5+var4是新值，只有当内存中的值等于var5，也就是当前内存中的值等于希望的旧值的时候，才会更新成功，返回true

    return var5;
}
```

1. ##### 基本类型

​	使用原子的方式更新基本类型

- `AtomicInteger`：整型原子类
- `AtomicLong`：长整型原子类
- `AtomicBoolean`：布尔型原子类

方法几乎相同，以 `AtomicInteger` 为例子来介绍

​	**常用方法**：

```java
public final int get() //获取当前的值
public final int getAndSet(int newValue)//获取当前的值，并设置新的值
public final int getAndIncrement()//获取当前的值，并自增
public final int getAndDecrement() //获取当前的值，并自减
public final int getAndAdd(int delta) //获取当前的值，并加上预期的值
public final int addAndGet(int delta) //先加上预期的值，再获取增加后的值
boolean compareAndSet(int expect, int update) //如果输入的数值等于预期值，则以原子方式将该值设置为输入值（update）
public final void lazySet(int newValue)//最终设置为newValue, lazySet 提供了一种比 set 方法更弱的语义，可能导致其他线程在之后的一小段时间内还是可以读到旧的值，但可能更高效。
    
```

​	**使用示例** :

```java
// 初始化 AtomicInteger 对象，初始值为 0
AtomicInteger atomicInt = new AtomicInteger(0);

// 使用 getAndSet 方法获取当前值，并设置新值为 3
int tempValue = atomicInt.getAndSet(3);
System.out.println("tempValue: " + tempValue + "; atomicInt: " + atomicInt);

// 使用 getAndIncrement 方法获取当前值，并自增 1
tempValue = atomicInt.getAndIncrement();
System.out.println("tempValue: " + tempValue + "; atomicInt: " + atomicInt);

// 使用 getAndAdd 方法获取当前值，并增加指定值 5
tempValue = atomicInt.getAndAdd(5);
System.out.println("tempValue: " + tempValue + "; atomicInt: " + atomicInt);

// 使用 compareAndSet 方法进行原子性条件更新，期望值为 9，更新值为 10
boolean updateSuccess = atomicInt.compareAndSet(9, 10);
System.out.println("Update Success: " + updateSuccess + "; atomicInt: " + atomicInt);

// 获取当前值
int currentValue = atomicInt.get();
System.out.println("Current value: " + currentValue);

// 使用 lazySet 方法设置新值为 15
atomicInt.lazySet(15);
System.out.println("After lazySet, atomicInt: " + atomicInt);
```

```java
tempValue: 0; atomicInt: 3
tempValue: 3; atomicInt: 4
tempValue: 4; atomicInt: 9
Update Success: true; atomicInt: 10
Current value: 10
After lazySet, atomicInt: 15
```



2. ##### 数组类型

​	使用原子的方式更新数组里的某个元素

- `AtomicIntegerArray`：整型数组原子类
- `AtomicLongArray`：长整型数组原子类
- `AtomicReferenceArray`：引用类型数组原子类

​	以 `AtomicIntegerArray` 为例子来介绍

​	**常用方法**：

```java
public final int get(int i) //获取 index=i 位置元素的值
public final int getAndSet(int i, int newValue)//返回 index=i 位置的当前的值，并将其设置为新值：newValue
public final int getAndIncrement(int i)//获取 index=i 位置元素的值，并让该位置的元素自增
public final int getAndDecrement(int i) //获取 index=i 位置元素的值，并让该位置的元素自减
public final int getAndAdd(int i, int delta) //获取 index=i 位置元素的值，并加上预期的值
boolean compareAndSet(int i, int expect, int update) //如果输入的数值等于预期值，则以原子方式将 index=i 位置的元素值设置为输入值（update）
public final void lazySet(int i, int newValue)//最终 将index=i 位置的元素设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

​	**使用示例** :

```java
int[] nums = {1, 2, 3, 4, 5, 6};
// 创建 AtomicIntegerArray
AtomicIntegerArray atomicArray = new AtomicIntegerArray(nums);

// 打印 AtomicIntegerArray 中的初始值
System.out.println("Initial values in AtomicIntegerArray:");
for (int j = 0; j < nums.length; j++) {
    System.out.print("Index " + j + ": " + atomicArray.get(j) + " ");
}

// 使用 getAndSet 方法将索引 0 处的值设置为 2，并返回旧值
int tempValue = atomicArray.getAndSet(0, 2);
System.out.println("\nAfter getAndSet(0, 2):");
System.out.println("Returned value: " + tempValue);
for (int j = 0; j < atomicArray.length(); j++) {
    System.out.print("Index " + j + ": " + atomicArray.get(j) + " ");
}

// 使用 getAndIncrement 方法将索引 0 处的值加 1，并返回旧值
tempValue = atomicArray.getAndIncrement(0);
System.out.println("\nAfter getAndIncrement(0):");
System.out.println("Returned value: " + tempValue);
for (int j = 0; j < atomicArray.length(); j++) {
    System.out.print("Index " + j + ": " + atomicArray.get(j) + " ");
}

// 使用 getAndAdd 方法将索引 0 处的值增加 5，并返回旧值
tempValue = atomicArray.getAndAdd(0, 5);
System.out.println("\nAfter getAndAdd(0, 5):");
System.out.println("Returned value: " + tempValue);
for (int j = 0; j < atomicArray.length(); j++) {
    System.out.print("Index " + j + ": " + atomicArray.get(j) + " ");
}
```

```java
Initial values in AtomicIntegerArray:
Index 0: 1 Index 1: 2 Index 2: 3 Index 3: 4 Index 4: 5 Index 5: 6
After getAndSet(0, 2):
Returned value: 1
Index 0: 2 Index 1: 2 Index 2: 3 Index 3: 4 Index 4: 5 Index 5: 6
After getAndIncrement(0):
Returned value: 2
Index 0: 3 Index 1: 2 Index 2: 3 Index 3: 4 Index 4: 5 Index 5: 6
After getAndAdd(0, 5):
Returned value: 3
Index 0: 8 Index 1: 2 Index 2: 3 Index 3: 4 Index 4: 5 Index 5: 6
```



3. **引用类型**

​	基本类型原子类只能更新一个变量，如果需要原子更新多个变量，需要使用 引用类型原子类。

		- `AtomicReference`：引用类型原子类
		- `AtomicMarkableReference`：原子更新带有标记的引用类型。该类将 boolean 标记与引用关联起来，~~也可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。~~

- `AtomicStampedReference`：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题。

以 `AtomicReference` 为例子来介绍

​	**`AtomicReference` 类使用示例** :

```java
// Person 类
class Person {
    private String name;
    private int age;
    //省略getter/setter和toString
}

// 创建 AtomicReference 对象并设置初始值
AtomicReference<Person> ar = new AtomicReference<>(new Person("SnailClimb", 22));

// 打印初始值
System.out.println("Initial Person: " + ar.get().toString());

// 更新值
Person updatePerson = new Person("Daisy", 20);
ar.compareAndSet(ar.get(), updatePerson);

// 打印更新后的值
System.out.println("Updated Person: " + ar.get().toString());

// 尝试再次更新
Person anotherUpdatePerson = new Person("John", 30);
boolean isUpdated = ar.compareAndSet(updatePerson, anotherUpdatePerson);

// 打印是否更新成功及最终值
System.out.println("Second Update Success: " + isUpdated);
System.out.println("Final Person: " + ar.get().toString());
```

```java
Initial Person: Person{name='SnailClimb', age=22}
Updated Person: Person{name='Daisy', age=20}
Second Update Success: true
Final Person: Person{name='John', age=30}
```

​	**`AtomicStampedReference` 类使用示例** :

```java
// 创建一个 AtomicStampedReference 对象，初始值为 "SnailClimb"，初始版本号为 1
AtomicStampedReference<String> asr = new AtomicStampedReference<>("SnailClimb", 1);

// 打印初始值和版本号
int[] initialStamp = new int[1];
String initialRef = asr.get(initialStamp);
System.out.println("Initial Reference: " + initialRef + ", Initial Stamp: " + initialStamp[0]);

// 更新值和版本号
int oldStamp = initialStamp[0];
String oldRef = initialRef;
String newRef = "Daisy";
int newStamp = oldStamp + 1;

boolean isUpdated = asr.compareAndSet(oldRef, newRef, oldStamp, newStamp);
System.out.println("Update Success: " + isUpdated);

// 打印更新后的值和版本号
int[] updatedStamp = new int[1];
String updatedRef = asr.get(updatedStamp);
System.out.println("Updated Reference: " + updatedRef + ", Updated Stamp: " + updatedStamp[0]);

// 尝试用错误的版本号更新
boolean isUpdatedWithWrongStamp = asr.compareAndSet(newRef, "John", oldStamp, newStamp + 1);
System.out.println("Update with Wrong Stamp Success: " + isUpdatedWithWrongStamp);

// 打印最终的值和版本号
int[] finalStamp = new int[1];
String finalRef = asr.get(finalStamp);
System.out.println("Final Reference: " + finalRef + ", Final Stamp: " + finalStamp[0]);
```

```java
Initial Reference: SnailClimb, Initial Stamp: 1
Update Success: true
Updated Reference: Daisy, Updated Stamp: 2
Update with Wrong Stamp Success: false
Final Reference: Daisy, Final Stamp: 2
```

​	**`AtomicMarkableReference` 类使用示例** :

```java
// 创建一个 AtomicMarkableReference 对象，初始值为 "SnailClimb"，初始标记为 false
AtomicMarkableReference<String> amr = new AtomicMarkableReference<>("SnailClimb", false);

// 打印初始值和标记
boolean[] initialMark = new boolean[1];
String initialRef = amr.get(initialMark);
System.out.println("Initial Reference: " + initialRef + ", Initial Mark: " + initialMark[0]);

// 更新值和标记
String oldRef = initialRef;
String newRef = "Daisy";
boolean oldMark = initialMark[0];
boolean newMark = true;

boolean isUpdated = amr.compareAndSet(oldRef, newRef, oldMark, newMark);
System.out.println("Update Success: " + isUpdated);

// 打印更新后的值和标记
boolean[] updatedMark = new boolean[1];
String updatedRef = amr.get(updatedMark);
System.out.println("Updated Reference: " + updatedRef + ", Updated Mark: " + updatedMark[0]);

// 尝试用错误的标记更新
boolean isUpdatedWithWrongMark = amr.compareAndSet(newRef, "John", oldMark, !newMark);
System.out.println("Update with Wrong Mark Success: " + isUpdatedWithWrongMark);

// 打印最终的值和标记
boolean[] finalMark = new boolean[1];
String finalRef = amr.get(finalMark);
System.out.println("Final Reference: " + finalRef + ", Final Mark: " + finalMark[0]);
```

```tex
Initial Reference: SnailClimb, Initial Mark: false
Update Success: true
Updated Reference: Daisy, Updated Mark: true
Update with Wrong Mark Success: false
Final Reference: Daisy, Final Mark: true
```

4. **对象的属性修改类型**

​	如果需要原子更新某个类里的某个字段时，需要用到对象的属性修改类型原子类。

- `AtomicIntegerFieldUpdater`:原子更新整型字段的更新器
- `AtomicLongFieldUpdater`：原子更新长整型字段的更新器
- `AtomicReferenceFieldUpdater`：原子更新引用类型里的字

​	要想原子地更新对象的属性需要两步。第一步，因为对象的属性修改类型原子类都是抽象类，所以每次使用都必须使用静态方法newUpdater()创建一个更新器，并且需要设置想要更新的类和属性。第二步，更新的对象属性必须使用volatile int 修饰符。



以 `AtomicIntegerFieldUpdater`为例子来介绍

​	**`AtomicIntegerFieldUpdater` 类使用示例** :

```java
// Person 类
class Person {
    private String name;
    // 要使用 AtomicIntegerFieldUpdater，字段必须是 volatile int
    volatile int age;
    //省略getter/setter和toString
}

// 创建 AtomicIntegerFieldUpdater 对象
AtomicIntegerFieldUpdater<Person> ageUpdater = AtomicIntegerFieldUpdater.newUpdater(Person.class, "age");

// 创建 Person 对象
Person person = new Person("SnailClimb", 22);

// 打印初始值
System.out.println("Initial Person: " + person);

// 更新 age 字段
ageUpdater.incrementAndGet(person); // 自增
System.out.println("After Increment: " + person);

ageUpdater.addAndGet(person, 5); // 增加 5
System.out.println("After Adding 5: " + person);

ageUpdater.compareAndSet(person, 28, 30); // 如果当前值是 28，则设置为 30
System.out.println("After Compare and Set (28 to 30): " + person);

// 尝试使用错误的比较值进行更新
boolean isUpdated = ageUpdater.compareAndSet(person, 28, 35); // 这次应该失败
System.out.println("Compare and Set (28 to 35) Success: " + isUpdated);
System.out.println("Final Person: " + person);
```

```tex
Initial Person: Name: SnailClimb, Age: 22
After Increment: Name: SnailClimb, Age: 23
After Adding 5: Name: SnailClimb, Age: 28
After Compare and Set (28 to 30): Name: SnailClimb, Age: 30
Compare and Set (28 to 35) Success: false
Final Person: Name: SnailClimb, Age: 30
```

