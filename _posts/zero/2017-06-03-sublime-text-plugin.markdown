---
category: from 0 to 1
tags: sublime-text-plugin python
title: Sublime Text 3 插件开发从0到1
---

本文介绍sublime插件开发的最基本知识。

## 开始

先看一个官方默认实例。

在Sublime Text菜单中依次选择`Tools >> Developer >> New Plugin`, 此时Sublime会自动打开一个标签页。如下：

```
import sublime
import sublime_plugin


class ExampleCommand(sublime_plugin.TextCommand):
    def run(self, edit):
        self.view.insert(edit, 0, "Hello, World!")

```

我们先保存一下这个文件，保存为`Packages/example/example.py`.

这是一个名字叫`ExampleCommand`的类，继承自`sublime_plugin.TextCommand`.  它会被解析成一个叫做`example`的命令(command)，运行命令时，会执行运行`run()`.

理解command: 现在从菜单中打开`Preferences >> Key Bindings`, 查看Default里面的内容，可以看到类似这样的结构`{ "keys": ["ctrl+c"], "command": "copy" }`, 我们平时用的撤销、复制等操作其实都是command. 

我们按照这种格式在User中添加并保存:

```
{"keys":["alt+x"], "command":"example"},
```

关掉快捷键设定窗口，回到`example.py`中，按`alt+x`执行`example`命令。可以看到在文档开头插入了"Hello, World!"

## API

至此，已经完成了简单的插件编写和执行。现在来回顾一下上面例子中的几个要点。

- `sublime_plugin.TextCommand`: 文本编辑相关的命令都继承自这个类。本例中是`ExampleCommand`, 注意，**类名要以Command结束，Command之前的部分，除了首字母外不能有其它大写。**

- `run(self, edit)`: 执行命令时会运行这个函数，这是固定格式，无需修改。其中的`edit`会在编辑时用到，`edit`用法很固定，这个先不管。

- `self.view.insert(edit, point, content)`: point是int类型，指定编辑器中的一个坐标。 
  所有插入、删除、选取等操作都通过`self.view`调用。这个例子中是在point位置插入了content.

