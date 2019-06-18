# Sunlands打点系统技术方案

## 1. 需求说明

### 1.1. 项目背景

1. 用户行为数据分散在各个系统，难以获取到同一用户的各种行为数据，难以基于多种用户行为进行筛选；
2. 现有的做法是，由数据中心汇总各系统的用户数据，离线分析是否同一用户。该方法在实时性上有一定局限性。

### 1.2. 项目目标

1. 实现全系统用户关键行为数据的统一；
2. 逐步完善统一uuid的关联逻辑，提高用户统一uuid的准确率；
3. 规范打点的事件和属性定义

## 2. 架构设计
### 2.1 整体架构设计
![image-20190424164958098](image/archetype.png)

### 2.2 打点序列图

![image-20190424164958098](image/sequent.png)

## 3. 详细设计

### 3.1 Api设计

1. POST + JSON的方式：

   http://datacenter.sunlands.com/slog/event

2. JSON格式定义：

```json
{
"time":1434556935000, // 时间戳，进度毫秒; 客户端可以不传，以服务端时间为准
"use_server_time": true, // 是否使用服务端时间戳; 默认true,会忽略time字段; 当为false时，使用time字段, 典型场景为导入历史数据
"stype":100, // 消息类型, 取值范围100~199，详见"3.2 消息类型"章节
"eid":"ViewViedo", // 事件名称; profile_**类型的消息不需要该字段
"sid":"12345", // 站点id
"ver":"js1.1.0", //sdk版本号
"r":13212, // 5位随机数，防止缓存
"project": "zhibo", // 产品/业务线名称
"ps": { // 事件属性字段
"url": "a/b/c/d",
"ip":"180.12.43.42",
"teacher": "abc",
"start_time": 1434556935000,
},
"uidentity": { // 用户不变量属性
"uuid":"123123123", // 用户id, 当用户登录后，需要设置该字段; 当未传该字段时，slog根据其他不变量从用户中心获取uuid，补上透传给MQ
"mobile": "1234566778"
}
}
```

3. 事件类型

| stype | 消息类型       | 消息说明                                                     | 目标数据库 |
| ----- | -------------- | ------------------------------------------------------------ | ---------- |
| 100   | track          | 普通的打点事件，如点击按钮、进入页面等                       | events     |
| 101   | profile_update | 设置用户的profile, 如果用户或用户的profile字段已经存在，直接覆盖；不存在，则自动创建 | users      |
| 102   | profile_insert | 用户或用户的profile字段存在，则不更新；不存在则创建          | users      |
| 103   | profile_unset  | 将用户的某个profile字段值空                                  | users      |
| 104   | profile_append | 向用户的某个数组类型的 Profile 添加一个或者多个值            | users      |
|       | profile_***    | 其他用户画像相关的接口，可在后续迭代过程中根据需要添加       | users      |

### 3.2 元数据管理系统设计

1. 元数据管理采用mysql + redis方案，redis作为缓存使用 

2. 元数据管理分为4个表：

(1) event定义表， 表名：events

| 字段        | 类型         | 主键        | 含义                                      |
| ----------- | ------------ | ----------- | ----------------------------------------- |
| id          | varchar(64)  | primary Key | Event id                                  |
| name        | varchar(128) |             | event展示名称                             |
| poject      | varchar(64)  | index       | event所属产品线，通用属性时，该字段为null |
| is_reserve  | tinyint(1)   |             | 是否预置事件，0表示非预置，1表示预置      |
| need_uuid   | tinyint(1)   |             | 是否需要补齐uuid, 0:不需补齐，1:需要补齐  |
| proposer    | varchar(64)  |             | 申请人                                    |
| create_time | int(10)      | index       | 创建时间                                  |

(2) event属性定义表， 表名： event_properties

| 字段        | 类型         | 主键        | 含义                                                        |
| ----------- | ------------ | ----------- | ----------------------------------------------------------- |
| key         | varchar(64)  | primary key | 属性名称                                                    |
| name        | varchar(128) |             | 属性说明                                                    |
| value_type  | int(10)      |             | 属性值的类型, 0:string, 1:datetime, 2:bool, 3:int, 4:double |
| is_reserve  | tinyint(1)   |             | 是否预置属性，0表示非预置，1表示预置                        |
| proposer    | varchar(64)  |             | 申请人                                                      |
| create_time | int(10)      | index       | 创建时间                                                    |

