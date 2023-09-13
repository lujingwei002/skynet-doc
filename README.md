# lualib

## type

### <span id="handle">handle</span>

- type handle number
- 数字类型的handdle,即handle地址

### address

- type address string
- string类型的地址，以`:`开头

### localname

- type localname string
- string类型的服务名, 以`.`开头

### globalname

- type globalname string
- string类型的服务名，全大写，最多16个字符

### ptype

- type ptype integer|string

### dispatch_message_func 

- type dispatch_message_func func(prototype, msg, sz, session, source)

## loader

- 设置`SERVICE_NAME`等于当前服务文件名
- 设置`SERVICE_PATH`
- 执行`preload`




## skynet

### skynet.address(addr [handle](#handle)|[address](#address)): (addr [address](#address))

- 返回handle的字符串地址



### skynet.harbor(addr [handle](#handle)):(harbor_id integer, remote bool)

- 返回handle的harbor_id




### skynet.memlimit(bytes integer)

- 修改服务的最大内存限制

- 修改`service_snlua.c`中snlua结构中的mem_limit

  ```c
  struct snlua {
  	lua_State * L;
  	struct skynet_context * ctx;
  	size_t mem;
  	size_t mem_report;
  	size_t mem_limit;
  	lua_State * activeL;
  	ATOM_INT trap;
  };
  
  static void *
  lalloc(void * ud, void *ptr, size_t osize, size_t nsize) {
  	struct snlua *l = ud;
  	size_t mem = l->mem;
  	l->mem += nsize;
  	if (ptr)
  		l->mem -= osize;
  	if (l->mem_limit != 0 && l->mem > l->mem_limit) {
  		if (ptr == NULL || nsize > osize) {//alloc 或者 realloc
  			l->mem = mem;//恢复刚才的l->mem += nsize;
  			return NULL;
  		}
  	}
  	if (l->mem > l->mem_report) {
  		l->mem_report *= 2;
  		skynet_error(l->ctx, "Memory warning %.2f M", (float)l->mem / (1024 * 1024));
  	}
  	return skynet_lalloc(ptr, osize, nsize);
  }
  
  struct snlua *
  snlua_create(void) {
  	struct snlua * l = skynet_malloc(sizeof(*l));
  	memset(l,0,sizeof(*l));
  	l->mem_report = MEMORY_WARNING_REPORT;
  	l->mem_limit = 0;
  	l->L = lua_newstate(lalloc, l);
  	l->activeL = NULL;
  	ATOM_INIT(&l->trap , 0);
  	return l;
  }
  ```

  



### skynet.term(service handle)

- aa
  - 清除全部watching_session
  - 清除全部unresponse
- 参考skynet.kill
- 参考skynet.exit



### skynet.timeout(ti integer, f function())

- 设置超时ti后，调用f



### skynet.trace_timeout(on bool)

- trace skynet.timeout函数
- skynet.task可以返回超时没有调用和skynet.timeout



### skynet.sleep(ti, token)

- 休眠指定时间
- 设置token后可以用skynet.wakeup提前唤醒



### skynet.wait(token)

- 无限暂停
- 设置token后可以用skynet.wakeup提前唤醒



### skynet.yield()

- 让出cpu



### skynet.killthread(thread)



### skynet.self()



### skynet.localname()

- .服务名转换成服务handle



### skynet.trace(info)

- 输出日志到logger服务, trace一般是在收到消息时开始调用，直到消息处理函数结束，则输出`"end"`
- <Trace $tag> trace $info
  - $tag=:handle-$traceid
  - $traceid是自增的
- <Trace $tag> request
- <Trace $tag> response
- <Trace $tag> call
- <Trace $tag> sleep
- <Trace $tag> error
- <Trace $tag> end



### skynet.tracetag()

- 返回当前的$tracetag
- 调用skynet.trace时会设置$tracetag



### skynet.starttime()

- 返回启动时间



### skynet.time()

- 返回当前时间



### skynet.exit()

- 关闭自身服务，比`skynet.kill(name)`更安全

- 可以用`skynet.monitor`监控退出的服务

