# Distributed deployment of Pomelo (take LordOfPomelo as an example)

### Methods and steps of the distributed deployment 
#### 1. Build and configure the system and application software environment 
All machines which are participanting in the distributed deployment:

* must be the same type of operating system (recommended for the same operating system. In this article, 4 machine's operating system are "Debian GNU/Linux 7.0"). 
* must have a same user name (such as "pomelo" and so on. In this article, every machine has a user named "pomelo"). 
* Node.js version must be identical. The `absolute path` to the installation must also be identical (In this article, the absolute path to the installation is "/home/pomelo/node-v0.10.21-linux-x64"). 
* The `absolute path` of "lordofpomelo" must also be identical (In this article, the absolute path is "/home/pomelo/lordofpomelo"). 
* Configure the `ssh login options` in all machines participating in a distributed deployment. The method is: to create a file called "config"(In this article, the directory is "/home/pomelo/.ssh") in "~/.ssh" directory. The file content is as follows: 

```
Host *
HashKnownHosts no
CheckHostIP no
StrictHostKeyChecking no
```

The purpose of the above-mentioned file is to make ssh login smoothly between machines. The meaning of each option, please refer to the [ssh_config](http://man.he.net/man5/ssh_config). 

#### 2. Install Pomelo in the global environment; install packages which lordofpomelo depends on 

```
$ npm install pomelo -g
$ cd lordofpomelo
$ sh npm-install.sh
```

Detailed steps, please refer to [Pomelo Installation](https://github.com/NetEase/pomelo/wiki/Installation) and [LordOfPomelo Installation Guide](https://github.com/NetEase/pomelo/wiki/LordOfPomelo-installation-guide).


#### 3. Modify the related configuration files of lordofpomelo

* Modify "lordofpomelo/shared/config/mysql.json": change the `host` address to the IP address of the mysql machine. Note: Do not fill out the "127.0.0.1" or "localhost". The specific configuration as shown below, you can modify the corresponding configuration items according to actual situation: 

```json
{
	"development": {
	  "host" : "pomelo3.server.163.org",
	  "port" : "3306",
	  "database" : "Pomelo",
	  "user" : "xy",
	  "password" : "dev"
	},
	"production": {
	  ...
	}
}
```

* Modify "lordofpomelo/game-server/config/master.json": change the `host` address to the `master` machine IP address (That is the machine's IP which you will use `pomelo start` to start `game-server` server cluster). Note: Do not fill out the "127.0.0.1" or "localhost". The specific configuration as shown below, you can modify the corresponding configuration items according to actual situation:

```json
{
    "development":{
        "id": "master-server-1", "host": "pomelo16.server.163.org", "port": 3005
    },
    "production":
    {
        ...
    }  	
}
```

* Modify "lordofpomelo/game-server/config/servers.json": change the `host` address to the machine's IP address which the corresponding service process is running on (That is the machine which the service process will run on). Note: Do not fill out the "127.0.0.1" or "localhost". The specific configuration as shown below, you can modify the corresponding configuration items according to actual situation:

```json
{
	"development": {
		...
		"area": [
			{"id": "area-server-1", "host": "pomelo16.server.163.org", "port": 3250, "area": 1},
			{"id": "area-server-2", "host": "pomelo18.server.163.org", "port": 3251, "area": 2},
			{"id": "area-server-3", "host": "pomelo19.server.163.org", "port": 3252, "area": 3},
			...
		],
		...
		"gate": [
			{"id": "gate-server-1", "host": "pomelo16.server.163.org", "clientPort": 3014, "frontend": true}
		],
		...
	},
	"production": {
		...
	}
}
```

* Modify "lordofpomelo/game-server/config/servers.json": change the `GATE_HOST` and `GATE_PORT` to the machine IP address and port which the `game-server's gate service process` is running on. Note: If the `web server` and `game-server's gate service process` are running on the same machine, `GATE_HOST` can be configured to `window.location.hostname`, otherwise, it should be configured to the corresponding IP. This configuration should be correspond to `gate` configuration in "lordofpomelo/game-server/config/servers.json". Specific configuration as shown below, you can modify the corresponding configuration items according to actual situation: 

```javascript
...
    IMAGE_URL: 'http://pomelo.netease.com/art/',
    GATE_HOST: 'pomelo16.server.163.org',
    GATE_PORT: 3014
...
```

After the above steps are completed, you can use `pomelo start` command in the `lordofpomelo/game-server` directory to start `game-server` server cluster on the `master` machine(Example in this article is "pomelo16.server.163.org"). You can use `pomelo stop` command in the `lordofpomelo/game-server` directory to stop `game-server` server cluster. You can use the `node app.js` command in the `lordofpomelo/web-server` directory to start `web-server` in another machine(The example in this article is "pomelo17.server.163.org". Also you can do it at the `master` machine). Because the `web-server` is stateless web server, you can use the `kill`/`Ctrl+c` to stop it. 

#### 4. Description
* In the distributed deployment, the code about "start/stop" the application server can refer to `sshrun` function and related parts in `lordofpomelo/game-server/node_modules/pomelo/lib/master/starter.js`. 