(3) 用户不变量标识属性定义表：user_identity_properties:

| 字段        | 类型         | 主键        | 含义                                                        |
| ----------- | ------------ | ----------- | ----------------------------------------------------------- |
| key         | varchar(64)  | primary key | 属性名称                                                    |
| name        | varchar(128) |             | 属性说明                                                    |
| value_type  | int(10)      |             | 属性值的类型, 0:string, 1:datetime, 2:bool, 3:int, 4:double |
| is_reserve  | tinyint(1)   |             | 是否预置属性，0表示非预置，1表示预置                        |
| proposer    | varchar(64)  |             | 申请人                                                      |
| create_time | int(10)      | index       | 创建时间                                                    |

(4) profile属性定义表， 表名：profile_properties

| 字段        | 类型         | 主键        | 含义                                                        |
| ----------- | ------------ | ----------- | ----------------------------------------------------------- |
| key         | varchar(64)  | primary key | 属性名称                                                    |
| name        | varchar(128) |             | 属性说明                                                    |
| value_type  | int(10)      |             | 属性值的类型, 0:string, 1:datetime, 2:bool, 3:int, 4:double |
| is_reserve  | tinyint(1)   |             | 是否预置属性，0表示非预置，1表示预置                        |
| proposer    | varchar(64)  |             | 申请人                                                      |
| create_time | int(10)      | index       | 创建时间                                                    |

### 3.3 预置event与属性字段

#### 3.3.1 预置event

| event         | 说明         |
| ------------- | ------------ |
| AppClick      | App元素点击  |
| AppViewScreen | App 浏览页面 |
| AppStart      | 启动 App     |
| AppEnd        | 退出App      |
| WebClick      | Web 元素点击 |
| pageview      | Web 浏览页面 |
| WebStay       | Web 视区停留 |

#### 3.3.2 event预置字段

**(1) web端**

| 字段名称       | 类型   | 说明                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| ct             | int    | 客户端类型, web端为100                                       |
| lib            | string | SDK类型，如python,js等                                       |
| lib_version    | string | SDK版本                                                      |
| referrer       | string | 前向地址                                                     |
| referrer_host  | string | 前向域名                                                     |
| first_referrer | string | 站外前向地址，如http://www.baidu.com, 若是浏览器直接打开，则为空 |

**(2) 移动端App**

| 字段名称               | 类型   | 说明                               |
| ---------------------- | ------ | ---------------------------------- |
| app_version            | string | 应用的版本                         |
| ct                     | int    | 客户端类型，android：200， ios:300 |
| lib                    | string | SDK类型，如python,js等             |
| lib_version            | string | SDK版本                            |
| manufacturer           | string | 设备制造商，如Huawei               |
| model                  | string | 设备型号, 如Mate20                 |
| screen_height          | int    | 屏幕高度                           |
| screen_width           | int    | 屏幕宽度                           |
| os                     | string | 操作系统                           |
| os_version             | string | 操作系统版本号                     |
| wifi                   | bool   | 是否使用wifi                       |
| resume_from_background | bool   | 是否从后台唤醒                     |
| carrier                | string | 运营商名称，如ChinaNet             |
| network_type           | string | 网络类型，如4G                     |

**(3)小程序**

| 字段名称     | 类型   | 说明                       |
| ------------ | ------ | -------------------------- |
| ct           | int    | 客户端类型, 微信小程序:400 |
| lib          | string | SDK类型，如python,js等     |
| lib_version  | string | SDK版本                    |
| model        | string | 设备型号, 如Mate20         |
| os           | string | 操作系统                   |
| os_version   | string | 操作系统版本号             |
| wifi         | bool   | 是否使用wifi               |
| carrier      | string | 运营商名称，如ChinaNet     |
| network_type | string | 网络类型，如4G             |

### 3.3.3 用户不变量标识预置属性

