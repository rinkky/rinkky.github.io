---
category: note
tags: appium
title: appium中的点击、长按、滑动等屏幕操作
---

### 简述

appium中的屏幕操作是通过`TouchAction`类来实现的，屏幕操作包括以下几种：

`（以Python为例。其他语言的方法名不尽相同，但原理一样)`

主要：

>`tap()`：轻击，就是普通的点击操作。
>
>`press()`：按压，可以实现点击效果，但与`tap`相比，`press`可以实现一些后续操作。
>
>`long_press()`：长按。
>
>`move_to()`：移动。与我们实际用手指操作手机一样，移动之前需要先通过按压定位。所以移动一般是跟在按压或长按操作之后。

其他：
>`release()`结束操作。
>
>`wait()`等待，设定等待时间/操作时间。
>
>`perform()`执行操作，见下面例子。

### 操作链

通过把以上操作串联起来，形成一个操作链，如滑动需要先`press`再`move_to`然后`release`. 例：

```Python
from appium.webdriver.common.touch_action import TouchAction

def touch_test():
    actions = TouchAction()
    actions.press(x=100, y=300) # 类似手指按压屏幕的(100, 300)位置
    actions.move_to(x=100, y=100) # 移动手指到达(100, 100)位置
    actions.release() # 放开手指
    actions.perform() # 将上面3个操作串联起来，依次执行
```
上面的例子所执行的操作为：
1. 将手指放到(100, 300)位置
2. 移动手指到(100, 100)位置
3. 放开手指

注意`perform()`，只有在`perform()`的时候，上面的操作链才会真正执行。

另外，由于`TouchAction`中每个操作都会返回对象本身，所以上面的操作链也可以直接写成：
```Python
    actions.press(x=100, y=300).move_to(x=100, y=100).release().perform()
```

### 定位

上面的例子使用的是坐标定位，也可以使用元素定位。如：

手指从元素`el_0`的位置滑动到元素`el_1`的位置。
``` Python
    actions.press(el_0).move_to(el_1).release().perform()
```
注意：这里的位置指的是**滑动之前**元素的位置。

### 控制滑动速度

我们在操作触屏设备的时候有快速滑动和慢速滑动两种。快速滑动在快速翻页时用到，滑动行为结束后页面会根据惯性继续滑动一段距离；大多数情况下我们使用的都是慢速滑动。

但是上述操作链只能实现快速滑动，无法控制滑动的速度。想要控制滑动速度，需要在操作链中加入`wait()`.

通过实际测试，得出了下面的结论：
1. move_to的速度是根据其之前的操作时间来定的
2. press的时间是根据press之后的wait时间来定的
3. 操作链`press().wait().move_to()`中的move_to操作会在press后立即执行，不需要等待wait结束

不知道这几个结论说的清不清楚，不清楚也不用在这纠结，直接看例子：

``` Python
    actions.press(el_0).wait(1000).move_to(el_1).release().perform()
```
这个操作链的意思是，手指触摸`el_0`，然后从`el_0`的位置滑动到`el_1`的位置，整个过程用时1000ms. 这样就实现了慢速滑动。


另外，也可以通过`long_press`实现慢速滑动
```Python
    actions.long_press(el_0).wait(1000).move_to(el_1).release().perform()
```

### appium python client的内置方法

appium的python客户端的driver中内置了几个方法，可以直接使用：

>`scroll(el0, el1)`：快速滑动；通过元素定位滑动的起点和终点。
>
>`drag_and_drop(el0, el1)`：慢速滑动，通过元素定位滑动的起点和终点。
>
>`swipe(x, y, time)`：滑动，可设定滑动的时间；通过坐标定位滑动的起点和终点。

例：
```Python
import appium
#...
    driver = appium.webdriver.Remote(url, opts)
    el0 = driver.find_element_by_id(id0)
    el0 = driver.find_element_by_id(id1)
    driver.drag_and_drop(el0, el1)
```
`drag_and_drop`字面意思是拖拽和释放，触摸操作中的拖拽和释放操作，其实就是慢速滑动(或许叫做定位滑动更合适，总之，就是在滑动结束后不让页面根据惯性继续移动)。
