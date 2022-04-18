# Alfred工作流简单介绍


#### 简介

每当有人问到有什么 macOS 上的提升效率的神器，Alfred 都是我脑海里第一个蹦出来的答案。Alfred with Powerpack 更是强上加强。
几乎 mac 里的一切都可以通过简单的 `⌘ + space`（我的 Alfred 快捷键）获取到。
Alfred 的 workflows 是一个使我非常兴奋的功能。它能充分满足程序员自给自足的乐趣。

#### workflow基本组件介绍

alfred的工作流是由5个类别的组件构成，分别是： `Triggers`、`Inputs`、`Actions`、`Utilities`、`Outputs`。下面我会详细介绍下每个类别的一些常用组件以及作用。

##### Triggers

这个很明显就是Alfred的触发器，可以在特定场景下触发这个工作流，一般设置在workflow的开始部分。下面介绍一下常用的触发器。

- `HotKey`  通过快捷键方式触发。
- `Remote`  远程触发，这个一般是配合ios的客户端使用，但是由于ios端已经停止维护了，所以基本不怎么使用。
- `Snippet` 通过片段关键字触发，一般用于编写比较高级的代码片段。
- `External` 这是允许通过AppleScript脚本的方式来触发当前工作流，这样就可以将工作流和苹果自带的自动化工具有机地结合在一起。下面是一个使用展示：

    ``` AppleScript
    tell application id "com.runningwithcrayons.Alfred" to run trigger "{key words}" in workflow "{workflow-bundle-id}" with argument "test"
    ```

- `FileAction` 和 `Universal Action` 这两个触发器都是 `Universal Action` 这个功能的触发器，只不过一个是只能触发文件相关的操作，而另一个则是包含了所有能够触发的类型。
- `Contact Action`  当在 Alfred 的联系人查看器中选择通讯簿联系人字段时，创建一个联系人操作触发器以设置要在该字段上执行的替代操作，需要结合 `Feature -> Contact` 的自定义动作来使用。

##### Inputs

这一个类别就是其中最重要的一个环节，它包含了我们所有的输入方式，以及参数的传递。

- `Keywords` 关键字输入，输入关键字，并输入对应的参数进入工作流中，进行后续的处理。
- `File Filter` 文件输入，可以指定搜索的文件类型以及文件的范围，进行更精确的定位和操作。
- `Dictionary Filter` 字典过滤类型，可以用来进行字典的一系列操作，包括实现对应的生词本功能等等。
- `List Filter` 这个可以定义二级菜单，可以对功能进行整合，简化工作流的输入长度。
- `Script Filter` 这个是脚本过滤类型，通过关键字来触发对应的脚本执行，并将结果在搜索框进行展示，也是我们用的非常多的一个类型

##### Actions

这个类型就十分简单了，就是执行一些系统的动作，例如打开文件、执行系统命令等等

##### Utilities

这个主要是流程控制相关的操作，包括添加额外的参数、对参数进行简单的处理、对流程进行条件控制等等。

##### Outputs

这个模块主要就是输出相关了，可以将之前处理的结果输出到你需要的地方。包括但不限于 剪切板、mac的通知、文件甚至进行流程之间的互相调用等等。

#### workflow如何获取参数和参数回写

在一个程序中，最重要的就是输入和输出了，那么在alfred中，我们如何对参数进行处理呢。
其实alfred中自带了参数解析器，在脚本编辑界面直接通过 {query}或者{var:variable} 就可以获取到变量了，但是如何在脚本内部去获得呢，又如何将这写入到alfred中，并在后续流程使用呢。

##### 设置变量

