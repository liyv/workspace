
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
