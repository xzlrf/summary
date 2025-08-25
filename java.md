#### 1、==和equals有什么区别

- Object中equals方法的实现就是==，所以如果一个类不重写equals方法，那么判断就是用==判断
- ==对于基本类型来说比较的是值是否相等，对用包装类型来说比较的是内存地址是否一致
- equals比较的是值
- 所以一般都会对equals方法进行重写，equals相等的两个类hashcode一定相等，反之则不一定（哈希冲突）

#### 2、什么是方法重写override、重载overload？

- 对于继承关系的父子类，父子类中存在的方法名称相同，参数列表相同，返回值要相同或者是其子类，访问修饰符子类的方法不能比父类更严格，子类抛出的异常也不能比父类更宽泛，是实现多态的基础

- 重载是在同一个类中方法名相同，参数列表不同，返回值无要求的一系列方法。

- 多态的实现原理是动态绑定和虚拟调用

  - 动态绑定：是指在程序运行时才根据对象的实际类型（而不是引用类型）来决定调用哪个类的方法，动态绑定看右边，实际调用类型，静态绑定看左边引用类型。几乎所有方法都是动态绑定，除了private、final、static、构造器、成员变量方法是静态绑定。

  - 虚拟调用：在程序**运行时**才能确定具体调用哪个方法的机制，所有普通实例方法的调用默认都是虚拟调用。虚拟调用需要查方法表，会有性能损失，非虚拟调用会更快，通常会被优化如内联。

    ```java
    class Parent {
        public final void finalMethod() { } // 非虚方法
        public static void staticMethod() { } // 非虚方法
        private void privateMethod() { } // 非虚方法
        public void normalMethod() { } // 虚方法
    }
    
    class Child extends Parent {
        // 不能重写finalMethod
        // 可以"隐藏"staticMethod，但不是重写
        // 无法继承privateMethod
    
        @Override
        public void normalMethod() { } // 重写，是虚方法
    }
    
    public class Test {
        public static void main(String[] args) {
            Child obj = new Child();
            obj.finalMethod();   // 非虚拟调用，编译时已确定
            Child.staticMethod(); // 非虚拟调用，编译时已确定
            obj.normalMethod();  // 虚拟调用，运行时确定
        }
    }
    ```

    

#### 3、finally中的代码一定会执行么
finally中的代码通常会执行，但是在一些极端情况下会导致不执行。比如：

- System.exit()
- 线程死锁/无限阻塞

#### 4、多态的实现原理

#### 5、ArrayList和LinkedList什么区别

#### 6、ArrayList和Vector有什么区别

#### 7、抽象类和普通类有什么区别

8、HashMap和ConcurrentHashMap有什么区别
