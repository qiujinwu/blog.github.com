---
title: 使用plantuml画UML图
date: 2018-05-25 21:14:21
tags:
  - uml
categories:
  - 开发环境
---

官网<http://plantuml.com/>

主流的工具都支持[渲染](http://plantuml.com/running)

1. [intellij](http://plugins.intellij.net/plugin/?idea&id=7017)
2. [eclipse](http://plantuml.com/eclipse)
3. [sublime](https://github.com/jvantuyl/sublime_diagram_plugin)
1. [atom](https://atom.io/packages/plantuml)
1. [notepad++](https://github.com/brianmaher84/PlantUML_Notepad-_UDL)
1. [vscode](https://marketplace.visualstudio.com/items?itemName=okazuki.okazukiplantuml)
1. [netbeans](http://plugins.netbeans.org/plugin/49069/plantuml)

## 活动图/流程图
``` plantuml
@startuml
start
if (condition A) then (yes)
  :Text 1;
elseif (condition B) then (yes)
  :Text 2;
  stop
elseif (condition C) then (yes)
  :Text 3;
elseif (condition D) then (yes)
  :Text 4;
else (nothing)
  :Text else;
endif
stop
@enduml
```

## 状态图
``` plantuml
@startuml
scale 350 width
[*] --> NotShooting

state NotShooting {
  [*] --> Idle
  Idle --> Configuring : EvConfig
  Configuring --> Idle : EvConfig
}

state Configuring {
  [*] --> NewValueSelection
  NewValueSelection --> NewValuePreview : EvNewValue
  NewValuePreview --> NewValueSelection : EvNewValueRejected
  NewValuePreview --> NewValueSelection : EvNewValueSaved
  
  state NewValuePreview {
	 State1 -> State2
  }
  
}
@enduml
```

## 时序图
``` plantuml
@startuml
actor Bob #red
' The only difference between actor
'and participant is the drawing
participant Alice
participant "I have a really\nlong name" as L #99FF99
/' You can also declare:
   participant L as "I have a really\nlong name"  #99FF99
  '/

Alice->Bob: Authentication Request
Bob->Alice: Authentication Response
Bob->L: Log transaction
@enduml
```

## 类图
``` plantuml
@startuml
class Foo1 {
  You can use
  several lines
  ..
  as you want
  and group
  ==
  things together.
  __
  You can have as many groups
  as you want
  --
  End of class
}

class User {
  .. Simple Getter ..
  + getName()
  + getAddress()
  .. Some setter ..
  + setName()
  __ private data __
  int age
  -- encrypted --
  String password
}

@enduml
```

完整查看官网<http://plantuml.com/>

## 参考
1. <https://www.jianshu.com/p/e92a52770832>
2. <https://yuque.com/yuque/help/editor-puml>
3. <https://blog.csdn.net/zhangjikuan/article/details/53484558>