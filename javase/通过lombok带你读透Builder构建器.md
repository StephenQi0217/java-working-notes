# 通过lombok带你读透Builder构建器

很久之前，我在《effective java》上看过Builder构建器相关的内容，但实际开发中不经常用。后来，在项目中使用了lombok,发现它有一个注解“@Builder”，就是为java bean生成一个构建器。于是，回头重新复习了下相关知识，整理如下。

## 1. lombok使用样例

```java
// 创建名为Officer的java bean
@Builder
public class Officer {
    private final String id;
    private final String name;
    private final int age;
    private final String department;
}

// 调用构建器生成Officer实例
class BuilderTest {
    public static void main(String[] args) {
        Officer officer = Officer.builder().id("00001").name("simon qi")
                .age(26).department("departmentA").build();
    }
}
```

## 2. 反编译lombok生成的Officer.class
*注意：下面请区分两组名词:"builder方法"和“build方法”，“构造器”和“构建器”。*

```java
public class Officer {
    private final String id;
    private final String name;
    private final int age;
    private final String department;

    Officer(String id, String name, int age, String department) {
        this.id = id;
        this.name = name;
        this.age = age;
        this.department = department;
    }

    public static Officer.OfficerBuilder builder() {
        return new Officer.OfficerBuilder();
    }

    public static class OfficerBuilder {
        private String id;
        private String name;
        private int age;
        private String department;

        OfficerBuilder() {
        }

        public Officer.OfficerBuilder id(String id) {
            this.id = id;
            return this;
        }

        public Officer.OfficerBuilder name(String name) {
            this.name = name;
            return this;
        }

        public Officer.OfficerBuilder age(int age) {
            this.age = age;
            return this;
        }

        public Officer.OfficerBuilder department(String department) {
            this.department = department;
            return this;
        }

        public Officer build() {
            return new Officer(this.id, this.name, this.age, this.department);
        }

        public String toString() {
            return "Officer.OfficerBuilder(id=" + this.id + ", name=" + this.name + ", age=" + this.age + ", department=" + this.department + ")";
        }
    }
}
```

我们通过反编译Officer.class，获得上方的源码(最好用idea自带的反编译器，jd-gui反编译的源码不全)。

我们发现源码中有一个OfficerBuilder的静态内部类，我们在调用builder方法时，实际返回了这个静态内部类的实例。这个OfficerBuilder类，具有和Officer相同的成员变量，且拥有名为id,name,age和department的方法。这些以Officer的成员变量命名的方法，都是给OfficerBuilder的成员变量赋值，并返回this。

这些方法返回this，其实就是返回调用这些方法的OfficerBuilder对象，也可称为“返回对象本身”。通过返回对象本身，形成了方法的链式调用。

再看build方法，它是OfficerBuilder类的方法。它创建了一个新的Officer对象，并将自身的成员变量值，传给了Officer的成员变量。所以`Officer officer = Officer.builder().id("00001").name("simon qi").age(26).department("departmentA").build();`的写法，等价于下面的写法：

```java
Officer.OfficerBuilder officerBuilder = new Officer.OfficerBuilder();
officerBuilder.id("00001").name("simon qi").age(26).department("departmentA");
Officer officer = officerBuilder.build();
```

所以为什么这种模式叫“构建器”，因为要创建Officer类的实例,首先要创建OfficerBuilder类的实例。而这个OfficerBuilder也就是构建器，是创建Officer对象的一个过渡者。所以利用这种模式，会有中间实例的创建，会加大虚拟机内存的消耗。

## 3. 只用@Builder注解的bug

我们只用@Builder注解，我发现lombok为Officer类生成的构造器是“default”的(不添加权限修饰符，默认为“default”的)。

我们之所以用构建器模式，是希望用户用构建器提供的方法去创建实例。但“default”的构造器，可以被同package的类调用(default限制不同package类的调用)。所以，我们需要将此构造器设为private的。这时就需要用到“@AllArgsConstructor(access = AccessLevel.PRIVATE)”。我们这时再看反编译后的构造器：

```java
private Officer(String id, String name, int age, String department) {
    this.id = id;
    this.name = name;
    this.age = age;
    this.department = department;
}
```

所以，使用lombok的构建器，应将“@Builder”和“@AllArgsConstructor(access = AccessLevel.PRIVATE)”相结合，最终写法：

```java
@Builder
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Officer {
    private final String id;
    private final String name;
    private final int age;
    private final String department;
}
```

## 4. 为什么使用构建器模式

若一个类具有大量的成员变量，我们就需要提供一个全参的构造器或大量的set方法。这让实例的创建和赋值，变得很麻烦，且不直观。我们通过构建器，可以让变量的赋值变成链式调用，而且调用的方法名对应着成员变量的名称。让对象的创建和赋值都变得很简洁、直观。

## 5. 链式方法赋值，一定要用构建器模式吗？

不一定要用到构建器模式，之所以使用构建器模式，是因为我们要创造的对象是一个成员变量不可变的对象。

你返回去看Officer类和OfficerBuilder类，你会发现Officer类的成员变量都是final的，而OfficerBuilder的成员变量却没用final修饰。因为final修饰的成员变量，需要在实例创建时就把值确定下来。但在类具有大量成员变量的时候，我们是不希望用户直接调用全参构造器的。

所以我们使用了OfficerBuilder的中间类。这个类为了实现链式赋值，才将变量设为非final的。无论你OfficerBuilder实例怎么赋值，怎么改变，当你调用build方法时，就会返回一个成员变量不可变的Officer实例。

那如果有大量属性，但不需要它是成员变量不可变的对象，我们还需要构建器模式吗？答案是，不需要，我们可以参考构建器，把代码赋值改成链式的即可：

```java
public class Officer {
    private String id;
    private String name;
    private int age;
    private String department;

    public static Officer build() {
        return new Officer();
    }

    private Officer() {
    }

    public Officer id(String id) {
        this.id = id;
        return this;
    }

    public Officer name(String name) {
        this.name = name;
        return this;
    }

    public Officer age(int age) {
        this.age = age;
        return this;
    }

    public Officer department(String department) {
        this.department = department;
        return this;
    }
}

调用样式：
Officer officer = Officer.build().id("00001").name("simon qi").age(26).department("departmentA");
其实这时候构造器设为非private也行，写成private，只是为了调用build()显得更好看。
将构造器设为非private，可以写为如下形式：
Officer officer = new Officer().id("00001").name("simon qi").age(26).department("departmentA");
```

所以，我觉得你在使用lombok的"@Builder"注解的时候，还是要思考一下。当你不需要成员变量不可变的时候，你完全没必要使用构建器模式，因为这会消耗java虚拟机的内存。

## 6. 总结

所以，我一直推荐学习知识，要在项目中去学习。通过项目，去探索项目以外的知识点，才是提升自己的快捷方法。而且知识不能学死了，不能网上有哪些知识点，我们就只考虑这些知识点。我们要去思考一些别人不常想到的问题。比如，我们为什么要用中间类去做过渡，这么写的目的是什么。

将上述知识吃透，面试应对构建器的时候，也就得心应手了。而且通过实战去回答问题，也更能彰显你是个爱思考的员工。

```
作者：simon Qi
QQ: 591232672
e-mail：simonqi0217@qq.com
版权声明：转载请保留此链接，不得用于商业用途。
虽然我不是最优秀的程序员，但我还是想尽自己最大的努力，去分享一些学习心得。
如有错误，欢迎指正。若有幸能博得您的喜爱，欢迎关注及点赞哦。
愿我们共同进步!
```

