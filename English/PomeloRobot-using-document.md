# pomelo-robot 
pomelo-robot is a performance test tool used for testing the pomelo game server framework, also can test performance of the other services based on socket.io. 
This module can be used standalone or distributed test modes. 
 
# Function 
This module's function is performance test and analysis for the game project automatically. It provides robot and script to game server, finally outputs performance test and analysis report. 
Module pomelo-robot runs user-defined JS script in a sandbox. In the process of running, clients will report data to the master node automatically. The master node refine and summarize all the nodes' test data. The master node calculates the average response time and some other statistics, and sends statistic data regularly to the built-in HTTP server to display in the web. 
 
# Module structure
Module's internal operating structure is as follows: 
 
![](https://raw.github.com/palmtoy/ImageBucket/master/Pomelo/Robot/1.jpg)
 
"Master" is responsible for collecting all the running data of "Client" and show results. <br/> 
"Client" is responsible for runnint custom script in multiple sandboxs("User"), reporting running data to the "Master" at the same time. 
 
## Example of use
Let's create a Node.js test project. The project directory structure is as shown below: <br/> 
 
![](https://raw.github.com/palmtoy/ImageBucket/master/Pomelo/Robot/2.jpg)
 
Installation of dependent libraries: <br/> 
``` javascript
npm install pomelo-robot
```
 
### config.json configuration 
"config.json" is divided into two kinds of running environments: development(dev) and production(prod) environments. "prod" environment's file content is as follows: 
``` javascript
{
  "master": {"host": "127.0.0.1", "port":8888, "webport":8889, "interval":500},
}
```
"master": master server IP, communication ports with client, web interface port, running a custom JS script's interval(such as: lord.js). <br/> 
 
### To implement a custom configuration script
##### In the "env.json", configuring custom script path:<br/> 
``` javascript
{
  "env": "prod",
  "script": "/app/script/lord.js"
}
```

##### To implement custom script "lord.js":
``` javascript
// ...
var queryHero = require(cwd + '/app/data/mysql').queryHero;
// ...
function entry(host, port, token, callback) {
  // ...
  // initialize socketClient
  pomelo.init({host: host, port: port, log: true}, function() {
    pomelo.request('connector.entryHandler.entry', {token: token}, function(data) {
	  // ...
      afterLogin(pomelo,data);
    });
  });
}
// ...
```
 
In "/app/data/mysql.js", the code which is looking for the login roles is as follows: 
``` javascript
// ...
queryHero = function(client,limit,offset,cb){
    var users = [];
    var sql = "SELECT User.* FROM User,Player where User.id = Player.userId and User.name like 'pomelo%' limit ? offset ? ";
	// ...
};
// ...
``` 
 
The above code is running in a sandbox. Firstly, a role logins game server. After the role's login, we call the "afterLogin" function. Users can do some follow-up related operations. <br/> 
 
### The entry file:"app.js"
"app.js" can judge according to launch parameters to start the "master" or "client" services. <br/> 
``` javascript
var envConfig = require('./app/config/env.json');
var config = require('./app/config/' + envConfig.env + '/config');
//...
if (mode === 'master') {
    robot.runMaster(__filename);
} else {
    var script = (process.cwd() + envConfig.script);
    robot.runAgent(script);
}
// ...
``` 
Specific code can refer to [pomelo-robot-demo](https://github.com/NetEase/pomelo-robot-demo). <br/>
 
### Run the test 
You can run the following commands to start the "master" service: <br/> 
"node app.js master" <br/>
Open a browser to access "http://masterIp:8889" <br/> 
 
You can Run the following commands to start the "client" service: <br/> 
"node app.js client" <br/>
Note: You can start "client" services on multiple machines to do performance and stress testing. 
 
WEB interface will display the number of "client" which connects to the "master". You can configure the the number of the sandbox for each "client" at the column of "Per Agent Users". You can click on the "Go" button to notify all of the clients began to run. The WEB interface will obtain the background data regularly to display. <br/> 
Running interface as shown below: <br/> 
 
![](https://raw.github.com/palmtoy/ImageBucket/master/Pomelo/Robot/3.jpg) 

![](https://raw.github.com/palmtoy/ImageBucket/master/Pomelo/Robot/4.jpg) 

### Other
Please refer to the source code[pomelo-robot](https://github.com/NetEase/pomelo-robot) 

