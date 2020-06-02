[TOC]

# Java8函数式编程

## Lambda表达式

Lambda是一个匿名函数，我们可以把Lambda表达式理解为是一段可以传递的代码(将代码像参数一样进行传递，称为行为参数化)。Lambda允许把函数作为一个方法的参数（函数作为参数传递进方法中）。

普通写法：

```java
new Thread(new Runnable() {
  @Override
  public void run() {
    System.out.println("hello lambda");
  }
}).start();
```

lambda写法：

```java
new Thread(() -> System.out.println("hello lambda")).start();
```

### 语法

一个lambda分为三部分：参数列表、操作符、lambda体

`(paramaters) -> { statements; }`，`paramaters`是参数列表，`->`是操作符，`statements`是lambda体

lambda表达式的特征：

- 可选类型声明：不需要声明参数类型，编译器可以统一识别参数值。
- 可选的参数圆括号：一个参数无需定义圆括号，但多个参数需要定义圆括号。
- 可选的大括号：如果主体包含了一个语句，就不需要使用大括号。
- 可选的返回关键字：如果主体只有一个表达式返回值则编译器会自动返回值，大括号需要指定明表达式返回了一个数值。



## 函数式接口

函数式接口(Functional Interface)就是一个有且仅有一个抽象方法，但是可以有多个非抽象方法的接口。

函数式接口可以被隐式转换为 lambda 表达式。

`@FunctionalInterface`注解标注一个函数式接口，不能标注类，方法，枚举，属性这些。

- 如果接口被标注了`@FunctionalInterface`，这个类就必须符合函数式接口的规范

- 一个接口没有标注`@FunctionalInterface`，但如果这个接口满足函数式接口规则，依旧被当作函数式接口。

### 常用函数式接口

#### `Consumer<T>`：消费型接口

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);

    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

```java
Consumer<String> consumer1 = (v) -> System.out.println(v + "step1,");
Consumer<String> consumer2 = (v) -> System.out.println(v + "step2,");
consumer1.andThen(consumer2).accept("test "); // test step1, test step2
```

#### `Supplier<T>`：供给型接口

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

```java
Supplier<String> supplier = () -> "777";
System.out.println(supplier.get()); // 777
```

#### `Function<T,R>`：函数型接口

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);

    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }

    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```

```java
Function<Integer, Integer> function1 = v -> v * 2;
Function<Integer, Integer> function2 = v -> v * v;
int result1 = function1.andThen(function2).apply(5); // 100
int result2 = function1.compose(function2).apply(5); // 50
```

#### `Predicate<T>` ：断言型接口

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }

    /**
     * Returns a predicate that is the negation of the supplied predicate.
     * This is accomplished by returning result of the calling
     * {@code target.negate()}.
     *
     * @param <T>     the type of arguments to the specified predicate
     * @param target  predicate to negate
     *
     * @return a predicate that negates the results of the supplied
     *         predicate
     *
     * @throws NullPointerException if target is null
     *
     * @since 11
     */
    @SuppressWarnings("unchecked")
    static <T> Predicate<T> not(Predicate<? super T> target) {
        Objects.requireNonNull(target);
        return (Predicate<T>)target.negate();
    }
}
```

```java
Predicate<String> predicate1 = v -> true;
Predicate<String> predicate2 = v -> false;
boolean test1 = predicate1.and(predicate2).test("and"); // false
boolean test2 = predicate1.or(predicate2).test("or"); // true
boolean test3 = predicate1.negate().test("negate"); // false
boolean test4 = Predicate.not(predicate2).test("not"); // true
```

### 自定义函数式接口

按照下面的格式定义，可以自定义函数式接口：

```java
 @FunctionalInterface
 修饰符 interface 接口名称 {
    返回值类型 方法名称(可选参数信息);
    // 其他非抽象方法内容
 }
```

```java
@FunctionalInterface
interface MyFunc {
    int func1(String s1, String s2);
  
  	default void xxx();
  
  	static void xxxx();
}
```

```java
MyFunc myFunc = (s1, s2) -> s1.length() + s2.length();
int result = myFunc.func1("aaa", "bbb"); // 6
```



## Stream

### Stream

### Sink