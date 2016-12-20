## 功能简介
本功能的实现，严重依赖nginx的upstream模块，同时使用Lua代码动态修改URI。

## 设计原则
1. 每个接口独立配置灰度发布策略
2. 选择器和调度器可以独立开发

## 背景
api manager自动生成upstream时，可以生成1-9个别名(可选)，比如upstream名称为dubboMonitor，那么api manager可能会生成若干个upstream

* dubboMonitor
* dubboMonitor1
* dubboMonitor2
* ......
* dubboMonitor9

特别的，1-8是给测试流水线使用的，9是专门为灰度发布保留。这样看来，9这个数字，在这个上下文环境里，是一个魔法数字，拥有特殊含义。


dubboMonitor >= dubboMonitor1 + ... + dubboMonitor8

dubboMonitor != dubboMonitor9 (**两者没有交集，需要在api manager里实现**)

## 数据流

Request  ==(selector)==>  Tag ==(scheduler)==>  Upstream

## 数据库设计
* 修改：api_version
* 新增：selector、scheduler


### [table] api_version
新增字段 is\_gray、selector_id、scheduler_id

| 字段名        | 类型           | 说明  |
| ------------- |:-------------:| :-----:|
| is\_gray | inter | 是否开启灰度发布功能 |
| selector_id | inter | 选择器，关联表selector | 
| scheduler_id | inter | 调度器，关联表scheduler |

### [table] selector
选择器(selector)可以从HTTP请求体中获取tag，传递给调度器(scheduler)

| 字段名        | 类型           | 说明  |
| ------------- |:-------------:| :-----|
| id | inter | 自动增长 |
| name | string | 选择器名称 | 
| source | tiny | 0代表从本地加载(load)，1代表从数据库加载(loadstring) |
| code | string | lua code | 
| update_time | timestamp | on update CURRENT_TIMESTAMP |

注意：name字段强耦合selector.lua代码文件

函数原型

```lua
function (ngx)
  return ngx.remote_addr
end
```

获取客户端的源IP作为分流策略的依据。

### [table] scheduler
调度器(scheduler)结合当前api_version的upstream和传入的tag字符串，生成新的upstream名称。

| 字段名        | 类型           | 说明  |
| ------------- |:-------------:| :-----|
| id | inter | 自动增长 |
| name | string | 调度器名称 | 
| source | tiny | 0代表从本地加载(load)，1代表从数据库加载(loadstring) |
| code | string | lua code |
| update_time | timestamp | on update CURRENT_TIMESTAMP |

注意：name字段强耦合scheduler.lua代码文件

函数原型

```lua
function (upstream_name, tag)
  if tag == "18601254750" then
      return upstream_name.."9"
  else
      return upstream_name
  end 
end
```

## 杂项
1. 需要增加一个全局开关，是否开启灰度功能
2. 如果流水线功能打开，可以被流水线功能覆盖
3. 需要增加自动同步两个表，selector和scheduler
4. 界面提醒，9有特殊含义，代表灰度发布
