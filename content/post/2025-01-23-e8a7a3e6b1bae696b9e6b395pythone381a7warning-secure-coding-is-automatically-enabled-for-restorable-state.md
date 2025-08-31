---
title: '[解決方法]PythonでWARNING: Secure coding is automatically enabled for restorable state.'
date: '2025-01-23T16:59:30+09:00'
author: coherence
categories:
  - Memo
tags:
  - macOS
  - Python
  - markdown
  - css
  - html
aliases:
  - migrate-from-jekyl
  - /archives/27
---

## 環境

・macOS Sonoma  
・pycharm  
・python 8

## 事象 

```
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

## **エラーが出た**

WARNING: Secure coding is automatically enabled for restorable state! However, not on all supported macOS versions of this application. Opt-in to secure coding explicitly by implementing NSApplicationDelegate.applicationSupportsSecureRestorableState:  
みたいな警告メッセージが出てきて、画面が飛んでくるはずだが、飛んでいなかった。

## **解決方法**

pycharmでpython 3.9をpython 3.11に更新した。

## 結果

治った、warningメッセージが出てこないし、画面も起動できるようになった。