- 清理工作

  - 调用 `.launcher`的`REMOVE`

  - 清理`session_coroutine_id`, 通知`session`不为0（即等待回复的服务）`PTYPE_ERROR`

  - 关闭`session_id_coroutine`里的`co`

  - 回复全部`unresponse` `resp(false)`

  - 清理`watching_session` 通知对方`PTYPE_ERROR`

  - 重置回调`c.callback(function() end)`

  - 清理`skynet_context` `c.command("EXIT)`

  - 关闭自身`co` `coroutine_yield "QUIT"`
  
  

### skynet.getenv(key)



### skynet.setenv(key, value)



### skynet.send(addr, typename, ...)



### skynet.rawsend(addr, typename, msg, sz)



### skynet.genid



### skynet.redirect(dest, source, typename, ...)



### skynet.pack



### skynet.packstring



### skynet.unpack



### skynet.tostring



### skynet.trash



### skynet.call(addr, typename, ...)



### skynet.rawcall(addr, typename, msg, sz)



### skynet.tracecall(tag, addr, typename, msg, sz)



### skynet.ret(msg, sz)



### skynet.contect()



### skynet.ignoreret()

- 忽略回复



### skynet.response()



### skynet.retpack(...)



### skynet.wakeup(token)



### skynet.dispatch(typename, func)



### skynet.dispatch_unknown_request(unknown)



### skynet.dispatch_unknown_response(unknown)



### skynet.fork(func, ...)



### skynet.dispatch_message(...)

- prototype的类型

  - 是response
    - 如果co已经"BREAK", 直接忽略
    - 如果co是空，则打印unknown_response警告
    - 唤醒co
  - 没注册的消息类型
    - 如果是PTYPE_TRACE, 记录trace tag
    - session非0，给对方回复一个PTYPE_ERROR
    - 打印unknown_request警告
  - 有注册的消息类型
    - 有处理函数
      - 创建co,并执行处理函数
    - 没处理函数
      - session非0， 给对方回复一个PTYPE_ERROR
      - session为0， 打印unknown_request警告

  

- 执行完所有的fork,直到他们全部结束或者suspend



### skynet.newservice(name, ...)

- 调用`.launcher`启动`snlua ...`服务



### skynet.uniqueservice(global, ...)

- 用`.service`启动服务



### skynet.queryservice(global, ...)

- 到`.service`中查询服务



### skynet.error

- 输出日志到logger服务



### skynet.tracelog



### skynet.traceproto

- 开关proto的trace功能



### skynet.init(f function())

- 注册初始化函数，可以注册多个



### skynet.pcall



### skynet.init_service(start function())

- 仅内部调用，由skynet.start调用



### skynet.start(start_func function())

- 每个服务都需要调用一次
- skynet.init注册的函数调用完后，就会执行start_func



### skynet.endless():integer

- 和skynet.stat('endless')一样

- 返回是否`endless`,并重置`endless`为`false`

  

### skynet.mqlen():integer

- 和skynet.stat("mqlen")一样
- 返回消息队列大小



### skynet.stat(what "mqlen"|"endless"|"cpu"|"time"|"message"): integer

- `"mqlen"`
  - 返回消息队列的大小
- `"endless"`
  - 是否`endless`,并重置`endless`为false, 因为当前的`context`是活动的
- `"cpu"`
  - 如果开启了profile, 则返回cpu cost，单位microsec
- `"time"`
  - 当前回调函数中，已经经过了的时间，单位是microsec
- `"message"`
  - 已处理的消息的数量



### skynet.task(ret)



### skynet.uniqtask()



### synet.term(service)



### skynet.memlimit(bytes)



## <span id="skynet.sprotoparser">skynet.sprotoparser</span>

### type

#### <span id="skynet.sprotoparser.SprotoBin">SprotoBin</span>

### export

#### sparser.dump(str [SprotoBin](#skynet.sprotoparser.SprotoBin))

#### sparser.parse(text string, name string|nil): (bin [SprotoBin](#skynet.sprotoparser.SprotoBin))

- 参数
  - text .sproto文件的文本
  - name 名字, 只是用来日志输出

## skynet.sprotoloader

### export

#### loader.register(filename string, index integer): void

- 解析协议并保存在槽中

- 参数
  - filename 文件名
  - index 槽下标 从0开始

#### loader.load(index integer): (sproto [Sproto](#skynet.sproto.Sproto))

- 从槽中加载sproto

- 参数
  - index 槽下标, 从0开始
- 返回
  - sproto no gc版本的sproto

