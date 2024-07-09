---
title: Activity - 启动模式
published: 2017-03-06
tags: [Android, Activity]
category: 技术
draft: false
---

Android的每个UI都是依附于Activity之上，Activity相当于UI的容器，而我们经常的切换页面，实际上就是在不停的startActivity。但是在startActivity的过程中，经常会有些不一样的需求，比如说，当我打开Activity A的时候，我想关掉其他所有已经打开的Activity，或者说，我想在新的Task中启动Activity。
> Task是Activity的容器，是一个栈，每个Task可以有很多个Activity，我们可以指定一个Activity是否存放在一个新的Task，或者放在当前Task的栈顶。

Activity共有4种启动模式：

* standard
* singleTask
* singleInstance
* singleTop

### standard

在standard模式下，我们首先启动Activity A，接着我们在此启动Activity A，我们发现两个Activity A会同时存在，也就是说，在standard launchmode下，每次调用startActivity必然会启动一个新的Activity，如下图所示：

![standard A](https://cloud.githubusercontent.com/assets/7019862/23596549/92da8a98-0266-11e7-8965-6af1553082db.png)

![standard A](https://cloud.githubusercontent.com/assets/7019862/23596598/109ec732-0267-11e7-917d-d9651a8184e7.png)

#### 一句话概括：不管有没有，我都要新建

### singleTop

在这种模式下，我们设计两个Activity 分别是A和B，当在A下调用startActivity A 的时候，没有启动新的Activity，而是继续显示A，

|     |
| --- |
|  A  |
| 栈底 |

但是如果我们调用startActivity B 时会启动 B，此时在Task中Activity如下：

|     |
| --- |
|  B  |
|  A  |
| 栈底 |

此时如果此时点击startActivity A，则会继续实例化一个新的Activity A，此时的Task为：

|     |
| --- |
|  A  |
|  B  |
|  A  |
| 栈底 |

所以我们得出结论，处于栈顶的Activity不允许实例化自身，但是可以实例化其他的Activity。

#### 一句话概括：不允许挨着的重复

### singleTask

我们设计如下界面：

![](https://cloud.githubusercontent.com/assets/7019862/23597379/617d77ce-026d-11e7-9243-6abb664e8b2d.png)

启动过程如下 A -> B -> A -> B -> B -> A

我们来看一下Task的情况：

|     |
| --- |
|  A  |
| 栈底 |

执行了这么多步骤，为什么就只剩下A了呢？

原来在singleTask模式下，Android会检查整个Task中的Activity是否包含要启动的Activity，如果有则移除它之上的，将它放在Task的栈顶，所以就会出现上面的情况了。
>在这个步骤下，我们发现B被实例化了很两次，为什么呢？
>
>第一次B被实例化以后，我们调用了start A，因为B在A之上，所以B被移除栈，接着我们再调用start B，因为B已经不在栈中，所以系统又会实例化B。

#### 一句话概括：如果要唤醒我，那么我上面的通通踢掉

### singleInstance

singleInstance模式比较简单，它的核心思想是，如果没有就创建，如果有就复用，每个Activity都拥有一个独立的Task，既每个Task只有一个Activity。

#### 一句话概括：I am the only one.