写插件时候会需要查看更多API，用到的时候可以参考 [官方API文档](https://www.sublimetext.com/docs/3/api_reference.html)。因为官方文档比较简陋，为了帮助理解，这里介绍一些有必要预先知道的概念：

#### 基本概念
- `point`概念，实际是一个整数，代表编辑器中的一个位置。可以通过`sublime.View.rowcol(point)`将point转化为行列信息(int, int).  或通过`sublime.View.text_point(row, col)`将行列信息转化为point.

- `sublime.Region`类，表示编辑器中的一个区域。有两个int类型的属性a和b. `Region.a`表示区域开始的点，`Region.b`表示区域结束的点。 

- `sublime.Selection`类，表示选择的区域。函数`sublime.View.sel()`返回的类型就是`Selection`, 表示目前选中的所有区域。`Selection.add(Region)`方法可以添加选择区域。

#### 查找相关

编写插件时，经常要对对编辑器中的内容进行查找，如找到函数的所有参数。查找时可以用正则表达式，但是由于同一语言同一功能的写法并不单一，而且经常有注释的干扰，如果用正则表达式可能会非常复杂。好在sublime提供了一些其他的查找方式。

- `sublime.View.classify(point)`函数， 对文档中point进行分类（如行首、行尾等），所有分类列举如下，基本看名字就能知道是什么意思：

```
sublime.CLASS_WORD_START            # 1
sublime.CLASS_WORD_END              # 2
sublime.CLASS_PUNCTUATION_START     # 4
sublime.CLASS_PUNCTUATION_END       # 8
sublime.CLASS_SUB_WORD_START        # 16
sublime.CLASS_SUB_WORD_END          # 32
sublime.CLASS_LINE_START            # 64
sublime.CLASS_LINE_END              # 128
sublime.CLASS_EMPTY_LINE            # 256
```

`sublime.View.find_by_class(point, forwoard, classes)`中的class指的就是这个。

- `selector`概念，选择器。可以通过选择器快速找到相应Region. 如`find_by_selector("meta.function.python")`中的参数"meta.function.python"就是一个选择器，可以快速找到python函数的区域。又如"comment"选择器可以快速找到注释区域。关于选择器的更详细信息，一会再说。更详细的选择器信息可以查看 [官方Scope Naming文档](https://www.sublimetext.com/docs/3/scope_naming.html)。

以上这些概念理解了，基本就可以根据官方API进行插件编写了。不理解也没关系，有个印象就行，用到的时候再回来看。下面继续完成我们的第一个插件的开发。

## 开发

我们要写一个为所有python函数添加注释的插件。

主要步骤其实只有两个：

1. 找到需要添加注释的地方；
2. 在相应的位置插入注释。

在`example.py`中我们已经了解到如何插入内容，即通过`self.view.insert(edit, point, content)`进行插入。接下来要做的就是找到需要插入的位置。

#### 找到位置

上面已经提到过通过selector(选择器)查找会简单很多。我们可以通过选择器`"punctuation.section.function.begin.python"`找到所有函数头结束的地方。

将代码修改如下：

```
    def run (self, edit):
        rgs = self.view.find_by_selector(
            "punctuation.section.function.begin.python"
        )
        
        self.view.sel().add_all(rgs) #选中所有找到的内容

```

这段代码通过选择器找到了所有函数头结束的位置，得到了一个Region列表。为了能看到效果，我们对这些Region进行了选中操作。按下`alt+x`查看运行效果。

我们可以看到run函数后面的冒号被选中了。添加几个测试用的函数，继续看效果。

```
    def test(a, b, c=(1,2)):
        pass
        
    #def test1():
    #    pass

```

添加上面代码后，按下`alt+x`可以看到run和test的冒号被选中了，被注释掉的test1未被选中。正如我们的期望。

#### 插入

既然找到了这些位置，我们就可以在这些位置插入注释。正如Hello World那样，我们用`self.view.insert()`进行插入操作。继续修改run()函数：

```
def run (self, edit):
    rgs = self.view.find_by_selector(
        "punctuation.section.function.begin.python"
    )
    
    for rg in rgs:
        self.view.insert(edit, rg.b, '\n\"\"\"\ncomments here\n\"\"\"')
```

关于`rg.b`我们已经在上面介绍Region时提到过。

运行之后效果类似下面这样：

```
...
    def test(a, b, c=(1,2)):
"""
comments here
"""
        pass
```

注释有了，但是并不是我们想要的效果，因为没有缩进。想要正确的缩进，我们就得知道函数起始位置的缩进量。

#### 继续查找位置

针对上面每个找到的`Region`, 继续找到其对应的函数起始位置。假设函数头只有一行，通过下面的方法找到缩进量

```
#rg.a所在的行
rg_line = self.view.line(rg.a)

#获取行首坐标
point1 = rg_line.a           

#找到point1之后第一个单词开始的地方。即函数定义开始的地方
point2 = self.view.find_by_class(
    point1, True, sublime.CLASS_WORD_START 
)
```

行首位置找到了，该行第一个单词开始的位置找到了，两个位置的间隔也就是函数缩进量。有了缩进量，在插入注释时添加缩进就可以了。

修改代码如下：

```
def run(self, edit):
    rgs = self.view.find_by_selector(
        "punctuation.section.function.begin.python"
    )
    for rg in rgs:
        rg_line = self.view.line(rg.a) 
        point1 = rg_line.a           
        point2 = self.view.find_by_class(
            point1, True, sublime.CLASS_WORD_START 
        )
        
        indentation = point2 - point1
        
        self.view.insert(edit, rg.b, 
            '\n{0}\"\"\"\n{0}comments here\n{0}\"\"\"'.format(
                indentation * " "
            )
        )
```

运行后得到形如下面的结果：

```
...
    def test(a, b, c=(1,2)):
    """
    comments here
    """
        pass
```

注释位置还是不太对，需要向后继续缩进一次。但是缩进一次是几个空格的位置，需要读取sublime的`tab_size`值来确定。python标准是4个空格代表一次缩进，但是这里我们为了演示，所以继续读取sublime的`tab_size`设置。

#### 读取设置

根据API文档，`sublime.View.settings()`可以获取设置，返回类型是`sublime.Settings`. 修改后的代码如下：

```
def run(self, edit):
    rgs = self.view.find_by_selector(
        "punctuation.section.function.begin.python"
    )
    for rg in rgs:
        rg_line = self.view.line(rg.a) 
        point1 = rg_line.a           
        point2 = self.view.find_by_class(
            point1, True, sublime.CLASS_WORD_START 
        )
        
        indentation = point2 - point1
        tab_size = self.view.settings().get("tab_size")
        
        self.view.insert(edit, rg.b, 
            '\n{0}\"\"\"\n{0}comments here\n{0}\"\"\"'.format(
                (indentation + tab_size) * " "
            )
        )
```

运行后得到形如下面的结果：

```
...
    def test(a, b, c=(1,2)):
        """
        comments here
        """
        pass
```

至此，我们已经初步完成了给函数添加注释的插件。

还有几个需要完善的地方：

- 我们并不想每次都给所有函数添加一次注释，所以需要根据光标位置推测用户意图，找到正确的需要注释的函数。
- 还需要给参数和返回值添加注释，最后得到的效果如下：

```
...

    def test(a, b, c=(1,2)):
        """this is test
        
        Args:
            a -- bla bla bla
            b -- bla bla
            c -- bla bla bla bla bla bla bla bla bla
        Returns: bla bla
        """

...

```

剩下的部分应该可以看着官方文档自己完成了。

## 总结

- 编辑相关的命令继承自`sublime_plugin.TextCommand`, 编辑类的命名规则为`XxxxCommand`, 首字母大写，其他字母不能大写，以Command结尾。
- `point`概念，一个整数，代表编辑器中的一个位置。
- `sublime.Region`类，表示编辑器中的一个区域。
- `sublime.View.find_by_class(point, forwoard, classes)`根据位置分类查找，如找到行首、找到单词开头等。
- `find_by_selector(selecor)`根据语法选择器查找，selector是一个字符串，多个选择器可以通过逗号连接成一个。更多语法选择器可以查看 [官方Scope Naming文档](https://www.sublimetext.com/docs/3/scope_naming.html)。Github上有sublime中各语言的语法定义文件，[这是Python的](https://github.com/sublimehq/Packages/blob/master/Python/Python.sublime-syntax)。
- [官方API文档位置](https://www.sublimetext.com/docs/3/api_reference.html)。