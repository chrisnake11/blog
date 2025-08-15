---
title: QScrollArea使用与悬浮滚动条
published: 2025-08-14T17:06:56Z
description: 'QScrollArea的使用，将子组件能够自适应QScrollArea的宽度。并且让滚动条需要时悬浮显示，避免原生滚动条影响子组件的宽度，导致组件宽度闪烁。'
image: ''
tags: [C++, ChatRoom, QT]
category: 'ChatRoom'
draft: false
---

# QScrollArea使用与悬浮滚动条

在构造ChatRoom聊天项目的QT客户端界面时，我需要对联系人和聊天记录进行滚动查看处理，因此使用了QScrollArea来实现这一功能。  

QScrollArea是QT中用于显示可滚动内容的组件，它可以包含一个子组件(QWidget)，并提供滚动条来查看子组件超出QScrollArea可视区域的部分。QScrollArea的界面结构如下：
+ QScrollArea
  + QScrollBar 
    + QWidget (子组件)
      + QVBoxLayout
        + Item(单个组件项)

## QScrollArea的使用与自适应子组件宽度

首先构建一个父`QWidget`用于存放这个`QScrollArea`，这样`QScrollArea`就会自动适配父`QWidget`的宽度。

随后，将`QScrollArea`和`子组件QWidget`的宽度都设置为`Expanding`，那么子组件就会与`QScrollArea`同步。

## 原生滚动条隐藏显示改变组件宽度的问题

原生的`QScrollArea`并不会自动隐藏滚动条。因此需要重载原生的`QScrollArea`类，重新实现滚动条的隐藏和实现的逻辑。

但是，如果这样设计，会出现**子组件`QWidget`的宽度在滚动条显示和隐藏时宽度发生闪烁**。因为滚动条会占据`QScrollArea`内部可视区域的宽度，从而导致子组件的宽度也随之变化。

**为了解决这个问题，我们可以自定义个`QScrollBar`类，作为`QScrollArea`的子组件，根据QT的Z-order机制，子组件会默认显示在父组件`QScrollArea`的上方。**

并且将这个自定义的`QScrollBar`与`QScrollArea`的滚动条的事件连接，使自定义QScrollBar能够同步`QScrollArea`的滚动条位置和范围。

```cpp
#ifndef CUSTOMSCROLLAREA_H
#define CUSTOMSCROLLAREA_H

#include <QScrollArea>
#include <QMouseEvent>
#include <QTimer>
#include <QPainter>
#include <QStyleOption>
#include <QResizeEvent>
/*
    自动隐藏滚动条的ScrollArea类
    当鼠标进入时显示滚动条，离开时隐藏滚动条。
*/
class CustomScrollArea : public QScrollArea
{
    Q_OBJECT

public:
    explicit CustomScrollArea(QWidget *parent = nullptr);

protected:
    void enterEvent(QEnterEvent *event) override;
    void leaveEvent(QEvent *event) override;

protected slots:
    // 延迟隐藏滚动条
    void hideScrollBars();
    // 悬浮滚动条同步方法
    void onOverlayScrollBarValueChanged(int value);
    void onVerticalScrollBarValueChanged(int value);
    void updateOverlayScrollBarRange(int min, int max);
    void showScrollBars();

protected:
    // 计时器，用于延迟隐藏滚动条
    QTimer *hideTimer;
    bool isScrollBarVisible;
    QScrollBar* _overlayScrollBar;
};

#endif // CUSTOMSCROLLAREA_H
```

