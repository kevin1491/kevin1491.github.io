---
layout:     post
title:      lager usage
subtitle:   erlang
date:       2019-12-02
author:     kevin1491
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - lager
    - erlang
---
- 
- - 代码来源 https://github.com/erlang-lager/lager

# 安装lager步骤

- 进入/code/erlang/dang/deps 删除原有的goldrush lager包

- 将goldrush.zip lager.zip 解压缩至deps

- 将app.config,rebar.config 移动至/code/erlang/dang目录下（原有的rebar.config备份一下，成功编译后可以删除）

- 启动脚本中加入 `-config app.config`参数

  app.config

  ```erlang
  [ {
           lager,
           [    {error_logger_redirect,false},%%both front and backend print 
                {handlers,[
                     {lager_file_backend,[{file,"error_logger.log"},{level,error}]}
                    ]}
            ]   
        }].
  ```

  

- vim /code/erlang/dang/src/qin/src/qin.erl在start()函数内加入以下代码：
  ```erlang
  ensure_started(syntax_tools),
  ensure_started(compiler),
  ensure_started(goldrush),
  ensure_started(lager),
  lager:start(),
  ```

# 使用方法
在需要打印日志的模块内加入
`-compile([{parse_transform, lager_transform}]).`

# 调用方法(参考io:format)
```erlang
lager:debug("").
lager:info("").
lager:warring("").
lager:error("").
```
# 日志查看
## 日志相关配置
`/code/erlang/dang/app.config`

## 后台日志
`tail -f /code/erlang/dang/error_logger.log`
## 前台console日志
`tail -f /code/erlang/dang/console_logger.log`