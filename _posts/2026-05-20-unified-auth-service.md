---
layout: post
title: "统一鉴权服务"
date: 2026-05-20
categories: [iot]
mermaid: false
math: false
toc: true
---
# 一、概述
负责执行设备与第三方平台的全面登录认证流程，以强化系统安全性与效率

1. 智能拦截未知设备：系统自动甄别并拦截任何无法识别的设备，确保仅授权设备接入。同时，针对成功登录的车辆，精准记录其对应关系，为后续指令的精准下发奠定坚实基础；
2. 严格审查第三方平台：实施严格的平台验证机制，拦截并拒绝任何未经认证或系统无法识别的第三方平台接入，维护系统生态的纯净与安全；
3. 精准追踪设备登入登出状态：细致记录每一台设备的登出状态，确保系统资源的合理分配与及时释放，同时也为后续的安全审计与性能优化提供宝贵数据支持；

<!-- 这是一张图片，ocr 内容为： -->
![1741767197908-b1f61a84-8054-4642-9ee3-047f9059dab5.png](/assets/img/unified-auth-service/1741767197908-b1f61a84-8054-4642-9ee3-047f9059dab5.png)

# 二、功能模块
## 32960协议
### 与其它服务交互逻辑
#### 32960设备网关
        设备登录网关时，网关会通过http调用设备登录接口进行鉴权

#### 32960平台网关
        平台登录网关时，网关会通过http调用平台登录接口进行鉴权

#### 影子服务
            1. 设备登录接口或平台登录接口会读取设备与车辆绑定关系redis、设备档案redis、车辆档案redis、三方平台档案redis进行鉴权，redis信息通过影子服务维护

### 32960设备登录接口
#### 接口说明
    1. 调用方：设备网关
    2. 调用方式：http

#### 流程图
<!-- 这是一张图片，ocr 内容为： -->
![1741767467738-1ef8c133-6df0-4a98-8818-f63283527a28.png](/assets/img/unified-auth-service/1741767467738-1ef8c133-6df0-4a98-8818-f63283527a28.png)

#### 接口明细
    **请求路径**：/tboxLogin

    **请求方式：**http post

    **请求参数**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| uid | String | 必需 | | 识别码 |
| gatewayIp | String | 必需 | | 网关IP地址 |
| gatewayPort | integer | 必需 | | 网关端口 |

      **返回信息**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| status | boolean | 必须 | false | 返回状态 true成功 false失败 |
| code | string | 必须 | | 状态码 |
| msg | string | 必须 | | 返回消息 |
| data | object | 必须 | | 返回数据 |
|  |-vin | string | 非必须 | | 车架号 |
|  |-logSwitch | boolean | 必须 | false | 日志开关 true开启 false关闭 |
|  |-invalid | boolean | 必须 | false | 无效数据 true是 false否 |

### 32960平台登录接口
#### 接口说明
            1. 调用方：平台网关
            2. 调用方式：http

#### 流程图
<!-- 这是一张图片，ocr 内容为： -->
![1741767569763-f045cd69-63d8-414e-8354-095d7473a683.png](/assets/img/unified-auth-service/1741767569763-f045cd69-63d8-414e-8354-095d7473a683.png)

#### 接口明细
        **请求路径：**/platformLogin

        **请求方式：**http post

        **请求参数**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| username | String | 必需 | | 用户名 |
| md5Password | String | 必需 | | 密码，需要MD5加密 |

      **返回信息**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| status | boolean | 必须 | false | 返回状态 true成功 false失败 |
| code | string | 必须 | | 状态码 |
| msg | string | 必须 | | 返回消息 |
| data | object | 必须 | | 返回数据 |
|  |-logSwitch | boolean | 必须 | false | 日志开关 true开启 false关闭 |

### 32960设备登出接口
#### 接口说明
            1. 调用方：设备网关
            2. 调用方式：http

#### 流程图
<!-- 这是一张图片，ocr 内容为： -->
![1741767590770-5e09514e-b039-4aff-ab7d-a27f00485b25.png](/assets/img/unified-auth-service/1741767590770-5e09514e-b039-4aff-ab7d-a27f00485b25.png)

#### 接口明细
        **请求路径：**/tboxLogout

        **请求方式：**http post

        **请求参数**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| uid | String | 必需 | | 识别码 |
| gatewayIp | String | 必需 | | 网关IP地址 |
| gatewayPort | integer | 必需 | | 网关端口 |

      **返回信息**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | :--- | --- |
| status | boolean | 必须 | false | 返回状态 true成功 false失败 |
| code | string | 必须 | | 状态码 |
| msg | string | 必须 | | 返回消息 |

### redis结构
#### 车辆与Tbox绑定关系
键=tbox:bind:设备号，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">vin</font> | <font style="color:rgb(0, 0, 0);">Lxxxxxxx1</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">车架号</font> |

#### 设备档案-TBOX
键=tbox:info:识别码，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">logSwitch</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">日志开关，true打开;false关闭</font> |

#### 车辆档案
键=car:info:车架号，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">logSwitch</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">日志开关，true打开;false关闭</font> |

