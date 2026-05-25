---
title: Avalonia Android Textbox输入法按换行消失
date: 2025-10-14 14:48:31
tags:
- 笔记
- avalonia
categories:
- [笔记,.NET,Avalonia]
---

在xaml后台初始化时加入

```cs
TextInputOptions.SetMultiline(textBox, true);
TextInputOptions.SetReturnKeyType(textBox, TextInputReturnKeyType.Return);
```
其中textBox为要修改的TextBox的name
