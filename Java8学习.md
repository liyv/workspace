
# Java8

## 1.一些概念

1. Lamabda--匿名函数、流、默认方法

### 例子：筛选苹果--》方法引用

```java
//苹果集合
List<Apple> inventory = Arrays.asList(new Apple(80,"green"),
                                              new Apple(155, "green"),
                                              new Apple(120, "red"));


//改进，使用谓词
public static List<Apple> filterApples(List<Apple> inventory, Predicate<Apple> p){
        List<Apple> result = new ArrayList<>();
        for(Apple apple : inventory){
            if(p.test(apple)){
                result.add(apple);
            }
        }
        return result;
}
public static boolean isGreenApple(Apple apple) {
        return "green".equals(apple.getColor()); 
    }

public static boolean isHeavyApple(Apple apple) {
        return apple.getWeight() > 150;
    }
//方法引用

 // [Apple{color='green', weight=80}, Apple{color='green', weight=155}]
List<Apple> greenApples = filterApples(inventory, FilteringApples::isGreenApple);
```

### Lambda--匿名函数

>为了简化isGreenApple、isHeavyApple等这些只会使用几次的方法，Java8引入了匿名函数Lambda

```java
 List<Apple> greenApples2 = filterApples(inventory, (Apple a) -> "green".equals(a.getColor()));
```

### 流 Stream

### 默认方法

## 2.通过行为参数化传递代码

```java
//筛选出绿色的苹果
public static List<Apple> filterGreenApples(List<Apple> inventory){
        List<Apple> result = new ArrayList<>();
        for (Apple apple: inventory){
            if ("green".equals(apple.getColor())) {
                result.add(apple);
            }
        }
        return result;
}
//筛选出红色的苹果
...

//筛选出重量大于150克的苹果
public static List<Apple> filterApplesByWeight(List<Apple> inventory, int weight){
    List<Apple> result = new ArrayList<>();
        for(Apple apple: inventory){
            if(apple.getWeight() > weight){
                result.add(apple);
            }
        }
        return result;
}

//更多需求变更：
....
```

>对需求抽象：需要根据Apple的某些属性来返回一个boolean值，即谓词(一个返回boolean值的函数)

 定义接口对选择标准建模

```java
interface ApplePredicate{
    public boolean test(Apple a);
}
//多个实现
static class AppleWeightPredicate implements ApplePredicate{
    public boolean test(Apple apple){
        return apple.getWeight() > 150; 
    }
}
static class AppleColorPredicate implements ApplePredicate{
    public boolean test(Apple apple){
        return "green".equals(apple.getColor());
    }
}

//使用 ApplePredicate 改进代码

public static List<Apple> filter(List<Apple> inventory, ApplePredicate p){
    List<Apple> result = new ArrayList<>();
    for(Apple apple : inventory){
        if(p.test(apple)){
            result.add(apple);
        }
    }
    return result;
}
//多种行为一个参数
```

缺点：繁琐，声明了很多只需要实例化一次的类

再次改进：使用匿名类

```java
List<Apple> redApples2 = filter(inventory, new ApplePredicate() {
    public boolean test(Apple a){
        return a.getColor().equals("red");
    }
});
```

还是不够直观，继续改进，使用Lambda表达式

```java
 List<Apple> greenApples2 = filterApples(inventory, (Apple a) -> "green".equals(a.getColor()));
```

## Lambda 表达式

>可把Lambda表达式理解为可传递的匿名函数的一种方式。
>没有名称，有参数列表、函数主体、返回类型，可能还有一个可以抛出的异常列表。

![img](lambda_1.png)

```java
//Java8中有效的Lambda表达式：
(String s) -> s.length();
//Lambda没有return语句，因为已经隐含了return
(Apple a) -> a.getWeight() > 150
() -> 42
...
//哪些是正确的Lambda表达式？
//1.
（） -> {}
//2.
() -> "Java*"
//3.
() -> {return "Java8"}
//4.
(Integer i) -> return "Alan" + i;
//5.
(String s) -> {"返回Java8"}
```
### 在哪里以及如何使用Lambda

>**函数式接口**
就是只定义一个抽象方法的接口。

- Comparator<T>
- Runnable
- Callable
...
  
接口还可以有默认方法，但不管有多少默认方法，只要接口中只有一个抽象方法，它就是一个函数式接口。想想扩展接口的情况？

Lambda表达式允许直接以内联的形式为函数式接口的抽象方法提供实现，并把整个表达式作为函数式接口的实例。

```java

Runnable run = () -> System.out.pringln("hello Lambda");
```

#### 函数描述符

函数式接口的抽象方法的签名基本上就是Lambda表达式的签名。将这种抽象方法叫做”函数描述符“。

#### Java8 中定义的函数式接口

在java.util.function包中

- Predicate< T>

- Consumer< T>

- Function

泛型函数式接口，由于泛型只能绑定到引用类型，所以在组合使用基本类型和泛型时，会引发装箱和拆箱机制，这在性能方面是有些代价的，为了避免装箱、拆箱，Java8引入了专门针对基本类型的一些接口：

