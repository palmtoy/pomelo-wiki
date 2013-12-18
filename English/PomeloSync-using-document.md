## Overview
pomelo-sync is used to solve the problem that is to synchronize game data which need to be persisted to database or file between memory and storage system. This module provides an asynchronous implementation of the data synchronization. It can do timing data synchronization according to user's configuration to the persistence layer(such as MySQL, Redis, disk files, etc.). 
 
## Application scene
In the process of the game, updating and synchronizing a large amount of data occur from time to time(such as the role's location coordinates, HP, equipment properties, etc.). If we do much operations with the data persistence layer frequently, the cost of IO operation will be very enormous. With the method of timing synchronization a bulk of changed data is a way to avoid IO cost too much. pomelo-sync is designed and implemented according to this idea. pomelo-sync's configuration is like iBATIS', a different point is that the ORM is controlled completely by the user configuration with great flexibility. Therefore, pomelo-sync can be applied to a variety of databases(such as MongoDB, etc.). 
 
## Module structure 
pomelo-sync module internal structure is as follows: 
 
![](http://pomelo.netease.com/resource/documentImage/pomelo-sync.png) 
 
Logic Interface: It provides intrusive data updating Interface, including conventional "add", "update", "delete", and the immediate execution of persistence interface "flush". <br/> 
 
Invoke Interface: It provides the actual interfaces to perform persistence <br/>
Queue: It contains objects need to be saved in this period, supporting multiple nodes forwarding data <br/> 
Timer: It can support the static configuration and dynamic modification synchronization interval <br/> 
Mapping: It supports custom synchronization method mapping with persistence <br/> 
 
Store Interface: It encapsulates persistent interfaces of various storage engines <br/> 
Log: It provides AOF logging of the changed data <br/> 
MySQL, Redis, File: They perform user custom synchronization methods according to user-defined incoming object <br/> 
 
## Installation
npm install pomelo-sync-plugin <br/>
Note: Currently, pomelo-sync module is encapsulated in the [pomelo-sync-plugin](https://github.com/NetEase/pomelo-sync-plugin).
 
## Example of use
### To use pomelo-sync-plugin 
``` javascript
// app.js
var sync = require('pomelo-sync-plugin');
// ...
// Configure database
app.configure('production|development', 'area|auth|connector|master', function() {
  var dbclient = require('./app/dao/mysql/mysql').init(app);
  app.set('dbclient', dbclient);
  app.use(sync, {sync: {path:__dirname + '/app/dao/mapping', dbclient: dbclient}});
});
// ...
```
Note: All js files under the "/app/dao/mapping" directory will be loaded into the pomelo-sync. These modules define the timing synchronization operations of the persistence layer. Loading modules code as shown below: 
``` javascript
// pomelo-sync-plugin/node_modules/pomelo-sync/lib/dbsync.js
var DataSync = module.exports = function(options) {
  // ...
  this.mapping = this.loadMapping(options.mappingPath);
  // ...
};
```
 
### Configure synchronization object mapping relationship
The module supports user-defined synchronization methods, such as: 
``` javascript
// game-server/app/dao/mapping/taskSync.js
module.exports = {
  updateTask: function(dbclient, val, cb) {
    var sql = 'update Task set taskState = ?, startTime = ?, taskData = ? where id = ?';
    // ...
    var args = [val.taskState, val.startTime, taskData, val.id];
    dbclient.query(sql, args, function(err, res) {
    // ...
    });
  }
};
```

### Create sync object 
``` javascript
// pomelo-sync-plugin/lib/components/sync.js
var DataSync = require('pomelo-sync');
// ...
var createSync = function(opts){
  var opt = opts || {};
  opt.mappingPath = opts.path;
  opt.client = opts.dbclient;
  opt.interval = opts.interval || 60 * 1000;
  return new DataSync(opt);
};
```
 
## API 
### sync.exec(key,id,val,cb)
Adding asynchronous operations that are performed regularly
#### Arguments 
+ key - mapping keyword, need to be unique. 
+ id  - the primary key of the entity object. 
+ val - synchronization object, will be cloned after adding.
+ cb  - an asynchronous callback after synchronization, can be null. 

Specific example:
``` javascript
// app/dao/taskDao.js
// ...
task.on('save', function() {
  app.get('sync').exec('taskSync.updateTask', task.id, task);
});
// ...
```

### sync.flush(key,id,val,cb)
Sync an object to persistence layer immediately
#### Arguments 
+ key - mapping keyword, need to be unique. 
+ id  - the primary key of the entity object. 
+ val - synchronization object.
+ cb  - an asynchronous callback after synchronization, can be null. 

### sync.isDone()
Check whether there are data objects in the memory that are needed to be synchronized to persistence layer. Generally used with "sync()" method. 
 
### sync.sync()
Ignoring the timer, it performs the synchronization process immediately. Generally, it is used when the server is shutting down. The code is as follows: 
``` javascript
// pomelo-sync-plugin/lib/components/sync.js
Component.prototype.stop = function(force, cb) {
  // ...
  this.state = STATE_STOPED;
  this.sync.sync();
  var self = this;
  var interval = setInterval(function(){
    if (self.sync.isDone()) {
      clearInterval(interval);
      cb();
    }
  }, 200);
};
``` 

## Other parameter options 
The module's default synchronization interval is "1000 * 60"(1 minute). You can modify it by using the "opt.interval" parameter. <br/>
The "AOF" logging function is off by default in the module. If you want to open, you can use "opts.AOF=true" to open it. 
 
## Notice 
If you need support transactions operation, it should be dependent on the persistence layer's built-in transaction model to guarantee. 
 
## Others 
More detailed example, please refer to the module [source](https://github.com/NetEase/pomelo-sync) and [GameDemo](https://github.com/NetEase/lordofpomelo). 

