---
title: 'WARNING: Secure coding is automatically enabled for restorable state.'
date: '2025-01-23T16:59:30+09:00'
author: coherence
categories:
  - Memo
tags:
  - macOS
  - Python
aliases:
  - /archives/27
---

## 环境

・macOS Sonoma  
・pycharm  
・python 8

## 画面 

```python
<mark class="has-inline-color has-vivid-purple-color" style="background-color:rgba(0, 0, 0, 0)">from turtle import Turtle, Screen
from paddle import Paddle
from ball import Ball
import time
screen = Screen()
screen.setup(width=800, height=600)
screen.bgcolor("black")
screen.title("Pong")
screen.tracer(0)</mark>
```

## 报错

WARNING: Secure coding is automatically enabled for restorable state! However, not on all supported macOS versions of this application. Opt-in to secure coding explicitly by implementing NSApplicationDelegate.applicationSupportsSecureRestorableState:  

## 解决

把pycharm从python 3.9更新到了python 3.11成功解决。