有几种方法可以设置变量。最明显的是工作流配置表中的[环境变量表](https://www.alfredapp.com/help/workflows/advanced/variables/#environment)，并使用arg和vars实用程序。配置表并没有什么特殊的配置，但在args和vars实用程序中，您可以使用变量扩展宏：{query} 将变量展开到输入，也可以为自己的自定义变量使用{var:VARIABLE_NAME}的宏。这在上述Alfred帮助页面中详细描述。

而相对而言，更灵活且常用的是可以通过发出适当的JSON来通过脚本（即动态）的输出设置变量。设置变量如何取决于您是否使用脚本过滤器或运行脚本操作。

###### 在Run Script动作中

在alfred中，你可以使用print、echo、puts等等标准输出函数来输出结果，同时alfred会解析对应的json并进行相应的处理。同样的，我们的参数也会放在json中，json的格式如下：

``` json
{"alfredworkflow": 
    {
        "arg": "https://www.google.com",
        "variables": {"browser": "Google Chrome"}
    }
}
```

在Run Script中，根路径 `alfredworkflow` 是必须的， 如果没有，则无法解析对应的结果。通过这样的配置，在后续的流程中，你就可以通过 `{query}` 来获取 `arg` 的值，而通过 `{var:browser}` 来获取浏览器参数了。

###### 在Script Filter动作中

和Run Script不同，Script Filter的处理结果是会展示在alfred的结果集中的，并且会根据我们选择的值进行后续的流程处理。在这个动作中有三个不同级别的工作流量变量：`root level`、`item level`和`modifier level`。

- Root Level
  
  不论如何，根级变量始终传递到下游元素。如果设置了`return`，它们也会传递回相同的脚本过滤器，因此你可以使用根级变量来实现[进度条](https://www.alfredforum.com/topic/9718-progress-bar/)。下面是返回值：

  ``` json
    {
        "variables": {
            "browser": "Safari"
        },
        "items": [
            {
            "title": "Google",
            "arg": "https://www.google.com"
            },
            {
            "title": "Reddit",
            "arg": "https://reddit.com",
            "variables": {
                "browser": "Google Chrome"
            }
            }
        ]
    }
  ```

- Item-level variables
  
  项目级别变量仅在使用当前设置的项目时通过下游，并且它们会覆盖了根级别变量。例如默认使用safari，而百度使用chrome。

  ``` json
    {
        "variables": {
            "browser": "Safari"
        },
        "items": [
            {
            "title": "Google",
            "arg": "https://www.google.com"
            },
            {
            "title": "Reddit",
            "arg": "https://www.baidu.com",
            "variables": {
                "browser": "Google Chrome"
            }
            }
        ]
    }
  ```

- Modifier-level variables

  修改级别的变量，只有当按下对应的修饰健时，才会向下游传递，他们会替换项目级别的变量。例如默认使用safari，百度使用chrome，而当按下cmd时也使用chrome。

  ``` json
    {
        "variables": {
            "browser": "Safari"
        },
        "items": [
            {
            "title": "Google",
            "arg": "https://www.google.com",
            "mods": {
                "cmd": {
                "variables": {
                    "browser": "Google Chrome"
                }
                }
            }
            },
            {
            "title": "Reddit",
            "arg": "https://www.baidu.com",
            "variables": {
                "browser": "Google Chrome"
            }
            }
        ]
    }
  ```

##### 使用变量

所以你已经设置了一些变量，现在你想用它们。在arg和vars或Filter Utilities等Alfred元素中，您使用上述{var:VARIABLE_NAME}宏。非常简单。

在我们自己的代码中，它会变得更加复杂。首先，{var:VARIABLE_NAME}不适用于运行脚本操作、脚本过滤器或者Alfred中的任何其他脚本框。

当Alfred运行代码时，它不会使用{var：...}，而是获取到任何工作流量变量，并将其设置为脚本的环境变量。再次使用上面的示例，Alfred将根据输入的“https://www.google.com”，将环境变量浏览器设置为Safari或GoogleChrome。如何获取环境变量取决于你所使用的语言。

- bash
  
  bash获取环境变量非常简单，通过 `$VARIABLE_NAME` 即可。

  ``` bash
    echo $browser
  ```

- Python

  python的话需要使用 `os.environ` 或者 `os.getenv('VARIABLE_NAME')` 来获取。

  ``` python
    import os
    browser = os.environ['browser']
    # 或者
    browser = os.getenv('browser')
  ```

- AppleScript
  
  使用 `system attribute`

  ``` applescript
    set theBrowser to (system attribute "browser")
  ```

- JavaScript (JXA)
  
  使用 `$.getenv()`

  ``` javascript
    ObjC.import('stdlib');
    var browser = $.getenv('browser');
  ```

- PHP
  
  使用 `getenv()`

  ``` php
    $browser = getenv('browser');
    // 或者
    $browser = $_ENV['browser'];
  ```

如果实例中没有你经常使用的语言可以自行通过搜索引擎了解。

当然了 alfred 已经出来了这么多年，其实已经有大量的大佬给我们封装了非常好用的sdk，从而不需要我们自己再去进行底层逻辑的处理，这里我简单列几个：

- [python](https://github.com/deanishe/alfred-workflow)
- [php](https://github.com/joetannenbaum/alfred-workflow)
- [golang](https://github.com/deanishe/awgo)
