# SSE（Strongene Smart Education）微信公众号整体设计

---

## 1. 微信公众号的功能

* 1. 学生家长通过我们的微信服务号来加入到我们的系统中来；

* 2. 家长微信号与学生账号绑定：
** 2.1  绑定家长微信号的时候需要通过短信获取验证码；
** 2.2  每个学生账号可由哪些手机号来获取验证码由学生设定（也可以学生告诉管理员，由管理员来录入）
** 2.3  学生设置的可以接收绑定验证码的手机号集合是一个白名单，当家长在微信中试图绑定学生账号时（可通过扫学生设备上的二维码触发），要求输入接收验证码的手机号，只有家长输入的手机号在白名单中，系统才会发送验证码，否则不允许绑定。这样就可以限制只有学生设定的手机号能够绑定到学生账号

* 3. 微信企业号所提供的功能；
** 3.1 推送信息：
*** 3.1.1  定时推送每日作业摘要，包含昨日作业批改结果和今日新布置的作业摘要信息（昨日做对做错多少道，今日有多少个作业）；
*** 3.1.2  睡前未交作业提醒，每日晚上10点左右，如果学生有未交作业，则提醒（此功能待定）；
*** 3.1.3  按需推送班级通知和针对单个学生的通知，班级通知全班学生的所有已绑定微信号的家长都能收到，针对单个学生的通知只有这个学生的已绑定家长能够收到；这些通知可由班级的任意一个老师在老师端平板上发送。

** 3.2 拉取消息：
*** 3.2.1  当日作业摘要（见3.1.1）；
*** 3.2.2  睡前未交作业（见3.1.2）；
*** 3.2.3  可能还可以拉取一个一段时间（比如一个星期）的摘要信息，并绘制一个图表来显示

## 2. 网络整体拓扑
整体拓扑涉及到三台服务器：公众号服务器、微信服务器、sseserver服务器；
拓扑结构是：
			sseserver <-------发送请求获取数据------ 公众号服务器 <------用户给公众号发送消息 / 调用微信API------> 微信服务器

公众号服务器既有服务端的行为：接受微信服务器发来的请求；
也有客户端的行为：向sseserver发送请求来获取数据、通过调用微信API向微信服务器发送请求；


## 3. 数据表
根据微信公众号的功能，需要在sseweixin上添加的mysql数据表有：

### 3.1 家长表

#### 字段如下表：  
  
| 字段名        | 类型       | 说明                                                       						 | 必选 |  
|:----------   |:---------- |:---------------------------------------------------------------------------------|:---- |
| id           | INT        | 当前记录的ID，自动自增字段，插入当前记录时由数据库自动生成 						 |  是  |  
| xueshengID   | INT        | 所绑定学生的id                                             						 |  是  |
| banji        | TEXT       | 所绑定学生的班级名称                                         						 |  是  |
| username     | TEXT       | 所绑定学生的登录用户名                                        						 |  是  |
| xueshengName | TEXT       | 学生的姓名                                         									 |  是  |  
| phone        | TEXT       | 该学生允许接收绑定验证码短信的一个手机号                   				         |  是  |
| parentID     | INT        | 家长微信的的OpenID																 |  是  |
| ip           | TEXT       | 学生所在学校的公网ip                                     					     |  是  |  
| port         | INT        | 学校服务的端口号。                                                               |  是  |  
| weixin       | TEXT       | 通过这个手机号绑定到这个学生账号的家长微信号               					     |  是  |  
| zhaiyao      | INT        | 是否接收每日摘要                                                                 |  是  |  
| weijiao      | INT        | 在设定的时间点发送睡前未交作业提醒，如果值为null或者负数，则关闭睡前未交作业提醒 |  是  |  
| shanchu      | INT        | 1: 表示这条记录被删除； 0： 表示没有被删除（默认值） 							 |  是  |  


### 3.2 白名单表

#### 字段如下表：

| 字段名     | 类型       | 说明                                                       						 | 必选 |  
|:---------- |:---------- |:---------------------------------------------------------------------------------|:---- |
| id         | INT        | 当前记录的ID，自动自增字段，插入当前记录时由数据库自动生成 						 |  是  |  
| student    | INT        | 学生的id                                             						     |  是  |  
| phone      | TEXT       | 该学生允许接收绑定验证码短信的一个手机号                   				         |  是  |  

#### 注意：
在插入数据到家长表之前，需要先检查白名单表中此学生是否允许该手机号对其进行绑定，如果白名单表中不存在这种对应关系就不允许插入家长表中；


### 3.3 家长通知消息表

#### 字段如下表：  
  
