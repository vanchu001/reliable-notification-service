# reliable-notification-service
这是一个封装RabbitMQ功能来达到消息可靠广播的组件.

## 定义解释

|名词|解释|
|-|-|
|scope|作用范围,消息的发布及订阅将会在这个作用范围内进行|
|clientId|客户端id,客户端的**唯一**标识,将会与``作用范围``绑定,客户端id一旦确定后应不再变化(**万一客户端挂了,为了下次启动还能收到挂之前的信息**)|

## 基本原理
以``e_${scope}``值为名自动创建持久性的Exchange,以``q_${scope}_${clientId}_${event}``值为名创建持久性的Queue,自动将Exchange与Queue绑定.
当向Exchange发布事件消息时,所有与事件绑定的Queue将会收到消息.组件内部将会根据订阅的事件调用对应处理函数.处理函数没报错/没订阅该事件的话,组件将会发送Ack给RabbitMQ.若报错的话**不发Ack并停止接受该类事件信息(此时Queue可以接受Exchange发来的信息,但不推送给客户端),等故障解决后重启服务,组件将会一一将之前未处理的信息再次传给客户端.**
故使用该组件时,应做好**处理失败**行为**监控**(处理失败会发送``error``事件)
## 用法

### 连接参数

|参数名|含义|必填|
|-|-|-|
|host|RabbitMQ Host|Y|
|port|RabbitMQ Port|Y|
|login|RabbitMQ Username|Y|
|password|RabbitMQ Password|Y|
|heartbeat|心跳包,默认10(秒)|N|
|vhost|RabbitMQ VHost,默认/|N|
|reconnect|断线重连停歇时长,默认2(秒)|N|
|ssl|使用ssl方式连接,默认undefined|N|
|ssl.pfx与ssl.passphrase|使用pkcs12证书|N|
|ssl.cert与ssl.key|使用cert+key证书|N|

### 核心方法

|方法名|作用|参数|备注|
|-|-|-|-|
|registrySubscriber|订阅事件及处理函数|(事件名, Function(data))|**处理函数若报错将不会确认信息被处理,并停止接下来的该事件信息接受**|
|publish|发布事件及内容|(事件名, 内容)|JSON.stringify可转换内容都支持|

### 用法

```js
const Notification = require('reliable-notification-service');

let Test = require('../client');
let test = new Test(connParams, 'test', '123456');

(async() => {
    const connParams = {
        host: '192.168.56.56',
        port: 5672,
        login: 'admin',
        password: 'admin'
    }
    const scope = "Test";
    const clientId = "12345";
    let notification = new Notification(connParams, scope, clientId);
    //监听出错事件(包括连接出错,处理函数处理错误)
    notification.on('error', (error) => {
        console.log(error);
    });
    notification.on('close', () => {
        console.log('close');
    });

    //初始化
    await notification.init();

    //订阅事件及处理函数
    notification.registrySubscriber('event1', (data) => {
        console.log('event1:' + JSON.stringify(data));
        // throw new Error('1111')
    });
    notification.registrySubscriber('event2', (data) => {
        console.log('event2:' + JSON.stringify(data));
    });

    setInterval(async() => {
        try {
            //发布事件及内容
            await notification.publish('event1', {a: 1, b: 2});
        }
        catch(error) {
            console.log(error)
        }
    }, 50)

})()
```

## V1.2.0 版本内容更新

`registrySubscriber` 方法新增第三个参数 `queueSuffix`（默认为 `undefined`）

若调用者，传入了 `queueSuffix` ，则 Queue 名变为 `q_${scope}_${clientId}_${event}_${queueSuffix}`。该更新的目的为将对同一个事件所需要进行的操作原子化 —— 避免操作Ａ失败，导致操作Ｂ无法进行。

例如：

```javascript
function a() {
    // just do something
    throw new Error('oh! no!!!');
}
function b() {
    // just do something
}
notification.registrySubscriber('event1', (data) => {
    a();b();
});
```

因为 `a` 函数的报错，导致 `b` 函数无法进行，但我们是希望 `a` 和 `b` 是分别独立的，不相互影响，此时我们可以改成如下的形式：

```javascript
function a() {
    // just do something
    throw new Error('oh! no!!!');
}
function b() {
    // just do something
}
notification.registrySubscriber('event1', (data) => a(), 'a');
notification.registrySubscriber('event1', (data) => b(), 'b');
```