#### 三方平台档案
键=platform:info:平台登录名，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">logSwitch</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">日志开关，true打开;false关闭</font> |
| <font style="color:rgb(0, 0, 0);">password</font> | <font style="color:rgb(0, 0, 0);">????</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">密码,需要MD5加密存储，示例：DigestUtils.md5DigestAsHex(req.getPassword().getBytes())</font> |

#### 设备登录信息
      键=tbox:login:识别码，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">ip</font> | <font style="color:rgb(0, 0, 0);">127.0.0.1</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">IP</font> |
| <font style="color:rgb(0, 0, 0);">port</font> | <font style="color:rgb(0, 0, 0);">8080</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">端口</font> |
| <font style="color:rgb(0, 0, 0);">onlineStatus</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">在线状态，true在线;false离线</font> |
| <font style="color:rgb(0, 0, 0);">loginTime</font> | <font style="color:rgb(0, 0, 0);">1725528903872</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">登录时间</font> |
| <font style="color:rgb(0, 0, 0);">logoutTime</font> | <font style="color:rgb(0, 0, 0);">1725528903872</font> | <font style="color:rgb(0, 0, 0);">否</font> | <font style="color:rgb(0, 0, 0);">退出时间</font> |

## GPS协议
### 与其它服务交互逻辑
#### GPS设备网关
        设备登录网关时，网关会通过http调用设备登录接口进行鉴权

#### 影子服务
            1. 设备登录接口会读取设备与车辆绑定关系redis、设备档案redis、车辆档案redis，redis信息通过影子服务维护

### GPS设备登录接口
#### 接口说明
            1. 调用方：设备网关
            2. 调用方式：http

#### 流程图
<!-- 这是一张图片，ocr 内容为： -->
![1741767756919-9a2c6605-b70a-4345-92b3-41268c0801a1.png](/assets/img/unified-auth-service/1741767756919-9a2c6605-b70a-4345-92b3-41268c0801a1.png)

#### 接口明细
        **请求路径**：/gpsLogin

        **请求方式：**http post

        **请求参数**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| uid | String | 必需 | | 识别码 |
| gatewayIp | String | 必需 | | 网关IP地址 |
| gatewayPort | integer | 必需 | | 网关端口 |

      **返回信息**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| status | boolean | 必须 | false | 返回状态 true成功 false失败 |
| code | string | 必须 | | 状态码 |
| msg | string | 必须 | | 返回消息 |
| data | object | 必须 | | 返回数据 |
|  |-vin | string | 非必须 | | 车架号 |
|  |-logSwitch | boolean | 必须 | false | 日志开关 true开启 false关闭 |

### GPS设备登出接口
#### 接口说明
            1. 调用方：设备网关
            2. 调用方式：http

#### 流程图
<!-- 这是一张图片，ocr 内容为： -->
![1741767782970-f31b009e-94c0-4dd5-91e6-ec3ed019a14a.png](/assets/img/unified-auth-service/1741767782970-f31b009e-94c0-4dd5-91e6-ec3ed019a14a.png)

#### 接口明细
        **请求路径：**/tboxLogout

        **请求方式：**http post

        **请求参数**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| uid | String | 必需 | | 识别码 |
| gatewayIp | String | 必需 | | 网关IP地址 |
| gatewayPort | integer | 必需 | | 网关端口 |

      **返回信息**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | :--- | --- |
| status | boolean | 必须 | false | 返回状态 true成功 false失败 |
| code | string | 必须 | | 状态码 |
| msg | string | 必须 | | 返回消息 |

### redis结构
#### 车辆与GPS绑定关系
      键=gps:bind:设备号，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">vin</font> | <font style="color:rgb(0, 0, 0);">Lxxxxxxx1</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">车架号</font> |

#### 设备档案-GPS
      键=gps:info:识别码，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">logSwitch</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">日志开关，true打开;false关闭</font> |

#### 车辆档案
      键=car:info:车架号，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">logSwitch</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">日志开关，true打开;false关闭</font> |

#### 设备登录信息
      键=gps:login:识别码，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">ip</font> | <font style="color:rgb(0, 0, 0);">127.0.0.1</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">IP</font> |
| <font style="color:rgb(0, 0, 0);">port</font> | <font style="color:rgb(0, 0, 0);">5510</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">端口</font> |
| <font style="color:rgb(0, 0, 0);">onlineStatus</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">在线状态，true在线;false离线</font> |
| <font style="color:rgb(0, 0, 0);">loginTime</font> | <font style="color:rgb(0, 0, 0);">1725528903872</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">登录时间</font> |
| <font style="color:rgb(0, 0, 0);">logoutTime</font> | <font style="color:rgb(0, 0, 0);">1725528903872</font> | <font style="color:rgb(0, 0, 0);">否</font> | <font style="color:rgb(0, 0, 0);">退出时间</font> |

## 808协议
### 与其它服务交互逻辑
#### 主动安全设备网关
        设备登录网关时，网关会通过http调用设备登录接口进行鉴权

#### 影子服务
    1. 设备登录接口会读取设备与车辆绑定关系redis、设备档案redis、车辆档案redis，redis信息通过影子服务维护