| 字段名     | 类型       | 说明                                                       						 | 必选 |  
|:---------- |:---------- |:---------------------------------------------------------------------------------|:---- |
| id         | INT        | 当前记录的ID，自动自增字段，插入当前记录时由数据库自动生成 						 |  是  |  
| laoshi     | INT        | 发布消息的老师的ID                                      						 |  是  |  
| banji      | INT        | 消息发送给的班级的ID                                     				         |  是  |  
| xuesheng   | INT        | 消息发送给的学生的ID，如果是发送给班级的全部学生的，则此字段为null 			     |  是  |  
| time       | INT        | 消息发布的时间                                                                   |  是  |  
| message    | TEXT       | 消息内容                                                                         |  是  | 
| shanchu    | INT        | 1: 表示这条记录被删除； 0： 表示没有被删除（默认值） 							 |  是  |


### 3.4 Token表

#### 字段如下表：  
  
| 字段名     | 类型       | 说明                                                       						 | 必选 |  
|:---------- |:---------- |:---------------------------------------------------------------------------------|:---- |
| id         | INT        | 当前记录的ID，自动自增字段，插入当前记录时由数据库自动生成 						 |  是  |  
| tokeninfo  | TEXT       | 公众号访问微信接口的accessToken信息，包括accessToken和expireTime                 |  是  |  


### 3.5 客服表

#### 字段如下表：  
  
| 字段名     | 类型       | 说明                                                                                | 必选 |  
|:---------- |:---------- |:---------------------------------------------------------------------------------|:---- |
| id         | INT        | 当前记录的ID，自动自增字段，插入当前记录时由数据库自动生成                              |  是  |  
| openID     | TEXT       | 该客服人员的openID                                                                |  是  | 
| zhiban     | INT        | 记录当前客服人员是否在值班：0（下班）；1（值班）默认为0                                  |  是  |


### 3.5 学校表

#### 字段如下表：

| 字段名     | 类型       | 说明                                                                                | 必选 |  
|:---------- |:---------- |:---------------------------------------------------------------------------------|:---- |
| id         | INT        | 当前记录的ID，自动自增字段，插入当前记录时由数据库自动生成                              |  是  |  
| schoolID   | INT        | 学校的ID                                                                           |  是  | 
| proxy      | TEXT       | 学校的url和端口号                                                                   |  是  |
| token      | TEXT       | 微信登陆学校后的token                                                                |  否  |


## 4. 账号绑定

基本流程：

* 学生先设置白名单，通过调用“设置白名单接口”公众号服务器向sseserver发送POST请求，在sseserver数据库的“白名单表”上插入白名单记录（该学生学号以及允许绑定的手机号）；
* 家长调用公众号中的扫码按钮扫描学生二维码后，微信服务器会调用扫码推事件的事件推送，将二维码信息推送给公众号；
* 公众号接到该事件后记录下事件信息（包括家长的OpenID等），然后公众号调用API中的“被动回复文本消息”的接口，要求家长输入接收验证码的手机号码；
* 当家长将手机号发送给公众号服务器时，通过调用“检查白名单接口”向sseserver发送GET请求，当验证手机号在白名单中存在时发送验证码给家长（发送验证码待定），反之通知家长不允许绑定；
* 当家长通过验证时，调用“添加家长信息接口”向sseserver发送POST请求，将家长的信息（手机号等）插入数据库的“家长表”中；

* 注意：
1. 先实现“简易”扫码绑定版本，即家长扫描二维码直接进行绑定，不进行“白名单”和接收验证码的相关步骤；
2. 客户端生成的二维码必须包含以下信息：学生所在学校的公网ip、学校的端口号（port）、学生的学号(xueshengID)、学生所在班级(banji)、学生的姓名(name)、学生在用户表的id(id)、；且必须以JSON格式生成二维码，比如：
   {"ip" : "101.200.123.36", "id" : "123456", "port" : "8000", "name" : "张三", "banji" : "初一（1）班", "xueshengID" : "abc12345"}，生成的二维码是一个字符串，即'{"ip" : "101.200.123.36", "id" : "123456", "port" : "8000", "name" : "张三", "banji" : "初一（1）班", "xueshengID" : "abc12345"}'；
3. 简易版先不提供设置是否接收每日摘要和未交作业的可选项，默认都接收；


## 5. 拉取消息

### 5.1  拉取今日布置的作业

目前所有模板消息点击后均不会跳转到其他页面

基本流程： 

* 用户点击公众号的 “查询”——“今日作业”按钮 ；
* 公众号通过接口向sseserver发送GET请求，获取到作业清单、答题记录等数据，再进行组织；
* 然后公众号通过模板消息将今天布置的作业通过模板消息接口发送给用户；

注意：
1. “今日作业”包括今日新布置的作业（今天0点到当前查询时间）；
2. 也可以通过向公众号回复【作业】、【新作业】、【今日作业】来查询今日作业；
3. “获取今日作业”接口应根据OpenID先从家长表中获取家长绑定的学生的ID，然后再根据这个ID从其他表中获取到相关数据；


