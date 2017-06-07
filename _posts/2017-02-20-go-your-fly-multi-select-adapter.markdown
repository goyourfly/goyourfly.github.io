---
layout: post
title: "Recycler的多选Adapter"
date: 2017-06-08
---

### 要实现的多功能：

1. 几行代码，在不修改原有Adapter的前提下实现Recycler的多选功能
2. 支持两种显示模式
	 - 左侧显示CheckBox表示选中，
	 - 改变颜色表示选中
3. 顶部利用PopupWindow显示Toolbar，当前选中多少个，取消按钮，确定按钮
4. Material风格