### 主动安全设备注册接口
#### 接口说明
    1. 调用方：设备网关
    2. 调用方式：http

#### 流程图
<!-- 这是一张图片，ocr 内容为： -->
![1741768340889-e6a82243-94ac-44c7-8efb-91cb9c78fc61.png](/assets/img/unified-auth-service/1741768340889-e6a82243-94ac-44c7-8efb-91cb9c78fc61.png)

#### 接口明细
        **请求路径**：/adasRegister

        **请求方式：**http post

        **请求参数**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| mobile | String | 必需 | | 手机号 |

      **返回信息**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | :--- | --- |
| status | boolean | 必须 | false | 返回状态 true成功 false失败 |
| code | string | 必须 | | 状态码 |
| msg | string | 必须 | | 返回消息 |
| data | object | 必须 | | 返回数据 |
|  |-authCode | string | 非必须 | | 鉴权码 |

### 主动安全设备登录接口
#### 接口说明
    1. 调用方：设备网关
    2. 调用方式：http

#### 流程图
<!-- 这是一张图片，ocr 内容为： -->
![1741768369559-555c476a-c55d-411f-8fb5-6e7663bd770b.png](/assets/img/unified-auth-service/1741768369559-555c476a-c55d-411f-8fb5-6e7663bd770b.png)

#### 接口明细
        **请求路径**：/adasLogin

        **请求方式：**http post

        **请求参数**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| mobile | String | 必需 | | 手机号 |
| authCode | String | 必需 | | _鉴权码_ |
| gatewayIp | String | 必需 | | _网关IP地址_ |
| gatewayPort | integer | 必需 | | _网关端口_ |

      **返回信息**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| status | boolean | 必须 | false | 返回状态 true成功 false失败 |
| code | string | 必须 | | 状态码 |
| msg | string | 必须 | | 返回消息 |
| data | object | 必须 | | 返回数据 |
|  |-vin | string | 非必须 | | 车架号 |
|  |-logSwitch | boolean | 必须 | false | 日志开关 true开启 false关闭 |

### 主动安全设备登出接口
#### 接口说明
    1. 调用方：设备网关
    2. 调用方式：http

#### 流程图
<!-- 这是一张图片，ocr 内容为： -->
![1741768399194-fab2f000-5e1c-4dea-8ea9-1ec3ef1e3728.png](/assets/img/unified-auth-service/1741768399194-fab2f000-5e1c-4dea-8ea9-1ec3ef1e3728.png)

#### 接口明细
        **请求路径：**/adasLogout

        **请求方式：**http post

        **请求参数**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | --- | --- |
| mobile | String | 必需 | | 手机号 |
| gatewayIp | String | 必需 | | 网关IP地址 |
| gatewayPort | integer | 必需 | | 网关端口 |

      **返回信息**

| **名称** | **类型** | **是否必须** | **默认值** | **备注** |
| --- | --- | --- | :--- | --- |
| status | boolean | 必须 | false | 返回状态 true成功 false失败 |
| code | string | 必须 | | 状态码 |
| msg | string | 必须 | | 返回消息 |

### redis结构
#### 车辆与ADAS绑定关系
      键=adas:bind:识别码，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">vinCode</font> | <font style="color:rgb(0, 0, 0);">Lxxxxxxx1</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">车架号</font> |

#### 设备档案-ADAS
      键=adas:info:识别码，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">logSwitch</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">日志开关，true打开;false关闭</font> |
| <font style="color:rgb(0, 0, 0);">authCode</font> | <font style="color:rgb(0, 0, 0);">111</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">鉴权码</font> |

#### 车辆档案
      键=car:info:车架号，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">logSwitch</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">日志开关，true打开;false关闭</font> |

#### 设备登录信息
     键=adas:login:识别码，类型=hash

| **<font style="color:rgb(0, 0, 0);">key</font>** | **<font style="color:rgb(0, 0, 0);">value</font>** | **<font style="color:rgb(0, 0, 0);">是否必填</font>** | **<font style="color:rgb(0, 0, 0);">备注</font>** |
| --- | --- | --- | --- |
| <font style="color:rgb(0, 0, 0);">ip</font> | <font style="color:rgb(0, 0, 0);">127.0.0.1</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">IP</font> |
| <font style="color:rgb(0, 0, 0);">port</font> | <font style="color:rgb(0, 0, 0);">5514</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">端口</font> |
| <font style="color:rgb(0, 0, 0);">onlineStatus</font> | <font style="color:rgb(0, 0, 0);">TRUE</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">在线状态，true在线;false离线</font> |
| <font style="color:rgb(0, 0, 0);">loginTime</font> | <font style="color:rgb(0, 0, 0);">1725528903872</font> | <font style="color:rgb(0, 0, 0);">是</font> | <font style="color:rgb(0, 0, 0);">登录时间</font> |
| <font style="color:rgb(0, 0, 0);">logoutTime</font> | <font style="color:rgb(0, 0, 0);">1725528903872</font> | <font style="color:rgb(0, 0, 0);">否</font> | <font style="color:rgb(0, 0, 0);">退出时间</font> |
