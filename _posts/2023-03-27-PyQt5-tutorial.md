---
layout: post
title: PyQt5 fundation
excerpt: "PyQt5 is a great tool for gui, so this is a learning note for pyqt5"
date: 2023-03-27 21:35:05
tags: ['pyqt5']
category: learning
---

<style>
.wrap {
    max-width: 900px;
}
p {
    font-family: sans-serif;
    font-size: 15px;
    font-weight: 300;
    overflow-wrap: break-word; /* allow wrapping of very very long strings, like txids */
}
.post pre,
.post code {
    background-color: #fafafa;
    font-size: 13px; /* make code smaller for this post... */
}
pre {
 white-space: pre-wrap;       /* css-3 */
 white-space: -moz-pre-wrap;  /* Mozilla, since 1999 */
 white-space: -pre-wrap;      /* Opera 4-6 */
 white-space: -o-pre-wrap;    /* Opera 7 */
 word-wrap: break-word;       /* Internet Explorer 5.5+ */
}
</style>

# pyqt5学习指南

PyQt5 GUI学习，参考知乎的一个系列文章[快速掌握PyQt5](https://zhuanlan.zhihu.com/p/75673557)，做笔记记录
## 1. pyqt5准备工作
pyqt5安装主要包括pyqt5，这里为了python环境干净，使用了anaconda工具
```python
pip install pyqt5
```
### 1.1 简单入门程序
下面是一个简单的窗口程序，显示hello world

```python
import sys
from PyQt5.QtWidgets import QApplication, QLabel

if __name__ == '__main__':
    app = QApplication(sys.argv)
    label = QLabel('hello world')
    label.show()
    sys.exit(app.exec())
```
1. 先创建一个实例化QApplication
2. 实例化一个QLabel控件展示文字或图片，可以创建时候就构造传入，也可以先实例化，再调用setText()方法来设置文本
3. 调用show方法显示控件
4. app.exec_()是执行应用，让应用开始运转循环，直到窗口关闭返回0给sys.exit()，退出整个程序



### 1.2 信号与槽
信号(signal)与槽(slot)机制，PyQt5中各个对象间或各个对象自身就是通过信号与槽机制来相互通信的。
信号视作裁判鸣枪，而用于行动的槽函数则视作选手开跑，当裁判鸣枪后(即信号发出)，选手就开始往前跑(槽函数启动)
#### 1.2.1 一个信号连接一个槽
```python
import sys
from PyQt5.QtWidgets import QApplication, QWidget, QPushButton

class Demo(QWidget):
    def __init__(self):
        super(Demo, self).__init__()
        self.button = QPushButton('start', self)
        self.button.clicked.connect(self.change_text)

    def change_text(self):
        print('change text')
        self.button.setText('Stop')
        self.button.clicked.disconnect(self.change_text)

if __name__ == '__main__':
    app = QApplication(sys.argv)
    demo = Demo()
    demo.show()
    sys.exit(app.exec_())
```
1. 该类继承QWidget，QWidget看作是一种毛坯房，而我们往其中放入QPushButton、QLabel等控件就相当于在装修这间毛坯房。类似的毛坯房还有QMainWindow和QDialog
2. 实例化一个QPushButton，因为继承于QWidget，所以self不能忘了(相当于告诉程序这个QPushButton是放在QWidget这个房子中的)
3. 连接信号与槽函数。self.button就是一个控件，clicked(按钮被点击)是该控件的一个信号，connect()即连接，self.change_text即下方定义的函数(我们称之为槽函数) 
   通用的公式可以是：widget.signal.connect(slot)
4. 将按钮文本从‘Start’改成‘Stop’
5. 信号和槽解绑，解绑后再按按钮你会发现控制台不会再输出‘change text’，如果把这行解绑的代码注释掉，你会发现每按一次按钮，控制台都会输出一次‘change text’
6. 实例化Demo类
7. 使demo可见，其中的控件自然都可见(除非某控件刚开始设定隐藏)

#### 1.2.2 按钮改变文字(一个信号连接一个槽)
```python
import sys
from PyQt5.QtWidgets import QApplication, QWidget, QPushButton

class Demo(QWidget):
    def __init__(self):
        super(Demo, self).__init__()
        self.button = QPushButton('Start', self)
        self.button.pressed.connect(self.change_text)
        self.button.released.connect(self.change_text)

    def change_text(self):
        if self.button.text() == 'Start':
            self.button.setText('Stop')
        else:
            self.button.setText('Start')

if __name__ == '__main__':
    app = QApplication(sys.argv)
    demo = Demo()
    demo.show()
    sys.exit(app.exec_())

```
这个例子是用多个信号signal连接一个槽，qpushbutton的pressed信号和released信号，当他鼠标按下不放，就是pressed的状态，松开就是released的状态
#### 1.2.3 一个信号连接另外一个信号
```python
import sys
from PyQt5.QtWidgets import QApplication, QWidget, QPushButton


class Demo(QWidget):
    def __init__(self):
        super(Demo, self).__init__()
        self.button = QPushButton('Start', self)
        self.button.pressed.connect(self.button.released)  # 1
        self.button.released.connect(self.change_text)     # 2

    def change_text(self):
        if self.button.text() == 'Start':
            self.button.setText('Stop')
        else:
            self.button.setText('Start')


if __name__ == '__main__':
    app = QApplication(sys.argv)
    demo = Demo()
    demo.show()
    sys.exit(app.exec_())
这里的代码pressed信号连接到released信号，released信号连接到change_text槽函数，和1.2.2中效果一样
```
#### 1.2.4 一个信号连接多个槽
```python
import sys
from PyQt5.QtWidgets import QApplication, QWidget, QPushButton


class Demo(QWidget):
    def __init__(self):
        super(Demo, self).__init__()
        self.resize(300, 300)                                   # 1
        self.setWindowTitle('demo')                             # 2
        self.button = QPushButton('Start', self)
        self.button.clicked.connect(self.change_text)
        self.button.clicked.connect(self.change_window_size)    # 3
        self.button.clicked.connect(self.change_window_title)   # 4

    def change_text(self):
        print('change text')
        self.button.setText('Stop')
        # self.button.clicked.disconnect(self.change_text)

    def change_window_size(self):                               # 5
        print('change window size')
        self.resize(500, 500)
        # self.button.clicked.disconnect(self.change_window_size)

    def change_window_title(self):                              # 6
        print('change window title')
        self.setWindowTitle('window title changed')
        # self.button.clicked.disconnect(self.change_window_title)


if __name__ == '__main__':
    app = QApplication(sys.argv)
    demo = Demo()
    demo.show()
    sys.exit(app.exec_())
```
这个的clicked信号连接了三个槽函数
#### 1.2.5 自定义信号
```python
import sys
from PyQt5.QtCore import pyqtSignal                             # 1
from PyQt5.QtWidgets import QApplication, QWidget, QLabel


class Demo(QWidget):
    my_signal = pyqtSignal()                                    # 2

    def __init__(self):
        super(Demo, self).__init__()
        self.label = QLabel('Hello World', self)
        self.my_signal.connect(self.change_text)                # 3

    def change_text(self):
        if self.label.text() == 'Hello World':
            self.label.setText('Hello PyQt5')
        else:
            self.label.setText('Hello World')

    def mousePressEvent(self, QMouseEvent):                     # 4
        self.my_signal.emit()                                   


if __name__ == '__main__':
    app = QApplication(sys.argv)
    demo = Demo()
    demo.show()
    sys.exit(app.exec_())
```
1. 需要先导入pyqtSignal
2. 实例化一个自定义的信号；
3. 将自定义的信号连接到自定义的槽函数上；
4. mousePressEvent()方法是许多控件自带的，这里来自于QWidget。该方法用来监测鼠标是否有按下。现在鼠标若被按下，则会发出自定义的信号。
# 参考文献
[【快速掌握PyQt5】](https://zhuanlan.zhihu.com/p/75673557)
[【第二章 信号与槽】](https://zhuanlan.zhihu.com/p/75520089)
