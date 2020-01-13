---
title: 使用landslide生成幻灯片
date: 2020-01-11 00:00:00
categories:
 - 开发环境
---

官网 <https://github.com/adamzap/landslide>

# 安装
```
pip install landslide
```
## 从源码安装
``` bash
$ git clone https://github.com/adamzap/landslide.git
$ cd landslide
$ python setup.py build
$ sudo python setup.py install
```

# 快捷键
+ Press h to toggle display of help
+ Press left arrow and right arrow to navigate
+ Press t to toggle a table of contents for your presentation. Slide titles are links
+ Press ESC to display the presentation overview (Exposé)
+ Press n to toggle slide number visibility
+ Press b to toggle screen blanking
+ Press c to toggle current slide context (previous and next slides)
+ Press e to make slides filling the whole available space within the document body
+ Press S to toggle display of link to the source file for each slide
+ Press '2' to toggle notes in your slides (specify with the .notes macro)
+ Press '3' to toggle pseudo-3D display (experimental)
+ Browser zooming is supported

# 使用
```
landslide a.md -i -o > name_you_like.html
```

# sample
``` markdown
# Landslide

---

# Overview

Generate HTML5 slideshows from markdown, ReST, or textile.

![python](http://i.imgur.com/bc2xk.png)

Landslide is primarily written in Python, but it's themes use:

- HTML5
- Javascript
- CSS

---

# Code Sample

Landslide supports code snippets

    !python
    def log(self, message, level='notice'):
        if self.logger and not callable(self.logger):
            raise ValueError(u"Invalid logger set, must be a callable")

        if self.verbose and self.logger:
            self.logger(message, level)
```

