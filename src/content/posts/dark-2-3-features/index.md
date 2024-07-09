---
title: "Dart 2.3 新特性"
published: 2019-04-24
tags: [Flutter,Dart]
category: 技术
---

用 Flutter 写 UI 的时候常能碰到一种情况，比如写一个列表：
````
Column(
    children:[
        Text("1"),
        Text("2"),
        Text("3"),
    ]
)
````
然后有一个需求：根据一定的条件是否显示 3 ，你可能首先想到的是这样：
````
Column(
    children:[
        Text("1"),
        Text("2"),
        if(condition) Text("3") else null,
    ]
)
````
令人失望的是，这种写法编译器不允许，再试试这样：
````
Column(
    children:[
        Text("1"),
        Text("2"),
        condition ? Text("3") : null,
    ]
)
````
恭喜你编译器没有报错，通过了！但是别高兴的太早，当你运行时应用报错，提示不应该传 null
这个时候，机智如我把 null 换成一个无关紧要的 Widget 如下：
````
Column(
    children:[
        Text("1"),
        Text("2"),
        condition ? Text("3") : Container(),
    ]
)
````
再运行，没有报错，一切 OK，
但是总感觉不够优雅，多加了一个没用的 Widget，有没有办法改进呢？有，请往下看：
````
Column(
    children:[
        Text("1"),
        Text("2"),
        condition ? Text("3") : null,// 这里仍然设置为 null
    ].where((w) => != null).toList();
)
````
我们用一个语法糖，过滤掉所有的 null，这么做比上面那种方法稍微强一点，但是要多遍历一遍列表，还是不够完美

在 Dart 2.3 以后，我们有了更优雅选择，既最开始那种报错等写法 Dart 2.3 直接支持
````
Column(
    children:[
        Text("1"),
        Text("2"),
        if(condition) Text("3") , // 不需要写 else
    ]
)
````
Dart 2.3 还有一个很实用的功能，`...` 这个符号，从语意上理解就是把一个列表拆开
如果想把 list 动态加入到另外一个列表中，可以用 `...` 这个符号
实现如下：
````
var list = <Widget>[Text("3"),Text("4")];
Column(
    children:[
        Text("1"),
        Text("2"),
        ...list,
        if(condition) ...[Text("5"),Text("6")],
    ]
)
````