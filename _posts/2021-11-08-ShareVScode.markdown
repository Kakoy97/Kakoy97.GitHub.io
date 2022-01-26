---
layout:     post
title:      "分享平常使用的vscode插件"
date:       2021-11-08 23:04:00
author:     "Kakoy"
header-img: "img/2021-bg.jpg"
tags:
    - vscode
---


## Debugger for Unity
在我们日常开发中，常常需要用到调试模式。这个插件可以让我们使用vsCode来对unity进行调试。安装完插件以后，在.vscode文件夹下对launch.json文件进行修改



    "version": "0.2.0",
    "configurations": [
        {
            "name": "Lua-Remote",
            "type": "lua",
            "request": "attach",
            "runtimeType": "Unity",
            "localRoot": "${workspaceRoot}",
            "port": 7003,
            "printType": 1
        },
        {
            "name": "Cocos-Remote",
            "type": "lua",
            "request": "attach",
            "runtimeType": "Cocos2",
            "localRoot": "${workspaceRoot}",
            "port": 7003,
            "printType": 1
        },
        {
            "name": "Cocos-Launch",
            "type": "lua",
            "request": "launch",
            "runtimeType": "Cocos2",
            "localRoot": "${workspaceRoot}",
            "mainFile": "src/main.lua",
            "commandLine": [
                "-workdir ${workspaceRoot}",
                "-file src/main.lua"
            ],
            "port": 7003,
            "exePath": "Path/to/exe",
            "printType": 1
        },
        {
            "name": "Lua51-Launch",
            "type": "lua",
            "request": "launch",
            "runtimeType": "Lua51",
            "localRoot": "${workspaceRoot}",
            "mainFile": "main",
            "port": 7003,
            "printType": 1
        },
    
        {            
            "name": "Unity Editor",
            "type": "unity",
            "path": "/E:/3.3/Library/EditorInstance.json",
            "request": "launch"
        },

        {
            "name": "Go-Luach",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceRoot}",
            "env": {
              "PATH": "C:/Program Files/Go"
            },
            "args": []
          }

    ] 


<div>
    <br>第一个配置和倒数第二个配置都是关联Unity的,不同之处是第一个是lua的,倒数第二个则是C#的。倒数第二个配置使用时需要将path改成自己项目里EditorInstance.json文件的路径。
    <br>
    <br>配置完成之后,我们只需将Unity和我们的vscode关联在一起:打开UNity->打开菜单栏上的Edit -> Preferences ->External Tools -> External Script Editor 改成vscode即可。
</div>

## vscode-solution-explorer
这个插件给VS Code增加了解决方案tab, 支持新建解决方案、新建工程、添加引用、Nuget包，并且我们可以直接在本地Main方法里运行c#代码

## Auto-Using for C#
这个插件会自动帮我引入命名空间，这样就不需要每次写代码去自己点灯泡引入了。不过有一点不好的是，有时候会引入错入的命名空间。>_<