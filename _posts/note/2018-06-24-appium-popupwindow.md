---
category: note
tags: appium
title: appium对于Android中浮窗PopupWindow的定位
---

自动化测试时发现，安卓中某些浮窗无法通过uiautomator定位。在uiautomatorviewer中查看时，鼠标放置在相应位置会自动穿透过去，定位到下层窗口中的元素。这种情况下就算知道元素id，在自动化脚本中也无法查找到浮窗中的元素。

### 原因
上述提到的浮窗类型是`PopupWindow`，后来发现并不是所有的`PopupWindow`都无法定位，只有在其`focusable`属性设定为`false`时才会定位不到。

### 解决办法

#### 办法1：
既然只要`focusable`设定为`true`就可以定位，最好的办法当然是让开发把用到的`PopupWindow`的`focusable`设定为`true`.

#### 办法2：
但是某些情况下由于一些特殊的原因无法修改`PopupWindow.focusable`，比如我们的APP中有一个`PopupWindow`的`focusable`设定为`true`之后会导致无法通过点击空白处关闭浮窗。这种情况下只能通过UI设计时的位置来确定想要操作的元素的坐标，通过`TouchAction`对该坐标点进行相应操作。
