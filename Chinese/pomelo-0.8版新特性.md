##安全性方面

### RPC调用的IP白名单

该功能可以为各个服务器的RPC调用提供IP白名单过滤功能.

###### 原理

RPC服务端每接受一个连接都会抛出一个连接事件, 这个事件中含有该连接的socket.id和RPC客户端IP. RPC服务端会捕获该连接事件, 并调用用户传入的获取IP白名单的函数, 如果该RPC客户端IP不在白名单中, 则立刻将对应的socket断开. 以此来实现RPC调用白名单过滤功能.

###### 使用

使用时只需要向`remoteConfig`的配置中传入一个获取IP白名单的函数(`whitelist: rpcWhitelist.whitelistFunc`)即可, 这个函数需要接受一个回调函数作为其参数, 该回调函数形如`function(err, tmpList) {...}`. 在获取IP白名单的函数内, 拿到IP白名单时(该白名单应为一维`JS Array`), 以类似于`cb(null, self.gWhitelist)`的形式调用IP过滤回调函数.

```
./game-server/app/util/whitelist.js
... ...
var self = this;
self.gWhitelist = ['192.168.146.100', '192.168.146.101'];
module.exports.whitelistFunc = function(cb) {
  cb(null, self.gWhitelist);
  };
... ...
```

```
./game-server/app.js
var rpcWhitelist = require('./app/util/whitelist');
... ...
// configure for global
app.configure('production|development', function() {
... ...
  // remote configures
  app.set('remoteConfig', {
    cacheMsg: true
    , interval: 30
    , whitelist: rpcWhitelist.whitelistFunc
  });
... ...
}
```

* 具体请参考[lordofpomelo](https://github.com/NetEase/lordofpomelo/tree/dev)

### pomelo-admin的IP白名单

该功能可以为master服务器的admin提供IP白名单过滤功能.

###### 原理

admin服务端每接受一个连接都会抛出一个连接事件, 这个事件中含有该连接的socket.id和admin客户端IP. admin服务端会捕获该连接事件, 并调用用户传入的获取IP白名单的函数, 如果该admin客户端IP不在白名单中, 则立刻将对应的socket断开. 以此来实现master服务器的admin白名单过滤功能.

###### 使用

使用时只需要向`masterConfig`的配置中传入一个获取IP白名单的函数(`whitelist: adminWhitelist.whitelistFunc`)即可, 这个函数需要接受一个回调函数作为其参数, 该回调函数形如`function(err, tmpList) {...}`. 在获取IP白名单的函数内, 拿到IP白名单时(该白名单应为一维`JS Array`), 以类似于`cb(null, self.gWhitelist)`的形式调用IP过滤回调函数.

```
./game-server/app/util/whitelist.js
... ...
var self = this;
self.gWhitelist = ['192.168.146.100', '192.168.146.101'];
module.exports.whitelistFunc = function(cb) {
  cb(null, self.gWhitelist);
};
... ...
```

```
./game-server/app.js
var adminWhitelist = require('./app/util/whitelist');
... ...
// configure for global
app.configure('production|development', function() {
... ...
  app.set('masterConfig', {
    authUser: app.get('adminAuthUser') // auth client function
    , authServer: app.get('adminAuthServerMaster') // auth server function
    , whitelist: adminWhitelist.whitelistFunc
  });
... ...
}
```

* 具体请参考[lordofpomelo](https://github.com/NetEase/lordofpomelo/tree/dev)


## csv配置文件自动热加载插件pomelo-data-plugin

该插件可以监控指定目录下的所有csv格式的配置文件, 并在某个文件被改变时自动地将其热加载进入Pomelo. 

###### 原理

该插件主要使用[csv模块](https://npmjs.org/package/csv)来解析csv配置文件, 使用`fs.watchFile`函数来监控文件变化事件. 当Pomelo框架启动该插件中的组件时, 组件会加载给定文件夹中的所有csv配置文件, 并为每个文件加一个watcher. 以此来实现csv配置文件自动热加载的功能.

###### 安装

```
npm install pomelo-data-plugin
```

###### 使用

```
./pomelo-data-plugin-demo/config/data/team.csv
# 队伍名称配置表
# 队名编号, 队名
id,teamName
5,The Lord of the Rings
6,The Fast and the Furious
```

上面的csv配置文件中, 以`#`开头的行是注释语句, 正文的第一行应为列名(`id`为该文件的主键列名, 为必须列), 下面的行是对应列名的具体数据.

```
var dataPlugin = require('pomelo-data-plugin');
... ...
app.configure('production|development', function() {
  ...
  app.use(dataPlugin, {
    watcher: {
      dir: __dirname + '/config/data',
      idx: 'id',
      interval: 3000
    }
  });
  ...
});
... ...
... ...
var npcTalkConf = app.get('dataService').get('npc_talk');
... ...
... ...
```

上面代码中的`dir`即为需要监控的配置文件夹; `idx`为`所有`csv配置文件的主键列名(如:`team.csv`所示的`id`); `interval`为`fs.watchFile`函数测试其所监控文件改变的时间间隔, 单位为毫秒. 

* 具体请参考[pomelo-data-plugin](https://github.com/palmtoy/pomelo-data-plugin)和[pomelo-data-plugin-demo](https://github.com/palmtoy/pomelo-data-plugin-demo)


