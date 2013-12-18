LordOfPomelo's startup process is using the startup mode of Pomelo. Before reading the following content, please read [pomelo startup process](https://github.com/NetEase/pomelo/wiki/pomelo%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B). 
 
app.js is LordOfPomelo's entrance, is mainly responsible for all the servers' configuration, as well as loading and starting components. LordOfPomelo's startup process are divided into two steps: start the master server, start other servers respectively by the master server. 
 
### component configuration and loading
LordOfPomelo uses many external components, these components are loaded at the server startup, providing various services: such as data statistic, routing function replacement, game scene initialization, etc. 
 
LordOfPomelo uses a script-based statistics. These components collect server's running data and generate reports by running custom script. More specific functions, please see: 
 
``` javascript
	var sceneInfo = require('./app/modules/sceneInfo');
	var onlineUser = require('./app/modules/onlineUser');
	if(typeof app.registerAdmin === 'function'){
    app.registerAdmin('sceneInfo', new sceneInfo());
    app.registerAdmin('onlineUser',new onlineUser(app));
	}
```
 
When LordOfPomelo is starting, it will load the configuration files for areas, to establish a mapping between scenes and servers: 
 
``` javascript
  //Set areasIdMap, a map from area id to serverId.
	if (app.serverType !== 'master') {
	  var areas = app.get('servers').area;
	  var areaIdMap = {};
	  for(var id in areas){
	  	areaIdMap[areas[id].area] = areas[id].id;
	  }
	  app.set('areaIdMap', areaIdMap);
	}
```

In order to get the correct routing in multiple scene servers, LordOfPomelo loads custom routing component. By using the mapping information between scenes and servers, LordOfPomelo ensures that the player's requests are distributed to the corresponding scene server: 
 
``` javascript
  // route configures
	app.route('area', routeUtil.area);
	app.route('connector', routeUtil.connector);
```

In addition to the general configuration of the server, the "app.js" is also responsible for the initialization of different services: such as the global server's initialization, scene server's initialization, as well as the pathfinding server's initialization. Different servers will do different initialization process according to the type of server: 
 
``` javascript
app.configure('production|development', 'area', function(){
  app.filter(pomelo.filters.serial());
  app.before(playerFilter());
  //Load scene server and instance server
  var server = app.curServer;
  if(server.instance){
    instancePool.init(require('./config/instance.json'));
    app.areaManager = instancePool;
  }else{
    scene.init(dataApi.area.findById(server.area));
    app.areaManager = scene;
  }
  //Init areaService
  areaService.init();
});
```

Data synchronization plugin and MySql initialization: 
 
``` javascript
// Configure database
app.configure('production|development', 'area|auth|connector|master', function() {
  var dbclient = require('./app/dao/mysql/mysql').init(app);
  app.set('dbclient', dbclient);
  app.use(sync, {sync: {path:__dirname + '/app/dao/mapping', dbclient: dbclient}});
}); 
```
 
### server startup 
 
LordOfPomelo's startup process is using the startup mode of Pomelo framework, uses the master as a default component. The master component will be loaded and started in the "app.js" after "app.start()".
 
Master component is responsible for starting all other services. The startup process is divided into two phases: The first stage, the master service starts all other servers. After launching the server, the monitor component will connect to master's corresponding listener port, showing that the server's startup is completed. The second stage, after finishing all servers' startup, mater will call all "afterStart" interfaces on servers, to perform the work after starting. 

