# Websocket
通过Websocket，客户端可以与开黑啦进行实时通信，来接收事件和数据。websocket的协议交互非常复杂，而且较差的实现会给服务端和客户端都带来较大困扰，因此建议你在编写自己的实现时，详细阅读本文档。  

**重要提示：** 并不是所有的字段都有文档记录，你应该依赖文档，而不是依赖接口中的字段。我们可能随时更改不在文档中的字段。  

**注意：** Webhook 模式与 Websocket 模式是互斥的，如果选择了 Webhook 模式，将不能再使用 Websocket 接收用户消息。同理，设置了 Websocket 后，平台将不会再向回调地址推送消息。

## Gateway
Gateway是websocket的网关，客户端通过连接Gateway可以获取到相应的推送消息等。

Gateway的获取需要走http接口获取，参见[Gateway](https://developer.kaiheila.cn/doc/http/gateway)

## 消息压缩

- 如果客户端连接中 `compress` 参数为 `1`, 所有方向为 `server->client` 的消息都是经过压缩后的`binary` 类型的消息.
- 与Webhook保持一样，默认情况下，我们的数据会进行 zlib 压缩 (deflate)，相应的数据你可能需要先进行 zlib 解压缩，再进行处理。如果不需要压缩，可以在获取gateway时加上参数`compress=0`。
- 客户端发给服务端的消息不要压缩。

## 连接流程

常规连接流程如下：  
1. 获取Gateway  
2. 连接Gateway。如果连接失败，回退到第1步。
3. 收到hello包，如果成功，开始接收事件。如果失败，回退至第1步。
4. 在连接中，每隔30秒发一次心跳ping包，如果6秒内，没有收到心跳pong包，则超时。进入到指数回退，重试。
5. 先发两次心跳ping(间隔为2,4),判断连接是否成功。如果成功，则连接恢复。
6. 如果不成功，再回退到第2步，尝试两次resume(间隔为8,16)。如果成功，会按正常往下走，但有一个resume过程（同步中间的离线消息），resume完了，会收到一个resumeOK包。
7. 如果失败，再回到第1步，尝试无数次获取Gateway(指数倒退，最大间隔为60),直到成功为止。
8. 任何时候，收到reconnect包，应该将当前消息队列，`sn`等全部清空，然后回到第1步，否则可能会有消息错乱等各种问题。

连接流程示意图：
![image](/img/state.png)

参考代码: [php-bot](https://github.com/kaiheila/php-bot/blob/main/src/base/StateSession.php)


## 信令格式

### 信令基本格式

```javascript
{
    "s" : 1,  // int, 信令，详情参照信令说明
    "d" : [], // 数据字段mixed
    "sn" : 0, // int, 该字段并不一定有，只在s=0时有，与webhook一致。
}
```
具体参见[Event](https://developer.kaiheila.cn/doc/event)

### 信令说明
|信令|方向|说明|
|--|--|--|
|0|server->client|消息(包含聊天和通知消息)|
|1|server->client|客户端连接 ws 时, 服务端返回握手结果|
|2|client->server|心跳，ping|
|3|server->client|心跳，pong|
|5|server->client|reconnect, 要求客户端断开当前连接重新连接|
|6|server->client|resume ack|


## 信令[1] HELLO

**方向：** server->client  
**说明：** 当我们成功连接websocket后，客户端应该在6s内收到该包，否则认为连接超时。  
**成功示例：**

```json
{
    "s": 1,
    "d": {
        "code": 0,
        "sessionId": "xxxx"
    }
}
```

**失败：**

| 状态码 | 含义          | 备注         |
| ------------- | ------------- | ------------ |
| 40100         | 缺少参数      |              |
| 40101         | 无效的token   |              |
| 40102         | token验证失败 |              |
| 40103         | token过期     | 需要重新连接 |

**示例：**
```javascript
{
    "s": 1,
    "d": {
        "code": 40103
    }
}
```



## 信令[0] EVENT

**方向：** server->client  
**说明：** 在正常连接状态下，收到的消息事件等。  
**参数列表：**

具体参见[Event](https://developer.kaiheila.cn/doc/event)

**注意：** 该消息会有 `sn`, 代表消息序号, 针对当前 `session` 的消息的序号, 客户端需记录该数字,并按顺序接收消息， **resume** 时需传入该参数才能完成。


**示例：**

```javascript
{
    "s": 0,
    "d": {
        "channelType": "GROUP",
        "toUserId": "xxx",
        "fromUserId": "1",
        "content": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
        "msgId": "bxx5",
        "msgTimestamp": 1605499238
    },
    "sn": 1000
}
```

**注意事项：**

1. 收到消息时需要按照 `sn`  顺序处理, 服务端会尽可能保证 `sn` 的顺序性
2. 假设收到消息的 `sn` 出现乱序, 需要先存入暂存区 (`buffer`) 等待正确的 `sn` 消息处理后再从暂存区顺序处理
3. 假设收到了一条已处理过的 `sn` 的消息, 则直接抛弃不处理 
4. 客户端需要存储当前已处理成功的最大的 `sn`, 待心跳ping时回传服务端, 如果服务端发现当前客户端最新处理成功的消息 `sn` 落后于最新消息 (丢包等异常情况), 服务端将会按照客户端指定的 `sn` 将之后所有最新的消息重传给客户端.  
5. 消息内容与webhook保持一致  


## 信令[2] PING

**方向：** client -> server  
**说明：** 每隔30s(随机-5，+5),将当前的最大 `sn` 传给服务端,客户端应该在6s内收到PONG, 否则心跳超时。  
**参数列表：**

| 参数 | 描述                              | 类型 | 必传 |
| ---- | --------------------------------- | ---- | ---- |
| sn   | 客户端目前收到的最新的消息 **sn** | int  | Y    |


**示例：**
```javascript
{
    "s": 2,
    "sn": 6
}
```

**注意事项：**
1. 心跳间隔： 30秒 + rand(-5,5)秒 
2. 如果发了ping, 6秒内没有收到pong，我们应该进入到超时状态。


## 信令[3] PONG

**方向：**  server -> client  
**说明：** 回应客户端发出的ping  
**示例：**  
```javascript
{
    "s": 3
}
```

## 信令[5] RECONNECT

**方向：** server->client

**说明：** 服务端通知客户端, 代表该连接已失效, 请重新连接。客户端收到后应该主动断开当前连接。

**注意：**  客户端收到该信令代表因为某些原因导致当前连接已失效, 需要进行以下操作以避免消息丢失.
1. 重新获取 gateway;
2. 清空本地的 sn 计数;
3. 清空本地消息队列.

| 状态码 | 描述                                    |      |
| ------------ | --------------------------------------- | ---- |
| 40106        | resume 失败, 缺少参数                   |      |
| 40107        | 当前 `session` 已过期 (resume 失败, PING的sn无效)      |      |
| 40108        | 无效的 `sn` ,  或 `sn` 已经不存在 (resume 失败, PING的 `sn` 无效) |      |

**示例：**
```javascript
{
    "s": 5
    "d": {
        "code": 41008,
        "err": "Missing params"
    }
}
```



## 信令[6] RESUME ACK

**方向：** server->client  
**说明：** 服务端通知客户端 resume 动作成功，中间所有离线消息已经全部发送成功   
**示例：**  

```javascript
{
    "s": 6
    "d": {
        "sessionId": "xxxx-xxxxxx-xxx-xxx"
    }
}
```