| 字段名称      | 类型   | 说明                               | 采集方式                 |
| ------------- | ------ | ---------------------------------- | ------------------------ |
| mobile        | string | 手机号                             | 用户输入&三方授权        |
| imei          | string | android标识                        | app读取                  |
| androidUUID   | string | android标识                        | app读取                  |
| macAddress    | string | mac地址                            | app读取                  |
| idfa          | string | ios标识                            | app读取                  |
| iosUUID       | string | ios标识                            | app读取                  |
| wx-id         | string | 微信标识                           | 用户录入&天网精灵获取    |
| openid        | string | 小程序公众号标识                   | 各种小程序/H5获取        |
| appid         | string | 小程序公众号标识                   | 各种小程序/H5获取        |
| unionid       | string | 公众平台用户标识                   | 用户微信授权认证获取     |
| userid        | string | 后端账户id                         | 用户注册时生成           |
| deviceUUID    | string | 用户中心生成的关联设备信息的随机数 | 数据库读取，接口请求携带 |
| opptunityId   | string | 机会id                             | 天网处理名片后系统生成   |
| uvId          | string | 项目框架页长效cookie               | 网页请求服务端可获取     |
| chatId        | string | 大熊会话id                         | 大熊生成并获取           |
| browserId     | string | 使用多种算法生成的浏览器唯一标识   | js页面中计算获得         |
| qq            | string | 天网名片信息                       | 天网名片录入获取         |
| idCard        | string | 身份证                             | 用户录入                 |
| email263      | string | 公司工作人员标识                   | ehr接口读取              |
| orderDetailId | string | 子订单号                           | 订单系统生成，数据库     |
| clickId       | string | 广告系统id                         | 广告系统从三方获取       |

#### 3.3.4 profile预置字段

| 字段名称            | 类型     | 说明             | 采集方式                                                     |
| ------------------- | -------- | ---------------- | ------------------------------------------------------------ |
| city                | string   | 用户所在城市     | 手动填写                                                     |
| province            | string   | 用户所在省份     | 手动填写                                                     |
| country             | string   | 用户所在国家     | 手动填写                                                     |
| name                | string   | 姓名             | 手动填写                                                     |
| sex                 | int      | 性别             | 手动填写                                                     |
| first_visit_time    | Datetime | 首次访问时间     | 后台自动计算                                                 |
| first_referrer      | string   | 首次站外前向地址 | 后台自动计算，slog中收到track事件时，比较first_referrer和referrer, 若两者不为空且相同，则同时发送profile_insert事件；<br />或者由MQ后面的流式计算算子来处理，这样可以开放给各业务线自己定义一些算子 |
| first_referrer_host | string   | 首次站外前向域名 | 后台自动计算，slog中收到track事件时，比较first_referrer和referrer, 若两者不为空且相同，则同时发送profile_insert事件 |
| signup_time         | string   | 注册时间         |                                                              |

## 4. 案例分析

为更清晰的分析整个打点系统的使用，下面以"课程付费分析"的案例来进行分析。

![image-20190428140506082](image/case1.png)

1. **此案例中，涉及到两种event：pageview(预置事件)和buyCourse(业务方自定义事件)**

2. **对pageview，需要上报的属性字段包括：**