### 5.2 拉取今日未交作业

基本流程：

* 用户点击公众号的 “查询”——“未交作业”按钮 ；
* 公众号通过接口向sseserver发送GET请求，获取到作业清单、答题记录等数据，再进行组织；
* 然后公众号通过模板消息将今天的未交作业通过模板消息接口发送给用户；

注意：
1. “未交作业”包括今日未交作业（今天布置的作业中没有完成的，今日作业定义详见5.1）；
2. 也可以通过向公众号回复【未交】、【未交作业】来查询今日未交的作业；
3. “获取未交作业”接口应根据OpenID先从家长表中获取家长绑定的学生的ID，然后再根据这个ID从其他表中获取到相关数据；


### 5.3 拉取昨日作业批改

基本流程：

* 用户点击公众号的 “查询”——“昨日批改”按钮 ；
* 公众号通过接口向sseserver发送GET请求，获取到作业清单、答题记录等数据，再进行组织；
* 然后公众号通过模板消息将昨日批改通过模板消息接口发送给用户；

注意：
1. “昨日批改”包括昨天作业的成绩，这里的成绩是指：作业清单的平均分（计算方法：作业清单中每个大题的分数求平均，每个大题的分数取该答题下面分数最低的小题的分数）；
2. 也可以通过向公众号回复【昨日成绩】、【昨日批改】来查询昨天作业的批改情况；
3. “获取昨日批改”接口应根据OpenID先从家长表中获取家长绑定的学生的ID，然后再根据这个ID从其他表中获取到相关数据；


### 5.3 拉取一周摘要信息 （功能待定）

* 说明：通过图表方式展现一周的摘要信息；


## 6. 推送消息

### 6.1 推送昨日批改

功能： 每天晚上21点（待定）向家长推送学生昨天作业的批改成绩；

定时任务基本流程：

* 向sseserver服务器发送GET请求，根据家长表中家长表中的学生ID查询到该学生昨天的作业信息, 在服务端进行组织；
* 根据家长表中绑定该学生的家长的OpenID向微信服务器发送模板消息，将组织好到的消息通过模板消息发送给家长；

* 注意：
1. 定时逻辑可以通过使用nodeJS的“定时器”来实现，也可以通过使用NPM中提供的第三方包来实现（如：node-schedule）；


### 6.2 推送今日未交作业

功能： 每天晚上21点（待定）向家长推送所绑定学生的未交作业（待定：如果没有未交作业则不推送）

定时任务基本流程：

* 通过“获取未交作业通知”接口向sseserver服务器发送GET请求，根据家长表中家长表中的学生ID查询到该学生的未交作业；
* 根据家长表中绑定该学生的家长的OpenID向微信服务器发送模板消息，将查询到的学生未交作业发送给家长；


### 6.3 按需推送消息 (待讨论)

基本流程：

* 老师在老师端平板上发送“需求通知”（针对单个学生或者某些学生的通知）到公众号服务器；
* 公众号服务器向sseserver发送GET请求获取到这些学生绑定的家长的OpenID；
* 公众号服务器给这些家长发送通知；

* 注意：
1. “需求通知”应该包括：通知的内容（客户端获取好了再以JSON形式或者XML发送给服务器）、学生的ID；




## 7. 客服
微信服务号运营技术方案：
 * 1、所有服务号值班人员必须先用自己的微信号关注“作业家”服务号；
 * 2、值班员第一次当客服时需要先申请，后台人员给值班员发送一个验证码，值班员给公众号回复这个验证码，验证码一致的话后台人员将这个值班人员的OpenID加入客服表中；
 * 3、在开始值班时，值班员用自己的微信号向“作业家”服务号发送一条订阅消息：“客服上线”；
 * 4、后台检测到收到“客服上线”消息，检查发送消息的用户的OpenID是否在客服表中，如果不在，不做处理；
 * 5、用户OpenID在客服表中，将用户的zhiban字段设置为1，并回复一条消息：“客服上线成功，开始值班”（立即回复消息可以保证服务号后台能够在接下来的48小时内随意给这个OpenID发送消息）；
 * 6、当收到其他任何人发来的消息时，服务号后台查询该OpenID绑定的学生账户，并将该消息本身和其绑定的用户信息一起发送给zhiban字段为1的所有客服；
 * 7、值班员通过对比接收到的消息与微信服务号管理控制台中看到的消息，确定OpenID与微信用户名的对应关系，并从自己微信号接收到的消息中得知其绑定的学生账号信息。
 * 8、值班员下班，通过自己的微信号给“作业家”服务号发送退订消息：“客服下线”；
 * 9、服务号后台将该值班员的OpenID从当前值班员列表中删除，并回复一条消息：“客服下线成功”；
 * 10、每天晚上9点后台会统一将所有客服的zhiban字段设在为0.