```cpp
#include "CustomScrollArea.h"
#include <QScrollBar>

CustomScrollArea::CustomScrollArea(QWidget *parent)
    : QScrollArea(parent)
    , hideTimer(new QTimer(this))
    , isScrollBarVisible(false)
{
    // 初始隐藏滚动条
    setHorizontalScrollBarPolicy(Qt::ScrollBarAlwaysOff);
    setVerticalScrollBarPolicy(Qt::ScrollBarAlwaysOff);
    
    // 子组件，默认悬浮出现
    _overlayScrollBar = new QScrollBar(Qt::Vertical, this);
    _overlayScrollBar->setStyleSheet(R"(
        QScrollBar:vertical {
            background: transparent;
            width: 8px;
            border-radius: 4px;
            margin: 0px;
        }
        QScrollBar::handle:vertical {
            background: rgba(0, 0, 0, 0.3);
            border-radius: 4px;
            min-height: 20px;
        }
        QScrollBar::handle:vertical:hover {
            background: rgba(0, 0, 0, 0.5);
        }
        QScrollBar::add-line:vertical,
        QScrollBar::sub-line:vertical {
            border: none;
            background: none;
            height: 0px;
            subcontrol-position: bottom;
            subcontrol-origin: margin;
        }
        QScrollBar::add-page:vertical,
        QScrollBar::sub-page:vertical {
            background: none;
        }
    )");
    
    // 初始隐藏悬浮滚动条
    _overlayScrollBar->hide();

    // 连接信号进行同步(滚动条位置，范围)
    // 直接操作自定义滚动条，同步原生滚动条
    connect(_overlayScrollBar, &QScrollBar::valueChanged,
            this, &CustomScrollArea::onOverlayScrollBarValueChanged);
    // QScrollArea原生滚动条滚动，同步自定义滚动条
    connect(verticalScrollBar(), &QScrollBar::valueChanged,
            this, &CustomScrollArea::onVerticalScrollBarValueChanged);
    // 原生滚动条范围改变，同步自定义滚动条范围。
    connect(verticalScrollBar(), &QScrollBar::rangeChanged,
            this, &CustomScrollArea::updateOverlayScrollBarRange);

    // 确保内容可以自适应大小
    setWidgetResizable(true);

    // 设置定时器
    hideTimer->setSingleShot(true);
    hideTimer->setInterval(500); // 默认500毫秒后隐藏
    // 定时器超时，隐藏滚动条
    connect(hideTimer, &QTimer::timeout, this, &CustomScrollArea::hideScrollBars);
    // 启用鼠标跟踪
    setMouseTracking(true);
}

void CustomScrollArea::enterEvent(QEnterEvent *event)
{
    showScrollBars();
    QScrollArea::enterEvent(event);
}

void CustomScrollArea::leaveEvent(QEvent *event)
{
    // 延迟隐藏滚动条
    hideTimer->start();
    QScrollArea::leaveEvent(event);
}

void CustomScrollArea::showScrollBars()
{
    if (!isScrollBarVisible) {
        _overlayScrollBar->show();
        // 设置悬浮滚动条的位置和大小
        _overlayScrollBar->setGeometry(width() - 8, 0, 8, height());
        // 同步滚动条范围和值
        updateOverlayScrollBarRange(verticalScrollBar()->minimum(), verticalScrollBar()->maximum());
        _overlayScrollBar->setValue(verticalScrollBar()->value());
        isScrollBarVisible = true;
    }
    hideTimer->stop();
}

void CustomScrollArea::hideScrollBars()
{
    if (_overlayScrollBar) {
        _overlayScrollBar->hide();
    }
    isScrollBarVisible = false;
}

void CustomScrollArea::onOverlayScrollBarValueChanged(int value)
{
    // 防止无限递归，触发值改变事件
    verticalScrollBar()->blockSignals(true);
    verticalScrollBar()->setValue(value);
    verticalScrollBar()->blockSignals(false);
}

void CustomScrollArea::onVerticalScrollBarValueChanged(int value)
{
    // 防止无限递归，触发值改变事件
    _overlayScrollBar->blockSignals(true);
    _overlayScrollBar->setValue(value);
    _overlayScrollBar->blockSignals(false);
}

void CustomScrollArea::updateOverlayScrollBarRange(int min, int max)
{
    _overlayScrollBar->setRange(min, max);
    _overlayScrollBar->setPageStep(verticalScrollBar()->pageStep());
    _overlayScrollBar->setSingleStep(verticalScrollBar()->singleStep());
}
```