| 字段名称               | 类型   | 说明                                                         | 采集方式                                                     |
| ---------------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| app_version            | string | 应用的版本                                                   | 移动端由sdk自动采集                                          |
| ct                     | int    | 客户端类型                                                   | 1~99: 服务端系列<br />    1: 服务端sdk<br />    2: 日志批量导入<br />100~199: web js系列<br />200~299:android/android tv/android pad等<br />300~399:ios系列<br />400~499: 小程序系列 |
| country                | string | 国家                                                         | 各前端sdk可自动采集; 若前端无该字段，且$ct>=100，则由slog根据ip解析 |
| city                   | string | 城市                                                         | 各前端sdk可自动采集; 若前端无该字段，且$ct>=100，则由slog根据ip解析 |
| province               | string | 省份                                                         | 各前端sdk可自动采集; 若前端无该字段，且$ct>=100，则由slog根据ip解析 |
| ip                     | string | 客户端ip地址                                                 | 各前端sdk可自动采集; 若前端无该字段，且$ct>=100，则由slog从http请求包中获取 |
| lib                    | string | SDK类型，如python,js等                                       | 各SDK自动采集                                                |
| lib_version            | string | SDK版本                                                      | 各SDK自动采集                                                |
| manufacturer           | string | 设备制造商，如Huawei                                         | 移动端SDK自动采集                                            |
| model                  | string | 设备型号, 如Mate20                                           | 移动端和小程序SDK自动采集                                    |
| screen_height          | int    | 屏幕高度                                                     | 移动端SDK自动采集                                            |
| screen_width           | int    | 屏幕宽度                                                     | 移动端SDK自动采集                                            |
| os                     | string | 操作系统                                                     | 各前端SDK可自动采集; 若前端无该字段，且$ct>=100，则由slog从http请求包中的useragent中获取 |
| os_version             | string | 操作系统版本号                                               | 各前端SDK可自动采集; 若前端无该字段，且$ct>=100，则由slog从http请求包中的useragent中获取 |
| browser                | string | 浏览器                                                       | web端SDK可自动采集; 若前端无该字段，且$ct在100~199，则由slog从http请求包中的useragent中获取 |
| browser_version        | string | 浏览器版本号                                                 | web端SDK可自动采集; 若前端无该字段，且$ct在100~199，则由slog从http请求包中的useragent中获取 |
| wifi                   | bool   | 是否使用wifi                                                 | 移动端SDK自动采集                                            |
| resume_from_background | bool   | 是否从后台唤醒                                               | 移动端SDK自动采集                                            |
| carrier                | string | 运营商名称，如ChinaNet                                       | 移动端SDK自动采集                                            |
| network_type           | string | 网络类型，如4G                                               | 移动端SDK自动采集                                            |
| referrer               | string | 前向地址                                                     | web端SDK自动采集                                             |
| referrer_host          | string | 前向域名                                                     | web端SDK自动采集                                             |
| first_referrer         | string | 站外前向地址，如http://www.baidu.com, 若是浏览器直接打开，则为空 | web端SDK自动采集                                             |
| url                    | string | 页面地址                                                     |                                                              |
| title                  | string | 页面标题                                                     |                                                              |
| url_path               | string | 页面路径                                                     |                                                              |
| course_name            | string | 课程名称，业务自定义字段                                     |                                                              |
| course_type            | string | 课程类型，业务自定义字段                                     |                                                              |
| course_difficulty      | string | 课程难度，业务自定义字段                                     |                                                              |

3. **对于buyCourse，可以由客户端上报，也可以由服务端上报。** **原则上，能有服务端上报的，最好由服务端上报**若客户端上报，则需要上报的字段包括：

| 字段名称               | 类型   | 说明                                                         | 采集方式                                                     |
| ---------------------- | ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| app_version            | string | 应用的版本                                                   | 移动端由sdk自动采集                                          |
| ct                     | int    | 客户端类型                                                   | 1~99: 服务端系列<br />    1: 服务端sdk<br />    2: 日志批量导入<br />100~199: web js系列<br />200~299:android/android tv/android pad等<br />300~399:ios系列<br />400~499: 小程序系列 |
| country                | string | 国家                                                         | 各前端sdk可自动采集; 若前端无该字段，且$ct>=100，则由slog根据ip解析 |
| city                   | string | 城市                                                         | 各前端sdk可自动采集; 若前端无该字段，且$ct>=100，则由slog根据ip解析 |
| province               | string | 省份                                                         | 各前端sdk可自动采集; 若前端无该字段，且$ct>=100，则由slog根据ip解析 |
| ip                     | string | 客户端ip地址                                                 | 各前端sdk可自动采集; 若前端无该字段，且$ct>=100，则由slog从http请求包中获取 |
| lib                    | string | SDK类型，如python,js等                                       | 各SDK自动采集                                                |
| lib_version            | string | SDK版本                                                      | 各SDK自动采集                                                |
| manufacturer           | string | 设备制造商，如Huawei                                         | 移动端SDK自动采集                                            |
| model                  | string | 设备型号, 如Mate20                                           | 移动端和小程序SDK自动采集                                    |
| screen_height          | int    | 屏幕高度                                                     | 移动端SDK自动采集                                            |
| screen_width           | int    | 屏幕宽度                                                     | 移动端SDK自动采集                                            |
| os                     | string | 操作系统                                                     | 各前端SDK可自动采集; 若前端无该字段，且$ct>=100，则由slog从http请求包中的useragent中获取 |
| os_version             | string | 操作系统版本号                                               | 各前端SDK可自动采集; 若前端无该字段，且$ct>=100，则由slog从http请求包中的useragent中获取 |
| browser                | string | 浏览器                                                       | web端SDK可自动采集; 若前端无该字段，且$ct在100~199，则由slog从http请求包中的useragent中获取 |
| browser_version        | string | 浏览器版本号                                                 | web端SDK可自动采集; 若前端无该字段，且$ct在100~199，则由slog从http请求包中的useragent中获取 |
| wifi                   | bool   | 是否使用wifi                                                 | 移动端SDK自动采集                                            |
| resume_from_background | bool   | 是否从后台唤醒                                               | 移动端SDK自动采集                                            |
| carrier                | string | 运营商名称，如ChinaNet                                       | 移动端SDK自动采集                                            |
| network_type           | string | 网络类型，如4G                                               | 移动端SDK自动采集                                            |
| referrer               | string | 前向地址                                                     | web端SDK自动采集                                             |
| referrer_host          | string | 前向域名                                                     | web端SDK自动采集                                             |
| first_referrer         | string | 站外前向地址，如http://www.baidu.com, 若是浏览器直接打开，则为空 | web端SDK自动采集                                             |
| url                    | string | 页面地址                                                     |                                                              |
| title                  | string | 页面标题                                                     |                                                              |
| url_path               | string | 页面路径                                                     |                                                              |
| course_name            | string | 课程名称，业务自定义字段                                     |                                                              |
| course_type            | string | 课程类型，业务自定义字段                                     |                                                              |
| course_difficulty      | string | 课程难度，业务自定义字段                                     |                                                              |
| money                  | int    | 支付金额，业务自定义字段                                     |                                                              |

