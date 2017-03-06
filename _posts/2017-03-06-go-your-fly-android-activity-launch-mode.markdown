---
title: "Activity 启动模式"
date: 2017-03-06
---

Android的每个UI都是依附于Activity之上，Activity相当于UI的容器，而我们经常的切换页面，实际上就是在不停的startActivity。但是在startActivity的过程中，经常会有些不一样的需求，比如说，当我打开Activity A的时候，我想关掉其他所有已经打开的Activity，或者说，我想在新的Task中启动Activity。
> Task是Activity的容器，是一个队列，每个Task可以有很多个Activity，我们可以指定一个Activity是否存放在一个新的Task，或者放在当前Task的栈顶。

Activity共有4种启动模式：

* standard
* singleTask
* singleInstance
* singleTop

### standard
在standard模式下，我们首先启动Activity A，接着我们在此启动Activity A，我们发现两个Activity A会同时存在，也就是说，在standard launchmode下，每次调用startActivity必然会启动一个新的Activity。

![standard A](https://cloud.githubusercontent.com/assets/7019862/23596549/92da8a98-0266-11e7-8965-6af1553082db.png)

![standard A](https://cloud.githubusercontent.com/assets/7019862/23596598/109ec732-0267-11e7-917d-d9651a8184e7.png)


|-----------------+------------+-----------------+----------------|
| Default aligned |Left aligned| Center aligned  | Right aligned  |
|-----------------|:-----------|:---------------:|---------------:|
| First body part |Second cell | Third cell      | fourth cell    |
| Second line     |foo         | **strong**      | baz            |
| Third line      |quux        | baz             | bar            |
|-----------------+------------+-----------------+----------------|
| Second body     |            |                 |                |
| 2 line          |            |                 |                |
|=================+============+=================+================|
| Footer row      |            |                 |                |
|-----------------+------------+-----------------+----------------|


|---
| Default aligned | Left aligned | Center aligned | Right aligned
|-|:-|:-:|-:
| First body part | Second cell | Third cell | fourth cell
| Second line |foo | **strong** | baz
| Third line |quux | baz | bar
|---
| Second body
| 2 line
|===
| Footer row