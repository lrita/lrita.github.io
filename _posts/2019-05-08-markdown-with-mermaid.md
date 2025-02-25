---
layout: post
title: 使用markdown画流程图/甘特图 - Mermaid
categories: [tool]
description: tool markdown flowchart sequence-diagram
keywords: tool markdown flowchart sequence-diagram
mermaid: true
---

在本博客中使用[mermaid](https://mermaidjs.github.io)将文字渲染成对应的流程图、甘特图等，是非常友好且利于版本管理的方式。具体语法可以参考连接中的文档。页面代码修改可以参考[本次提交](https://github.com/lrita/lrita.github.io/commit/5e7f86066148652727867c626752368f63cd4c6e)。

甘特图demo代码：

        ```mermaid
        gantt
        dateFormat  YYYY-MM-DD
        title Adding GANTT diagram functionality to mermaid
        section A section
        Completed task            :done,    des1, 2014-01-06,2014-01-08
        Active task               :active,  des2, 2014-01-09, 3d
        Future task               :         des3, after des2, 5d
        Future task2               :         des4, after des3, 5d
        section Critical tasks
        Completed task in the critical line :crit, done, 2014-01-06,24h
        Implement parser and jison          :crit, done, after des1, 2d
        Create tests for parser             :crit, active, 3d
        Future task in critical line        :crit, 5d
        Create tests for renderer           :2d
        Add to mermaid                      :1d
        ```

显示效果：

```mermaid
gantt
        dateFormat  YYYY-MM-DD
        title Adding GANTT diagram functionality to mermaid
        section A section
        Completed task            :done,    des1, 2014-01-06,2014-01-08
        Active task               :active,  des2, 2014-01-09, 3d
        Future task               :         des3, after des2, 5d
        Future task2               :         des4, after des3, 5d
        section Critical tasks
        Completed task in the critical line :crit, done, 2014-01-06,24h
        Implement parser and jison          :crit, done, after des1, 2d
        Create tests for parser             :crit, active, 3d
        Future task in critical line        :crit, 5d
        Create tests for renderer           :2d
        Add to mermaid                      :1d

```

流程图demo代码：

        ```mermaid
        graph TD;
                A-->B;
                A-->C;
                B-->D;
                C-->D;
        ```

显示效果：

```mermaid
graph TD;
        A-->B;
        A-->C;
        B-->D;
        C-->D;
```

时序图demo代码：

        ```mermaid
        sequenceDiagram
        participant Alice
        participant Bob
        Alice->John: Hello John, how are you?
        loop Healthcheck
                John->John: Fight against hypochondria
        end
        Note right of John: Rational thoughts <br/>prevail...
        John-->Alice: Great!
        John->Bob: How about you?
        Bob-->John: Jolly good!
        ```


显示效果：

```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    Alice->John: Hello John, how are you?
    loop Healthcheck
        John->John: Fight against hypochondria
    end
    Note right of John: Rational thoughts <br/>prevail...
    John-->Alice: Great!
    John->Bob: How about you?
    Bob-->John: Jolly good!
```