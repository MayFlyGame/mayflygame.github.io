---
title: Jackson多态地反序列化Json
date: 2022-04-26 21:45:39
tags:
---

# Jackson多态地反序列化Json

Java里面，我们可以使用Jackson将一个json字符串简单地反序列化为一个Java对象，但是，如何多态地进行反序列化呢？

可以考虑使用@JsonTypeInfo这个注解。网上有很多教程（例如：[https://zhuanlan.zhihu.com/p/96108902](https://zhuanlan.zhihu.com/p/96108902)），可以参考。

比较主流地用法是，显式地或隐式地通过在类里面定义一个标志（例如下面json格式里面地的type，为cat时就解析为Cat对象；为dog时就解析为Dog对象)，然后通过@JsonTypeInfo.property来指定标志为type。

```java
{
    "type":"cat" ,
    "favoriteToy": "fish"
}
{
    "name":"dog" ,
    "breed":"Golden retriever"
}
```

那么，我的需求是，不通过property来区分怎么办？（也不可通过className，因为这样序列化出来的字符串会有className之类的关键字, 例如：{"animals":[{"**className**":"dog1","breed":"Golden retriever"},{"**className**":"cat1","favoriteToy":"fish"}]}），我只想通过类的变量名来区分如何处理？

例如下面的类和json字符串（注意，json字符串没有type，没有classname），如果里面有favoriteToy，就解析为Cat；如果里面有breed，就解析为Dog。

```java
public abstract class Animal {
    private String lqbz;  // 乱7八糟
}

public class Cat extends Animal {
    private String favoriteToy;
}

public class Dog extends Animal {
    private String breed;
}
```

```json
{"animals":[
	{"lqbz":"wtf","breed":"Golden retriever"}, ## 有breed，解析为Dog
  {"lqbz":"wtf","favoriteToy":"fish"}]    ## 有favoriteToy, 解析为Cat
}
```

当然，这个有非常多的解决方案，这里仅提供一种目前我认为比较优秀的解决方案（前提是：要使用**jackson2.12+**版本）

## 解决方案，使用@JsonTypeInfo.Id.DEDUCTION（!!!注意：使用jackson2.12+版本），完整代码如下：

```java
/**
  我这里本身的需求是，有些json是需要解析为Animal的(Animal可以实例化），既不是Dog，也不是Cat。
  所以增加defaultImpl = Animal.class一项，而且Animal没有声明为abstract。
  如果不需要解析为Animal，那就把defaultImpl去掉，并且加上abstract关键字
**/
@JsonIgnoreProperties(ignoreUnknown = true)
@JsonTypeInfo(use = JsonTypeInfo.Id.DEDUCTION, defaultImpl = Animal.class)
@JsonSubTypes({
        @JsonSubTypes.Type(Dog.class),
        @JsonSubTypes.Type(Cat.class) }
)
public class Animal {
    private String lqbz;  // 乱7八糟
    public Animal() {
    }
    public Animal(String lqbz) {
        setLqbz(lqbz);
    }
    public String getLqbz() {
        return lqbz;
    }
    public void setLqbz(String lqbz) {
        this.lqbz = lqbz;
    }
}
```

Cat/Dog的定义

```java
public class Cat extends Animal {
    private String favoriteToy;
	  public Cat() {}
    public Cat(String lqbz, String favoriteToy) {
        setLqbz(lqbz);
        setFavoriteToy(favoriteToy);
    }
	  public String getFavoriteToy() {
        return favoriteToy;
    }
    public void setFavoriteToy(String favoriteToy) {
        this.favoriteToy = favoriteToy;
    }
}

public class Dog extends Animal {
    private String breed;
    public Dog() {
    }
    public Dog(String lqbz, String breed) {
        setLqbz(lqbz);
        setBreed(breed);
    }
    public String getBreed() {
        return breed;
    }
    public void setBreed(String breed) {
        this.breed = breed;
    }
}
```

测试代码如下：

```java
public static void main(String[] args) {
        ObjectMapper objectMapper = new ObjectMapper();
        Animal myAnn = new Animal("ann");
        Animal myDog = new Dog("ruffus","english shepherd");
        Animal myCat = new Cat("goya", "mice");
        try {
            String annJson = objectMapper.writeValueAsString(myAnn);
            System.out.println("annJson: " + annJson );  // annJson: {"lqbz":"ann"}
            Animal deAnn = objectMapper.readValue( annJson, Animal.class );
            System.out.println( "Des object: " + deAnn );  // Des object: com.grapetec.vulcan.Animal@77888435
            System.out.println( "Des class: " + deAnn.getClass().getSimpleName() ); // Des class: Animal
            System.out.println("***************************************");
            String dogJson = objectMapper.writeValueAsString(myDog);
            System.out.println("dogJson: " + dogJson);  // dogJson: {"lqbz":"ruffus","breed":"english shepherd"}
            Animal deserializedDog = objectMapper.readValue(dogJson, Animal.class);
            System.out.println("Deserialized dogJson Class: " + deserializedDog.getClass().getSimpleName()); // Deserialized dogJson Class: Dog
            System.out.println("***************************************");
            String catJson = objectMapper.writeValueAsString(myCat);
            System.out.println("catJson: " + catJson); // catJson: {"lqbz":"goya","favoriteToy":"mice"}
            Animal deseriliazedCat = objectMapper.readValue(catJson, Animal.class);
            System.out.println("Deserialized catJson Class: " + deseriliazedCat.getClass().getSimpleName()); // Deserialized catJson Class: Cat

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

