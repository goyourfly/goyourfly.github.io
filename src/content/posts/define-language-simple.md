---
title: 设计一个全新的程序语言 'Simple'
published: 2017-01-25
tags: [Java]
category: 技术
draft: false
---

# 设计一个全新的程序语言"Simple"

### 语法规则

* 语法规则的目的是讲例如 `y = x + 1` 这样的有效程序和`1dkfa/,d`这种无效的字符串区分开
* 语法规则解决具有二义性程序问题，如 `1 + 2 * 3`的运算优先级
* 语法规则程序表面是否合乎规则，不关心程序的含义
* 语法解析器总是把源代码转换成抽象语法🌲（AST）

### 语义
> 程序作了什么？

#### 1.小步语义

小步语义的含义有点类似于数学求值的方式，如，`(1 * 2) + (3 * 4)`的计算步骤如下：

* 执行 `1 * 2` 的乘法得到 2
* 执行 `3 * 4` 的乘法得到 12
* 执行 `2 + 12` 的加法得到 14

#### 2.表达式
> `1 + 2 * 3` 这就是表达式😊

如果我们用Java写一个 `1 + 2 * 3` 的程序，并将结果打印出来，该怎么做？

```java
...
int a = 1 + 2 * 3;
System.out.println("a:" + a);
...
```

看起来很简单，那是因为Java在后台默默的帮我们做了很多工作，前面讲过，首先将 1 + 2 * 3抽象成语法树，然后在使用分析语义，算出结果，当然，具体实现逻辑会更复杂。

那我们怎么让Simple实现这个计算逻辑呢？

到现在为止，我们的Simple还是一张白纸，像一个刚出世的小孩，它什么都不认识，那我们首先要教它认识数字，所以我们定义一个特殊的名称Number，告诉它只要遇到这个字符串，就认为是数字。
>注意，Simple中的数字和数学中的数字不是一个概念，这里Simple语言的底层实现我们用的是Java，就行Java的底层用到的是C或C++一样。

```java
//在我们Simple的语法中，万物都基于Simple，在以后的写法中
//默认所有类型都继承Simple
class Number extends Simple{
	int value;
	public Number(int value){
		this.value = value;
	}
}
```

OK，那现在在Simple中，我们就可以用`Number(1)`表示 `1`了。但是想要实现上面的表达式，我们还需要让Simple这个小孩认识加法和乘法。

```java
class Add extends Simple{
	Simple left;
	Simple right;
	
	public Add(Simple left, Simple right){
		this.left = left;
		this.right = right;
	}
}

class Multipy extends Simple{
	Simple left;
	Simple right;
	
	public Multipy(Simple left, Simple right){
		this.left = left;
		this.right = right;
	}
}

```

到目前为止呢，我们已经定义了数字、加法和乘法，所以上面的表达式我们已经可以用Simple来表示了：
> 好长...长...污污污...

```Java
Add(Number(1), Multipy(Number(2),Number(3)));
```

请注意，到目前为止，我们只是定义了Simple语法规则，离让它真正跑起来还有几步之遥🙂...

我们前面语义部分介绍到小步语义，没错，在这里我们就用到了它，按照小步语义的规则，这个计算应该是被拆分为3步：

* 1 + 2 * 3
* 1 + 6
* 7