#### loader.save(bin [SprotoBin](#skynet.sprotoparser.SprotoBin), index integer): void

- 将sproto保存在槽中

- 参数
  - bin [skynet.sprotoparser](#skynet.sprotoparser)解析出来的协议二进制内容
  - index 槽下标 从0开始

## skynet.sproto

### type

#### <span id="skynet.sproto.SprotoType">SprotoType</span>

- struct
  - request
  - response
  - name string
  - tag integer

#### <span id="skynet.sproto.SprotoHost">SprotoHost</span>
##### host:dispatch((data ptr, sz integer)|(data string))
  - 处理消息
    - request
      - 有session, return "REQUEST", name, result, response, ud
      - 无session, return "REQUEST", name, result, nil, ud
    - response
      - 有记录response, return "RESPONSE", session, nil, ud
      - 无记录, return "RESPONSE", session, result, udjj
##### host:attach(sp [Sproto](#skynet.sproto.Sproto)): function(name string, args, session, ud)
  - 返回function(name, args, session, ud)
  - 根据协议`name`编码`args`
  - 设置`session`, `host:dispatch`时回调
#### <span id="skynet.sproto.Sproto">Sproto</span>

##### sproto:host(packagename string|nil): (host [SprotoHost](#skynet.sproto.SprotoHost))

- 创建一个host对象，用于request, response协议

- 参数

##### sproto:exist_type(typename string): bool
  - 判断是否存在结构`typename`
  - 参数
    - typename 结构名


##### sproto:encode(typename string, tbl table): string
  - 将`tbl`按`typename`结构编码
  - 线程不安全，因为内部用了一个`upvalue`做`buffer`
  - 返回`lstring`
##### sproto:decode(typename string, ...): (tbl table, sz integer)
  - 根据`typename`解码
  - `...`
    - `string`
    - `lightuserdata`, `integer`
  - - 
##### sproto:pencode(typename string, tbl table): string
  - 和`sproto.encode`类似，但多了一个`sproto.pack`步骤
##### sproto:pdecode(typename string, ...): (tbl table, sz integer)
  - 和`sproto.decode`类型，但多了一个`sproto.unpack`步骤
  - 返回
    - sz 使用的字节长度
##### sproto.queryproto(protoname string): (type [SprotoType](#skynet.sproto.SprotoType))
  - 查询协议
##### sproto:exist_proto(protoname string):bool
  - 检查是否存在协议`pname`	
##### sproto:request_encode(protoname string, tbl table): (data string, tag integer)
  - 根据`protoname`的`request`编码`tbl`
##### sproto:response_encode(protoname string, tbl table): string
  - 根据`protoname`的`response+`编码`tbl`
##### sproto:request_decode(protoname string, ...): (tbl table, sz integer, name string)|(nil, name string)
##### sproto:response_decode(protoname string, ...): (tbl table, sz integer)|(nil)
##### sproto.pack
- 压缩打包
##### sproto.unpack
- 解压缩
##### sproto:default(typename string, type nil|"REQUEST"|"RESPONSE")
  - 参数
    - `type`
      - `nil` 结构
      - `REQUEST`
      - `RESPONSE`



### export
#### sproto.new(bin):(sproto [Sproto](#skynet.sproto.Sproto))
  - 创建协议
  - 参数
    - `bin` 用`sprotoparser.parse`后的
#### sproto.sharenew(cobj):(sproto [Sproto](#skynet.sproto.Sproto))
  - 和`sproto.new`类似，但没有`gc`,`gc`就是在lua回收时自动将`c`里的内存结构释放
#### sproto.parse(text string):(sproto [Sproto](#skynet.sproto.Sproto))
  - 从`.sproto`文件创建
  - 参数
    - text .sproto文件内容



## skynet.manager

### export

#### skynet.launch(...):nil|[handle](#handle)

- 调用c.command("LAUNCH", ...) 启动服务， 可以启动c服务



#### skynet.kill(name [handle](#handle)|[address](#address)): void

- 杀死服务,
  - 调用`.launcher`的`REMOVE`
  - c.command("KILL", name)
  
- 参考skynet.exit
  
  

#### skynet.abort()

- 中止整个程序
  - c.command("ABORT")
    - skynet_handle_retireall()
    
    

#### skynet.register(name [globalname](#globalname)|[localname](#localname))

- 绑定服务名，可以是本地服务名或者全局服务名
- 参考skynet.name



#### skynet.name(name [globalname](#globalname)|[localname](#localname), handle [handle](#handle))

- 将handle命名为name，可以是本地服务名或者全局服务名
- 参考skynet.register



#### skynet.forward_type(map {[ptype]=ptype]}, start_func function())

- 类似skynet.start, 但会转换ptype,再调用dispatch_message



#### skynet.filter(f dispatch_message_func, start_func)

- 类似skynet.start，但对dispatch_message进行wrap



#### skynet.monitor(service string, query bool)

- 将一个服务设置成monitor服务
  - 设置`G_NODE.monitor_exit`
  - 服务退出时，monitor会收到`PTYPE_CLIENT`消息

- 如果query为true,就只查询，不启动服务

- 监控服务
  - c.command("MONITOR",  address)

## skynet.service

- 使用`service_provider`启动服务

### export

#### service.new(name, mainfunc, ...)

#### service.close(name)

#### service.query(name)

## skynet.habor

### exprt
#### habor.globalname(name globalname, handle handle): void
  - 注册全局服务名字
#### habor.queryname(name globalname): (h handle)
  - 查询全局服务名
  - 如果服务名不存在，会一直阻塞
  - 返回`handle`
#### habor.link(id habor_id)
  - 监控从节点退出状态。
#### habor.connect(id habor_id)
  - 监控从节点连接状态。
#### habor.linkmaster()
  - 监控和`.matster`的连接状态。

## skynet.mqueue

### type

#### message

- type message {session integer, addr source, ...}

### proto

#### PTYPE_QUEUE|"queue"

### export

#### mqueue.register(f function(msg message))

- 注册queue消息

- ```lua
  local mqueue = require "skynet.queue"
  mqueue.register(f)
  ```

#### mqueue.call(addr handle|localname, ...)

- 给服务发送`"queue"`消息

#### mqueue.send(addr handle|localname, ...)

- 给服务发送`"queue"`消息

#### mqueue.size()

- 返回消息队列的大小



## skynet.queue

- 包装消息处理函数，使得消息处理函数能按顺序执行。函数返回才算完成，挂起不算。

### type

#### queue

##### queue(f): f 

- wrap函数,使函数按顺序执行

### export

#### skynet.queue(): queue

- 创建队列



## skynet.require

- 由`loader.lua`require
- 修改`require`，使得每个模块第一次加载时可以执行`skynet.init`函数

## <span id="skynet.sharemap">skynet.sharemap</span>

---
### example

```lua
local skynet = require "skynet"
local sharemap = require "skynet.sharemap"

local mode = ...

if mode == "slave" then
--slave

local function dump(reader)
	reader:update()
	print("x=", reader.x)
	print("y=", reader.y)
	print("s=", reader.s)
end

skynet.start(function()
	local reader
	skynet.dispatch("lua", function(_,_,cmd,...)
		if cmd == "init" then
			reader = sharemap.reader(...)
		else
			assert(cmd == "ping")
			dump(reader)
		end
		skynet.ret()
	end)
end)

else
-- master
skynet.start(function()
	-- register share type schema
	sharemap.register("./test/sharemap.sp")
	local slave = skynet.newservice(SERVICE_NAME, "slave")
	local writer = sharemap.writer("foobar", { x=0,y=0,s="hello" })
	skynet.call(slave, "lua", "init", "foobar", writer:copy())
	writer.x = 1
	writer:commit()
	skynet.call(slave, "lua", "ping")
	writer.y = 2
	writer:commit()
	skynet.call(slave, "lua", "ping")
	writer.s = "world"
	writer:commit()
	skynet.call(slave, "lua", "ping")
end)

end
```



### use case

- 协程之间共享sproto

### type

#### <span id="skynet.sharemap.stm">stm</span>
- 引用着sproto编码出来的二进制内容，使之可以共享

  
#### sharemap_writer
##### sharemap_writer:copy():[stm](#skynet.sharemap.stm)
##### sharemap_writer:commit():void
#### sharemap_reader
##### sharemap_reader:update()
- 从stm中更新最新的内容，然后解码
### export
#### sharemap.register(protofile string):void
- 解析.proto文件，并保存在[sprotoloader](#skynet.sprotoloader)`的槽0中

#### sharemap.reader(typename, stmcpy [stm](#skynet.sharemap.stm)):sharemap_reader
- 参数
  - stmcpy 来自sharemap_writer:copy()
#### sharemap.writer(typename string, obj table):sharemap_writer
- 参数
  - typename sproto里的结构名
  - obj 需要编码的table
## skynet.datacenter

- 将数据保存在服务`DATACENTER`中
### export
#### datacenter.get(...)
#### datacenter.set(...)
#### datacenter.wait(...)
  - 如果数据不存在，会一直等待



## skynet.snax

### service

#### accept

#### response

#### system.init

#### system.exit

#### system.hotfix

#### system.profile

### proxy

#### post

- 对应accept

#### req

- 对应response

### export

#### snax.bind(handle, type): proxy

#### snax.newservice(name, ...): proxy

#### snax.uniqueservice(name, ...): proxy

#### snax.globalservice(name, ...): proxy

#### snax.queryservice(name): proxy

- 查询服务并返回proxy

#### snax.queryglobal(name): proxy

- 查询全局服务并返回proxy

#### snax.kill(proxy, ...)

- 关闭服务

#### snax.self()

- 返回当前服务的proxy

#### snax.exit(...)

- 退出自身服务

#### snax.hotfix(proxy, source, ...)

- 热修复

## skynet.debug

### export

#### skynet.init(skynet, {dispatch dispatch_message, suspend, resume})

#### skynet.reg_debugcmd(name string, fn)

### proto "debug"

#### MEM()

#### GC()

#### STAT()

#### KILLTASK(threadname)

#### TASK(session)

#### UNIQTASK()

#### INFO(...)

#### EXIT()

#### RUN(source, filename, ...)

#### TERM(service)

#### REMOTEDEBUG(...)

#### SUPPORT(typename)

- 是否能处理`typename`消息类型

#### PING()

- skynet.ret()

#### LINK()

- 用`call`调用LINK，如果出错就会返回错误

#### TRACELOG(proto, flag)

- 调用skynet.traceproto

## skynet.socket

### proto

#### `"socket"` `PTYPE_SOCKET`

### export

#### socket.open(addr string, port integer): (nil, err string)|id integer

- 连接，阻塞直到成功或者失败

#### socket.bind(os_fd integer): (nil, err string)|id integer

- 绑定`os_fd`

- ```lua
  socket.bind(0)
  # 绑定标准输入
  ```

#### socket.stdin(): (nil, err string)|id integer

- 绑定标准输入

#### socket.start(id integer, func)

#### socket.parse(id integer)

#### socket.shutdown(id integer)

- 关闭id

#### socket.close_fd(id)

- 关闭链接，但并没有释放id

#### socket.close(id)

- 关闭连接，并释放id
- 等级数据read完再释放id

#### socket.onclose(id, callback)

  - 设置关闭回闭
  - id不存在，也会设置成功
#### socket.warning(id, callback)

  - 处理`SKYNET_SOCKET_TYPE_WARNING`
  - id必须存在
#### socket.resolve(host): error string|table

  - 如果返回`table`,是一个地址数组
#### socket.netstat()   : table
#### socket.udp(callback, host, port)
#### socket.udp_address(addr string): (address string, port integer)
#### socket.udp_connect(id integer, addr string, port integer, callback function)
  - 连接udp
#### socket.sendto(id integer, address string, buffer)：(succ bool)
  - 发送udp数据
#### socket.limit(id integer, limit integer)
  - 设置buffer上限, 如果接收数据时超过上限，将会报错,并导致关闭
#### socket.abandon(id integer): void
  - 丢弃id,不处理消息了，交给其他处理
#### socket.listen(host string, port integer, backlog integer): (id integer, addr string, port integer)
  - 侦听, 直到收到`SOCKET_OPEN`
#### socket.disconnected(id integer): bool | nil
  - socket是否已经断开
  - `listen`id也有`connected`
#### socket.invalid(id integer): bool
#### socket.header(lstring string): (sz int)
  - lstring大小必须等于4
  - 将lstring换成长度
#### socket.lwrite(id integer, buffer): (succ bool)
#### socket.write(id integer, buffer): (succ bool)
#### socket.block(id integer)
#### socket.readline(id integer, sep string): (false)|()



### skynet.socketchannel

#### socketchannel.channel(desc):channel

- 创建channel
- desc.__backup 地址数组 
  - [{host string, port integer}]
  - [host string] port使用desc.__port
- desc.host 主机地址
- desc.port 主机端口
- desc.nodelay: bool 设置nodelay
- desc.overload: function(bool) 设置overload回调
- desc.auth: function(channel):(ok bool, message) 认证函数
  - auth可以改变host
  - auth可以修改backup

#### channel:connect(once bool)

- 如果`once`为true, 连接失败的话，1秒重试一次，总共尝试10次

#### channel:changebackup(backup): void 

#### channel:changehost(host string, port integer): void

- 修改地址，会触发重连

#### channel:close(): void

- 关闭channel

#### channel:request(request, response, padding)

#### channel:response(response)



# service-src

## databuffer

### usercase

### type

#### messagepool

#### messagepool_list

#### message

#### databuffer

```c
#define MESSAGEPOOL 1023

struct message {
	char * buffer;
	int size;
	struct message * next;
};

struct databuffer {
	int header;
	int offset;
	int size;
	struct message * head;
	struct message * tail;
};

struct messagepool_list {
	struct messagepool_list *next;
	struct message pool[MESSAGEPOOL];
};

struct messagepool {
	struct messagepool_list * pool;
	struct message * freelist;
};
```



## hashid

### usecase
  - 将用户id转为[0-max)之间的下标

### type

#### hashid

```c
struct hashid_node {
	int id;
	struct hashid_node *next;
};

struct hashid {
	int hashmod;
	int cap;
	int count;
	struct hashid_node *id;
	struct hashid_node **hash;
};
```

- id是空闲节点列表，init时就根据max值申请好了。

### export

#### int hashid_insert(hashid *hi, int id)

#### int hashid_full(hashid *hi)

#### int hashid_remove(hashid *hi, int id)

#### int hashid_lookup(hashid *hi, int id)

#### void hashid_clear(hashid *hi)

#### void hashid_init(hashid *hi, int max)

## snlua

## logger

- 用于输出日志
- 启动参数
  - 参数1 日志文件路径， 如果没指定则输出到stdout

### proto PTYPE_SYSTEM

- freopen日志文件

### proto PTYPE_TEXT

- 输出日志



## <span id="harbor">habor</span>

- `slave`指定`snlua cslave`
- `habor_id`是自身的节点id
- 保存每个节点和每个节点开启的全局服务

### proto PTYPE_SYSTEM

- 发送消息给其他节点

- ```c
  struct remote_name {
  	char name[GLOBALNAME_LENGTH];
  	uint32_t handle;
  };
  
  struct remote_message {
  	struct remote_name destination;
  	const void * message;
  	size_t sz;
  	int type;
  };
  
  ```

### proto PTYPE_HARBOR

- 处理`snlua cslave`的消息

#### 'N 服务名'

- 注册服务名

#### 'S fd id'

- connect slave，并接管socket
- 置为STATUS_HANDSHAKE状态
- 给对方发送handshake

#### 'A fd id'

- accept slave，并接管socket
- 并分发之前缓存下来的消息
- 置为STATUS_HEADER状态
- 给对方发送handshake
- 

### proto PTYPE_SOCKET

- 接收到的消息转发给本地的服务

# service

## .bootstrap

- 首先启动的服务
- 启动[`.launcher`](#.launcher)
- 单节点模式(harbor_id为 0)
  - 设置standalone=true
  - 启动[`cdummy`](#.cdummy)
- 多节点模式(harbor_id非0)
  - 启动[`.cmaster`](#.cmaster)
  - 启动[`.cslave`](#.cslave)
- 章节点模式或者master节点
  - 启动[`DATACENTER`](.DATACENTER)
- 启动 [`.service`](#.service)
- 启动`main`

## <span id=".cdummy">.cdummy</span>

## <span id=".launcher">.launcher</span>

- 启动服务, 可以是`snlua`服务，也可以是c服务

### proto "text"

#### LAUNCHOK

- `skynet.init_service`启动完后，会发送`LAUNCHOK`

#### ERROR



### proto "lua"

#### LAUNCH(_, service string, ...): bool

- 启动服务


#### REMOVE（_, handle [handle](#handle), kill bool): void
  - 删除服务
  - no response


#### QUERY(_, request_session): handle|nil


#### GC (addr, ti)
#### LIST
  - 返回 {$address: "启动参数"}
#### KILL(_, handle)

  - skynet.kill(handle)

  - response

    - {[$address]="启动参数"}
#### STAT(source [handle](#handle), ti integer): table

- 给所有服务发送 "debug STAT"

  - response
    - {[$address]="ERROR"|"debug STAT 返回值"}
#### MEM(source [handle](#handle), ti integer): table

- 给所有服务发送"debug MEM"

  - response
    - {[$address]="ERROR"|"内存大小 (启动参数)"}

## <span id=".service">.service </span>

## <span id="SERVICE">SERVICE</span>

 - 支持`snax`
   - 服务名是"snaxd.服务文件名"
- 服务名
  - `snaxd.服务文件名`
  - `服务文件名`
- command
  - LAUNCH (service_name, subname)
    - 在本地启动服务
    - 会阻塞直到成功或者失败
  - QUERY (service_name, subname)
    - 查询服务
    - 会阻塞直到成功或者失败
  - LIST()
    - 从节点只列出本地的服务
    - 主节点可以列出所有服务，包括从节点上的服务
  - a



## service_provider

- 由`service/service_provider.lua` `service/service_cell.lua` `lualib/service.lua`三个源文件组成

## <span id=".cmaster">.cmaster</span>

- 保存着全局名字`global_name`
- 侦听地址在`env.standalone`
  - 听`cslave`连接, 接收`"H slave_id slave_addr"`完成握手
  - 发送`"C slave_id slave_addr"`给其他节点
  - 发送`"W n"`给当前连接的节点当前有n个节点准备连你

### master=>save

#### "W n"

- 等待n个slave节点连接上来，然后关闭侦听

#### "C slave_id slave_address"

- 连接从节点

#### "N globalname address"

- 新服务名

#### "D slave_id"

- 和slave节点断开连接后，给其他所有节点发送`"D slave_id"`
- 其他slave节点收到消息后，会断开和这个slave节点的连接

### slave=>master

#### "H slave_id slave_addr"

- slave节点连接上来时要发送`"H slave_id slave_addr"`完成握手

#### "R globalname address"

- 注册全局服务名
- 给所有节点回复`"N globalname address"`

#### "Q globalname"

- 查询服务名
- 返回`"N globalname address"`



## <span id=".cslave">.cslave</span>

- 启动[`habor`](#harbor)服务
- 连接`master` `master`配置地址, `address`配置本地地址
- 启动
  - 连接`master`,发送`H`，得到返回`W`,由此得到一共有多少个`slave`数量,除了自己
- 处理`master`发来的消息
- 收到新`slave`节点通知，然后连接，连接上去后，连接就交给[habor](#habor)处理
- `slave`之间相互之间相互连接，且权有一条链接。
- `slave`断开之后不会重连。

### proto "text"

#### "D slave_id"

- [habor](#habor)检查到slave断开，给[.cslave](#.cslave)发送断开消息

#### "Q globalname"

- [habor](#habor)查询服务地址，如果查询不到，[.cslave](#.cslave)就会到master查询

### proto "lua"

#### REGISTER(master_fd integer, name [globalname](#globalname), handle [handle](#handle)): void

- 绑定handle到globalname

#### LINK(master_fd integer, id integer)

- 监控和从节点的连接，直到断开

#### LINKMASTER()

- 监控和master的连接，直到断开

#### CONNECT(master_fd integer, id integer)

- 监控节点，直到节点连接上来

#### QUERYNAME(master_fd integer, name string):(h [handle](#handle))

- 查询服务名
- 如果服务名不存在，会到master查询
- 一直阻塞，直到master返回或者REGISTER注册了这个服务名。

## DATACENTER

- command
  - WAIT( ...)
  - QUERY(key, ...)
  - UPDATE(...)
- afa

## .multicastd

- 节点地址保存在DATACENTER `datacenter.get("multicast", id)`
- channel规则是0x7fffff00 后面8位保留，用来保存节点Id
  - 如果要订阅其他节点的channel,就自己组合相应的channel就可以了，或者通知其他方法去获得。
- COMMAND
  - USUB(source, c)
  
  - USUBR(source, c)
  
  - SUB(source, c)
  
  - SUBR(source, c)
  
  - PUB(source, c, pack, size)
  
  - DEL(source, c)
  
  - DELR(source, c)
  
  - NEW()
- interface
  - 源文件`skynet/multicast.lua`
  - new(conf)
    - conf.channel
      - 名称
      - 如果为空就生成一个
      - 如果要订阅其他节点的消息，就指定channel
    - conf.dispatch（chan, source, ...)
      - 订阅消息处理回数
  - :delete()
  - :publish(...)
  - :subscribe()
  - :unsubscribe()
  - 

## console

- 启动后读从标准输入读取命令
  - snax service_name ...
    - 启动snax服务
  - service_name ...
    - 启动snlua服务

# lualib

## sharedata

- sharedata.lua

  - 每份数据都新建一个lua_State来保存

  - 数据query出来后，会调用sd.box(obj)包装一下，返回这个box给用户，然后fork一个monitor协程监控obj是否变化或者被删除，然后更新box。

- afaf



# lualib-src

## <span id="lualib-src.lua-netpack">lua-netpack</span>

- 分包功能，包的格式如下

- 一个服务有一个独立的`queue`,`queue`中包含全部有效的`socket`的消息列队

  ```mermaid
  classDiagram
  class pack
  pack : int16 header 
  pack : binary content 
  ```

  

- afaf

---

title: aaa

---

```mermaid
classDiagram
class netpack
class uncomplete
class queue

```

### type

#### type

- type type "data"|"more"|"error"|"open"|"close"|"warning"|"init"

### lua

#### pack(data string): (msg ptr, sz integer)

- 拷贝msg
  - sz=len(data)+2
  - msg前面2个字节是len(data),后面是data

#### pop(q [queue](.lualib-src.lua-netpack.queue)): (fd integer, msg ptr, sz integer)

#### clear(q [queue](.lualib-src.lua-netpack.queue)):void

#### tostring(msg ptr, sz integer):(data string)

#### filter(q [queue](.lualib-src.lua-netpack.queue)|nil, msg ptr, sz integer): (q [queue](.lualib-src.lua-netpack.queue), type [type](.lualib-src.lua-netpack.type), fd integer, (msg string)|(msg ptr, sz integer))

- 转换socket的消息

  - `SKYNET_SOCKET_TYPE_WARNING`转为`"warning"`
  - `SKYNET_SOCKET_TYPE_ERROR`转为`"error"`
  - `SKYNET_SOCKET_TYPE_ACCEPT`转为`"open"`
  - `SKYNET_SOCKET_TYPE_CLOSE`转为`"close"`
  - `SKYNET_SOCKET_TYPE_CONNECT`转为`"init"`
  - `SKYNET_SOCKET_TYPE_DATA`转为`"data"`

- 参数

  - msg 是skynet_socket_message

    ```c
    struct skynet_socket_message {
        int type;
        int id;
        int ud;
        char * buffer;
    }
    ```

    

## lua-stm

- 被[skynet.sharemap](#skynet.sharemap)用到

### example

```lua
writer = stm.new(data)
reader = stm.copy(writer)
writer(newdata)
# 更新数据
reader()
```

### type

#### <span id="lualib-src.lua-stm.boxstm">boxstm</span>

##### boxstm((data ptr, sz integer)|(data string)): void

- 更新数据

##### boxstm:__gc()

- 释放[stm_object](#lualib-src.lua-stm.stm_object)

#### <span id="lualib-src.lua-stm.stm_object">stm_object</span>

#### boxreader

##### boxreader(function(data ptr, sz integer, ud), ud)

- 检测[stm_object](#lualib-src.lua-stm.stm_object)有没有改变，如果有改变则更新
- 回调中会返回最新的数据

##### boxreader:__gc()

- 释放[stm_object](#lualib-src.lua-stm.stm_object)

### lua

#### new((data ptr, sz integer)|(data string)): [boxstm](.lualib-src.lua-stm.boxstm)

- 新建一个writer, 可以更新里面的数据

#### newcopy(obj [stm_object](#.lualib-src.lua-stm.stm_object)): [boxreader](.lualib-src.lua-stm.boxreader)

- 将obj交给reader,不增加引用

#### copy(box [boxstm](#lualib-src.lua-stm.boxstm)]): [stm_object](#lualib-src.lua-stm.stm_object)

- 解开box, 返回里面的object, 增加对object的引用