- IntPredicate

- DoublePredicate

- IntConsumer

等等

**Java8中常用的函数式接口**

函数式接口 | 函数描述符 | 原始类型特化
------------ | ------------- | -------------
Predicate\<T> | T->boolean | IntPredicate,LongPredicate等
Consumer\<T> | T->void | IntConsumer
Function\<T,R>|T->R| ...
Supplier\<T> | ()->T |...
UnaryOperator\<T> | T->T|..
BinaryOperator\<T>| (T,T)->T|
BiPredicate\<L,R>|(L,R)->boolean|
BiConsumer\<T,U>| (T,U)->void|
BiFunction\<T,U,R>|(T,U)->R|

**编译器如何对Lambda做类型检查?**

#### 类型检查、类型推断以及限制

#### 方法引用

排序的例子

```java
//
inventory.sort((Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight()));

//方法引用
inventory.sort(comparing(Apple::getWeight));
```

方法引用可以被看作仅仅调用特定方法的Lambda的一种快捷写法。方法引用就是让你根据已有的方法实现来创建Lambda表达式，方法引用不需要括号，因为不是实际调用这个方法，仅仅是一种快捷写法。

方法引用主要有3类：

- 指向*静态方法*的方法引用 Integer::parseInt

- 指向*任意类型实例方法*的方法引用 String::length

- 指向*现有对象的实例方法*的方法引用

第二种思想就是：在引用一个对象的方法，而这个对象本身是Lambda的一个参数。

```java
(String s) -> s.toUpperCase();

String::toUpperCase
```

第三种的思想是：你在Lambda的方法主体中调用一个已经存在的外部对象中的方法

```java
//Lambda表达式
() -> someObj.getValue();

//可以写为:
someObj::getValue
```

构造函数引用

```java
//其功能与指向静态方法的引用类似
ClassName::new
```

用不同的排序策略给apple列表排序

```java
List.sort();

//1.传递代码

 static class AppleComparator implements Comparator<Apple> {
        public int compare(Apple a1, Apple a2){
            return a1.getWeight().compareTo(a2.getWeight());
        }
}

inventory.sort(new AppleComparator());

//2.使用匿名类
inventory.sort(new Comparator<Apple>() {
            public int compare(Apple a1, Apple a2){
                return a1.getWeight().compareTo(a2.getWeight()); 
}});

//3.使用Lambda表达式
inventory.sort((Apple a1, Apple a2) -> a1.getWeight().compareTo(a2.getWeight()));
//类型推断
inventory.sort((a1, a2) -> a1.getWeight().compareTo(a2.getWeight()));

//Comparator有一个叫作comparing的静态辅助方法，其可以接受一个Function来提取Comparable键值，并生成一个Comparator对象,
//为什么接口可以有静态方法？
Comparator<Apple> c = Comparator.comparing((Apple a) -> a.getWeight());
//4.使用方法引用
inventory.sort(comparing(Apple::getWeight));

```

## 流--Stream

```java
 public static final List<Dish> menu =
            Arrays.asList( new Dish("pork", false, 800, Dish.Type.MEAT),
                           new Dish("beef", false, 700, Dish.Type.MEAT),
                           new Dish("chicken", false, 400, Dish.Type.MEAT),
                           new Dish("french fries", true, 530, Dish.Type.OTHER),
                           new Dish("rice", true, 350, Dish.Type.OTHER),
                           new Dish("season fruit", true, 120, Dish.Type.OTHER),
                           new Dish("pizza", true, 550, Dish.Type.OTHER),
                           new Dish("prawns", false, 400, Dish.Type.FISH),
                           new Dish("salmon", false, 450, Dish.Type.FISH));
}

//提取低热量的菜肴的名称

//Java7实现方式
public static List<String> getLowCaloricDishesNamesInJava7(List<Dish> dishes){
        List<Dish> lowCaloricDishes = new ArrayList<>();
        //首先取出低热量的菜肴
        for(Dish d: dishes){
            if(d.getCalories() < 400){
                lowCaloricDishes.add(d);
            }
        }
        //排序
        List<String> lowCaloricDishesName = new ArrayList<>();
        Collections.sort(lowCaloricDishes, new Comparator<Dish>() {
            public int compare(Dish d1, Dish d2){
                return Integer.compare(d1.getCalories(), d2.getCalories());
            }
        });
        //取出菜肴的名称
        for(Dish d: lowCaloricDishes){
            lowCaloricDishesName.add(d.getName());
        }
        return lowCaloricDishesName;
}
//Java8的实现方式
public static List<String> getLowCaloricDishesNamesInJava8(List<Dish> dishes){
        return dishes.stream()
                .filter(d -> d.getCalories() < 400)
                .sorted(comparing(Dish::getCalories))
                .map(Dish::getName)
                .collect(toList());
}

```

流简介：从支持数据处理操作的源生成的元素序列，流的目的在于表达计算，集合讲的是数据，流讲的是计算。

```java


```