用我们人的思维逻辑，一眼就能看出应该先计算 `2 * 3` 再和一相加，但是对于程序来说，它如何能知道呢？在这里，我们引入一套规则，我们定义每个表达式（数字也认为是一个表达式）都有一个是否可以继续规约(reducible,类似于数学中的求值），如表达式 `2 * 3` 可以继续规约为 `6` ，但是 `6` 不可以规约。

我们前面讲到在Simple语言中，万物基于Simple，所以，我们给Simple加两个方法。

```java
abstract class Simple{
	//是否可规约
	public boolean abstract reducible();
	//规约
	public Simple abstract reduce();
}
```

那么，我们所有的子类都要实现这两个方法。

```java
class Number extends Simple{
	...
	//因为Number不可继续规约，故返回false
	public boolean reducible(){
		return false;
	}
	
	public Simple reduce(){
		//do nothing
		return this;
	}
	...
}

class Add extends Simple{
	...
	//加法可以规约
	public boolean reducible(){
		return true;
	}
	
	public Simple reduce(){
		if(left.reducible()){
		//如果这个表达式左边可以规约，则优先规约
			return new Add(left.reduce(),right);
		}else if(right.reducible()){
		//如果右边可以规约，则先规约右边
			return new Add(left,right.reduce());
		}else{
		//如果两边都不可以规约，则两边都是数字，直接加起来
			return new Number(left.value + right.value);
		}
	}
	...
}

class Multipy extends Simple{
	...
	//乘法可以规约
	public boolean reducible(){
		return true;
	}
	
	public Simple reduce(){
		if(left.reducible()){
		//如果这个表达式左边可以规约，则优先规约
			return new Multipy(left.reduce(),right);
		}else if(right.reducible()){
		//如果右边可以规约，则先规约右边
			return new Multipy(left,right.reduce());
		}else{
		//如果两边都不可以规约，则两边都是数字，直接相乘
			return new Number(left.value * right.value);
		}
	}
}
```
好了，现在Simple已经能处理加法和乘法的混合运算了，让我给它输入一个表达式试试效果吧。

```java
Simple simple = new Add(new Multipy(new Number(1),new Number(2)),new Multipy(new Number(3),new Number(4)));
System.out.println(simple);
    while (simple.reducible()){
    simple = simple.reduce();
        System.out.println(simple);
    }
}

/* 以下是输出日志：
 1  *  2  +  3  *  4 
 2  +  3  *  4 
 2  +  12 
 14 
*/
```

#### 以下是完整Demo

```java
/**
 * 完整运行Demo，如有错误，欢迎指正
 * Created by gaoyufei on 2017/1/25.
 */
public class MySimpleLanguage {

    interface Simple {
        public boolean reducible();

        public Simple reduce();
    }

    static class Number implements Simple {
        public int value;

        public Number(int value) {
            this.value = value;
        }

        public boolean reducible() {
            return false;
        }

        public Simple reduce() {
            return this;
        }

        @Override
        public String toString() {
            return " " + value + " ";
        }
    }

    static class Add implements Simple {
        public Simple left;
        public Simple right;
        public Add(Simple left,Simple right){
            this.left = left;
            this.right = right;
        }
        public boolean reducible() {
            return true;
        }

        public Simple reduce() {
            if(left.reducible()){
                return new Add(left.reduce(),right);
            }else if(right.reducible()){
                return new Add(left,right.reduce());
            }else {
                return new Number(((Number)left).value + ((Number)right).value);
            }
        }

        @Override
        public String toString() {
            return left.toString() + " + " + right.toString();
        }
    }

    static class Multipy implements Simple {
        public Simple left;
        public Simple right;
        public Multipy(Simple left,Simple right){
            this.left = left;
            this.right = right;
        }
        public boolean reducible() {
            return true;
        }

        public Simple reduce() {
            if(left.reducible()){
                return new Multipy(left.reduce(),right);
            }else if(right.reducible()){
                return new Multipy(left,right.reduce());
            }else {
                return new Number(((Number)left).value * ((Number)right).value);
            }
        }

        @Override
        public String toString() {
            return left.toString() + " * " + right.toString();
        }
    }

    public static void main(String args[]) {
        System.out.println("---------Start simple---------");
        Simple simple = new Add(new Multipy(new Number(1),new Number(2)),new Multipy(new Number(3),new Number(4)));
        System.out.println(simple);
        while (simple.reducible()){
            simple = simple.reduce();
            System.out.println(simple);
        }
        System.out.println("---------End simple-----------");
    }
}


```