若由服务端上报，则上报的属性字段如下：

| 字段名称          | 类型   | 说明                     |
| ----------------- | ------ | ------------------------ |
| course_name       | string | 课程名称，业务自定义字段 |
| course_type       | string | 课程类型，业务自定义字段 |
| course_difficulty | string | 课程难度，业务自定义字段 |
| money             | int    | 支付金额，业务自定义字段 |

**事件关联：**由于后台上报时，前端相关的属性字段信息丢失，可在数据分析环节做事件关联(可参考神策的方案，做session分析，在此案例中，可将同一用户的buyCourse和$pageview事件关联做成一个session，然后通过session分析)

4. **数据分析查询API示例**

**(1) 各课程销量的API实例：**

- 使用事件分析API: `[POST /events/report]`

- Request (application/json)

```json
{
    "measures":[
        {
            // 事件名称，特别的，可以使用 $Anything 表示任意事件
            "event_name":"buyCourse",
            // 聚合操作符, uniq_count指按触发用户数聚合
            "aggregator":"uniq_count"
        }
    ],
    // 起始日期
    "from_date":"2019-04-21",
    // 结束日期
    "to_date":"2019-04-27",
    // 是否计算合计值
    "detail_and_rollup":true,
    // 分组属性
    "by_fields":[
        "event.buyCourse.course_name"
    ],
    // 最大分组个数
    "limit":100,
    // 使用缓存，若缓存中找不到相应数据，则从数据库读出
    "use_cache":true
}
```

- Respose 200 (application/json)

```json
{
    "by_fields":[
        "event.buyCourse.course_name"
    ],
    "rows":[
        {
        "values":[
            2645
        ],
        "by_values":[
            "超级外教"
        ]
        },
        {
        "values":[
            2754
        ],
        "by_values":[
            "新概念”
        ]
        }，
      	{
        "values":[
            2789
        ],
        "by_values":[
            "趣味阅读"
        ]
        }
    ],
    "num_rows": 3,
    "total_rows": 3,
    "report_update_time": "2019-04-28 13:51:08.356",
    "data_update_time": "2019-04-27 16:03:32.000",
    "data_sufficient_update_time": "2019-04-27 16:03:32.000",
    "truncated": false
}

```

**(2) 各难度等级用户复购情况分布数据分析API示例**

- 使用分布分析API: `[POST /addictions/report]` 

- Request (application/json)

```json
{
    // 事件名称
    "event_name": "buyCourse",
    // 起始时间
    "from_date": "2019-04-21",
    // 结束时间
    "to_date": "2019-04-27",
    // 分组属性
    "by_field": "event.buyCourse.course_name",
    // 抽样因子，64为全量，32为2分之1抽样
    "sampling_factor":64,
    // 事件单位，可以是 day/week/month
    "unit": "day", 
    // 测量类型,可以是times/period, period是当按小时数或者天数进行分布分析时使用
    "measure_type":"times",
    //测量类型如果是times，即可以自定义分桶，可省略
    "result_bucket_param": [
      1,
      2,
      3,
      4
    ],
    // 使用缓存，若缓存中找不到相应数据，则从数据库读出
    "use_cache":true
}

```

