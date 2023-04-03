# chatgpt-web-java

# 分支 main

## 介绍 

- [Chanzhaoyu/chatgpt-web](https://github.com/Chanzhaoyu/chatgpt-web) 项目的 Java 后台
- 该分支关联项目的 [2.10.8](https://github.com/Chanzhaoyu/chatgpt-web/releases/tag/v2.10.8) 版本，在不改动前端的情况下更新后台
- [管理端开源代码](https://github.com/hncboy/chatgpt-web-admin)

## 框架

- Spring Boot 2.7.10
- JDK 17
- MySQL 8.x
- SpringDoc 接口文档
- MyBatis Plus
- MapStruct
- Lombok
- [Hutool](https://hutool.cn/) 
- [SaToken](https://sa-token.cc/) 权限校验
- [Grt1228 ChatGPT java sdk](https://github.com/Grt1228/chatgpt-java)

## 地址

- 接口文档：http://localhost:3002/swagger-ui.html
- 客户端：https://front.stargpt.top/ 密码：stargpt
- 管理端：https://admin.stargpt.top/ 账号密码 admin-admin

## 已实现功能

### 上下文聊天

通过 MySQL 实现聊天数据存储来实现 apiKey 方式的上下文聊天，AccessToken 默认支持上下文聊天。可以通过配置参数 limitQuestionContextCount 来限制上下问问题的数量。

数据库存储了每次聊天对话的记录，在选择上下文聊天时，通过 parentMessageId 往上递归遍历获取历史消息，将历史问题以及回答消息都发送给 GPT。

![](pics/question_context_limit_test.png)

### 敏感词过滤

在项目启动时会将敏感词文件 sensitive_word_base64.txt 的数据导入到敏感词表，目前还未提供后台管理敏感词的接口，提供后这种方式可以去掉。在文件中敏感词以 base64 形式存放。并将敏感词表的数据构建到 HuTool 提供的 WordTree 类中。在发送消息调用方法判断是否属于敏感词，是的话消息发送不成功。为了兼容前端保持上下文关系，在消息内容属于敏感词的情况下会正常返回消息格式，但是带的是请求的的 conversationI 和 parentMessagId。

![](pics/sensitive_word_test.png)

### 限流

分为全局限流和 ip 限流，基于内存和双端队列实现滑动窗口限流。在限流过程会异步的将数据写入的文件中，在项目重启时会读取该文件恢复限流状态。

在配置文件中配置 maxRequest、maxRequestSecond、ipMaxRequest、ipMaxRequestSecond

![](pics/rate_limit_test.png)

## 待实现功能

- GPT 接口异常信息特定封装返回
- 其他没发现的点

## 存在问题

- 在接口返回报错信息时，不会携带 conversationid 和 parentMessageId，导致前端下一次发送消息时会丢失这两个字段，丢失上下文关系。

## 管理端

### 消息记录

展示消息的列表，问题和回答各是一条消息。通过父消息 id 关联上一条消息。父消息和当前消息一定是同一个聊天室的。

![](pics/chat_message_1.png)

### 限流记录

查看各个 ip 的限流记录，只记录在限流时间范围的限流次数。

![](pics/rate_limit_1.png)

### 聊天室管理

查看聊天室。这里的聊天室和客户端左边的对话不是同一个概念。在同一个窗口中，我们既可以选择关联上下文发送后者不关联上下文发送。如果不关联上下文发送每次发送消息都会产生一个聊天室。

![](pics/chat_room_1.png)

### 敏感词管理

查看敏感词列表，目前只提供了查询的功能，后期可以增加管理。

![](pics/sensitive_word_1.png)

## 接口

| 路径          | 功能         | 完成情况 |
| ------------- | ------------ | -------- |
| /config       | 获取聊天配置 | 已完成   |
| /chat-process | 消息处理     | 已完成   |
| /verify       | 校验密码     | 已完成   |
| /session      | 获取模型信息 | 已完成   |

## chat-gpt配置

- 有2个配置文件`application-dev.yml`和`application-prod.yml`，对应两个环境dev和prod，不指定profile默认使用dev。
- 指定profile的方式有两种
   - 是在`application.yml`的`spring.profiles.active`配置项
   - 参考下节docker-compose运行
- 配置项在配置yaml的chat节点下，具体可以参考代码注释。

## 运行

### IDEA 运行

需要本地提前准备好端口为3309的MySQL实例，如果没有可以直接使用Dockerfile_mysql构建一个docker的MySQL容器：

```shell
  # 删除旧版container（如果有的话）
  docker stop mysql_gpt && docker rm mysql_gpt
  # 构建image
  docker build -t mysql_gpt_img:latest . -f Dockerfile_mysql
  # 运行container
  docker run -d -p 3309:3306 \
       --name mysql_gpt \
       -v ~/mydata/mysql_dummy/data:/var/lib/mysql \
       -v  ~/mydata/mysql_dummy/conf:/etc/mysql/conf.d \
       -v ~/mydata/mysql_dummy/log:/var/log/mysql \
       mysql_gpt_img:latest
```

之后使用`chatgpt-bootstrap`下的`ChatGptApplication`类启动即可。

这里也提供下java应用构建镜像的方法。

```shell
  # 删除旧版container（如果有的话）
  docker stop chatgpt-web-java && docker rm chatgpt-web-java
  docker build -t chatgpt-web-java .
  docker run -d -p 3002:3002 chatgpt-web-java
```
如果要显式指定DB和chat-gpt参数，可以在docker run后添加-e选项，配置application.yml用到的参数。例如：

```shell
  # 删除旧版container（如果有的话）
  docker stop chatgpt-web-java && docker rm chatgpt-web-java
  docker build -t chatgpt-web-java . 
  # 如果这里要使用java的容器访问mysql容器，需要使用host.docker.internal而不是localhost，才可以访问到宿主机的3009端口（mysql开放了3009端口）
  docker run -d -p 3002:3002 \
      -e '--spring.datasource.url=jdbc:mysql://host.docker.internal:3309/chat?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai' \
      -e --spring.datasource.username=root \
      -e --spring.datasource.password=123456 \
      -e --chat.openai_api_key=xxx \
      -e --chat.openai_access_token=xxx \
      -e --chat.openai_api_base_url=http://xxx.com \
      -e --chat.http_proxy_host=127.0.0.1 \
      -e --chat.http_proxy_port=7890 \
      chatgpt-web-java
```
  ![](pics/docker_run.png)

### docker-compose运行

使用`compile_build_up.sh`可一键启动，具体使用方法：

```shell
cd chatgpt-web-java
# 增加执行权限
chmod +x compile_build_up.sh
# 启动
# dev环境
./compile_build_up.sh -e dev
# prod环境 
./compile_build_up.sh -e prod 
```

## 表结构

具体可以参见sql文件，路径是`chatgpt-bootstrap/src/main/resources/db/schema-mysql.sql`。

- 聊天室表

| 列名                  | 数据类型     | 约束             | 说明                       |
| --------------------- | ------------ | ---------------- | -------------------------- |
| id                    | BIGINT       | PRIMARY KEY      | 主键                       |
| ip                    | VARCHAR(255) |                  | ip                         |
| conversation_id       | VARCHAR(64)  | UNIQUE, NULL     | 对话 id，唯一              |
| first_chat_message_id | BIGINT       | UNIQUE, NOT NULL | 第一条消息主键，唯一       |
| first_message_id      | VARCHAR(64)  | UNIQUE, NOT NULL | 第一条消息 id，唯一        |
| title                 | VARCHAR(255) | NOT NULL         | 对话标题，从第一条消息截取 |
| api_type              | VARCHAR(20)  | NOT NULL         | API 类型                   |
| create_time           | DATETIME     | NOT NULL         | 创建时间                   |
| update_time           | DATETIME     | NOT NULL         | 更新时间                   |

- 聊天记录表

| 列名                       | 数据类型      | 约束        | 说明                     |
| -------------------------- | ------------- | ----------- | ------------------------ |
| id                         | BIGINT        | PRIMARY KEY | 主键                     |
| message_id                 | VARCHAR(64)   | NOT NULL    | 消息 id                  |
| parent_message_id          | VARCHAR(64)   |             | 父级消息 id              |
| parent_answer_message_id   | VARCHAR(64)   |             | 父级回答消息 id          |
| parent_question_message_id | VARCHAR(64)   |             | 父级问题消息 id          |
| context_count              | BIGINT        | NOT NULL    | 上下文数量               |
| question_context_count     | BIGINT        | NOT NULL    | 问题上下文数量           |
| message_type               | INTEGER       | NOT NULL    | 消息类型枚举             |
| chat_room_id               | BIGINT        | NOT NULL    | 聊天室 id                |
| conversation_id            | VARCHAR(64)   |             | 对话 id                  |
| api_type                   | VARCHAR(20)   | NOT NULL    | API 类型                 |
| ip                         | VARCHAR(255)  |             | ip                       |
| api_key                    | VARCHAR(255)  |             | ApiKey                   |
| content                    | VARCHAR(5000) | NOT NULL    | 消息内容                 |
| original_data              | TEXT          |             | 消息的原始请求或响应数据 |
| response_error_data        | TEXT          |             | 错误的响应数据           |
| prompt_tokens              | BIGINT        |             | 输入消息的 tokens        |
| completion_tokens          | BIGINT        |             | 输出消息的 tokens        |
| total_tokens               | BIGINT        |             | 累计 Tokens              |
| status                     | INTEGER       | NOT NULL    | 聊天记录状态             |
| is_hide                    | TINYINT       | NOT NULL    | 是否隐藏 0 否 1 是       |
| create_time                | DATETIME      | NOT NULL    | 创建时间                 |
| update_time                | DATETIME      | NOT NULL    | 更新时间                 |

- 敏感词表

| 字段名      | 数据类型     | 约束        | 描述                      |
| ----------- | ------------ | ----------- | ------------------------- |
| id          | BIGINT       | PRIMARY KEY | 主键                      |
| word        | VARCHAR(255) | NOT NULL    | 敏感词内容                |
| status      | INTEGER      | NOT NULL    | 状态，1为启用，2为停用    |
| is_deleted  | INTEGER      | NULL        | 是否删除，0为否，NULL为是 |
| create_time | DATETIME     | NOT NULL    | 创建时间                  |
| update_time | DATETIME     | NOT NULL    | 更新时间                  |

# 联系

<div style="display: flex; align-items: center; gap: 20px;">
  <div style="text-align: center">
    <img style="max-width: 100%" src="pics/wechat_group.png" alt="微信" />
    <p>微信群</p>
  </div>
</div>
<div style="display: flex; align-items: center; gap: 20px;">
  <div style="text-align: center">
    <img style="max-width: 100%" src="pics/qq_group.png" alt="QQ" />
    <p>631171246</p>
  </div>
</div>


# 赞助

如果觉得项目对你有帮助的，条件允许的话可以点个 Star 或者在赞助一小点。感谢支持~

<div style="display: flex; align-items: center; gap: 20px;">
  <div style="text-align: center">
    <img style="max-width: 100%" src="pics/wechat_pay.png" alt="微信" />
    <p>微信支付</p>
  </div>
  <div style="text-align: center">
    <img style="max-width: 100%" src="pics/zhifubao_pay.png" alt="支付宝" />
    <p>支付宝</p>
  </div>
</div>

## License

MIT © [hncboy](license)