- Respose 200 (application/json)

```json
{
	"by_field": "event.buyCourse.course_name",
	"rows": [{
		"by_value": "初级",
		"total_people": 14419,
		"cells": [{
			"people": 10765,
			"percent": 74.66,
			"bucket_start": 1.0,
			"bucket_end": 2.0
		}, {
			"people": 2997,
			"percent": 20.79,
			"bucket_start": 2.0,
			"bucket_end": 3.0
		}, {
			"people": 561,
			"percent": 3.89,
			"bucket_start": 3.0,
			"bucket_end": 4.0
		}, {
			"people": 96,
			"percent": 0.67,
			"bucket_start": 4.0
		}]
	}, {
		"by_value": "困难",
		"total_people": 14341,
		"cells": [{
			"people": 10758,
			"percent": 75.02,
			"bucket_start": 1.0,
			"bucket_end": 2.0
		}, {
			"people": 2943,
			"percent": 20.52,
			"bucket_start": 2.0,
			"bucket_end": 3.0
		}, {
			"people": 572,
			"percent": 3.99,
			"bucket_start": 3.0,
			"bucket_end": 4.0
		}, {
			"people": 68,
			"percent": 0.47,
			"bucket_start": 4.0
		}]
	}, {
		"by_value": "中等",
		"total_people": 14273,
		"cells": [{
			"people": 10727,
			"percent": 75.16,
			"bucket_start": 1.0,
			"bucket_end": 2.0
		}, {
			"people": 2979,
			"percent": 20.87,
			"bucket_start": 2.0,
			"bucket_end": 3.0
		}, {
			"people": 474,
			"percent": 3.32,
			"bucket_start": 3.0,
			"bucket_end": 4.0
		}, {
			"people": 93,
			"percent": 0.65,
			"bucket_start": 4.0
		}]
	}, {
		"total_people": 22,
		"cells": [{
			"people": 20,
			"percent": 90.91,
			"bucket_start": 1.0,
			"bucket_end": 2.0
		}, {
			"people": 2,
			"percent": 9.09,
			"bucket_start": 2.0,
			"bucket_end": 3.0
		}, {
			"people": 0,
			"percent": 0.0,
			"bucket_start": 3.0,
			"bucket_end": 4.0
		}, {
			"people": 0,
			"percent": 0.0,
			"bucket_start": 4.0
		}]
	}],
	"report_update_time": "2019-04-29 10:02:15.749",
	"data_update_time": "2019-04-29 00:00:03.000",
	"data_sufficient_update_time": "2019-04-29 02:00:00.000",
	"truncated": false,
	"sampling_factor": 64
}
```

## 5. 后续可能的扩展功能

### 5.1 虚拟事件

某些场景下面，为方便做数据分析，可能需要将多种事件归类为同一种事件。针对此场景可以在前端数据分析模块，增加通用的虚拟事件分析模块，用于管理虚拟事件。场景举例：

用户首次启动APP、用户首次浏览页面等事件，都可以认为是新用户访问，如果需要做新用户访问的事件分析，就需要做比较复杂的查询。此时，可以新建"newUser"虚拟事件，规则如下：

| 元事件   | 触发条件           |
| -------- | ------------------ |
| AppStart | is_first_time=true |
| pageview | is_first_time=true |

只要两个事件中，任意一个满足条件的事件被触发，则同时生产"newUser"虚拟事件。

这样后续就可很方便的做新用户来访相关的数据分析。

### 5.2 Session管理

1. 在上一节中已经提到了session的概念，其实session还可以扩展到更广的范围，如：需要分析用户一次来访中浏览的页面数、用户上课过程中的互动情况等，这些场景都是需要将一定时间内的多个事件进行关联分析，针对这种场景，可以在数据存储部分增加自定义的session表，在前置数据分析模块增加session分析相关的功能，当有相应的事件触发时，自动更新对应的session信息，以方便后续的数据分析。
