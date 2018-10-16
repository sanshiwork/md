Anchor 框架
======
一个基于SpringBoot和Netty的网络框架。

目录
--------
1. [开始使用]()
2. [网络架构]()
3. [工程目录]()
4. [网络协议]()
5. [异步任务]()
6. [线程模型]()
7. [命令行构建]()
8. [关于LoginService]()
9. [注意事项]()

开始使用
--------
* 为IDE添加插件lombok

* 使用SpringBoot构建你的工程，根据业务情况，创建Frontend工程或者Backend工程

* 在Maven工程添加依赖:
```xml
<dependency>
    <groupId>com.changyou.fusion</groupId>
    <artifactId>anchor-spring-boot-starter</artifactId>
    <version>1.0</version>
</dependency>
```

* 在application.properties添加关配置信息，样例配置如下:
```properties
# 本机节点ID
anchor.network.id=S1
# 全局节点信息
anchor.network.nodes=\
  S1:127.0.0.1:50000:true,\
  S2:127.0.0.1:50001:true,\
  G1:127.0.0.1:60000:false
# Backend和Frontend间连接监控间隔(单位毫秒)
anchor.bootstrap.socket-connect-tick=5000
# 游戏Tick逻辑间隔(单位毫秒)
anchor.bootstrap.main-logic-tick=10
# 每个tick处理线程间消息上限
anchor.tick.max-message=500
# 每个tick处理Backend和Frontend间输入消息包上限
anchor.tick.max-server-input-packet=100
# 每个tick处理Backend和Frontend间输出消息包上限
anchor.tick.max-server-output-packet=100
# 每个tick处理客户端输入消息包上限
anchor.tick.max-client-input-packet=100
# 每个tick处理客户端输出消息包上限
anchor.tick.max-client-output-packet=100
# 异步任务最大并发数
anchor.event.max-thread=16
# WebSocket路径
anchor.ws.path=/ws
# WebSocket超时时间(单位秒)
anchor.ws.timeout=300
# SocketClient超时时间(单位秒)
anchor.socket.client.timeout=-1
```

* 根据业务情况增加消息包处理逻辑，并选择性重写业务Bean，如LoginService<T>,Client<T>,Server等。

网络架构
--------
1. 手机、浏览器终端使用WebSocket协议连接Frontend。
2. Frontend与所有Backend使用TCP协议连接。
3. Frontend和Backend使用本框架提供的异步任务服务调用三方服务，如Http接口，数据库，缓存等。
![](./assets/img/ANCHOR_NET_V1.png)

工程目录
--------
1. .doc:文档目录
2. .idea:IDE自动生成的目录，不需要关心，不要上传至SVN
3. .mvn:Maven Wrapper定义文件，打包时候提供自动下载Maven的功能
4. anchor:框架核心，提供功能API接口，Bean，注解，线程模型，网络模型，异步任务处理等等。
5. anchor-spring-boot-starter:提供了框架接口的默认实现和自动配置，可根据需求自行重写这些实现。
6. game-common:样例工程。包含消息包，表格工具等通用类的定义。
7. game-frontend:样例工程。包含Frontend的简单实现。
8. game-backend:样例工程。包含Backend的简单实现。

网络协议
--------
WebSocket&TCP消息格式如下:

| 消息包ID   | 消息包长度 |  消息包内容(Protobuf序列化)              |
| :------:   | :--------: | :----------:             |
| 2byte      | 4byte      | N byte                   |
| 1000       | 12         | 01A19B3D939F01A19B3D939F |

消息包处理逻辑定义流程:
1. 定义消息包处理模块（类），并使用@PacketModule注解标记
2. 定义消息包处理方法，使用@PacketHandler(消息包ID)标记
3. 方法的第一个参数定义为Server或者Client(根据消息包来源来判断)
4. 方法的第二个参数定义为消息包反序列化后的Protobuf所定义的对象
```java
@PacketModule
@Slf4j
public class ScoreModule {

    @PacketHandler(PacketID.FB_SCORE_REQ)
    public void handleClientRegister(Server server, Protocol.FB_SCORE_REQ packet) {
        // JUST DO IT.
    }
}
```
异步任务
--------
对于耗时操作，如数据库操作，Cache读写，第三方HTTP接口调用等，请务必使用框架提供的异步服务来进行处理，避免造成游戏逻辑阻塞。

异步任务处理使用流程:
1. 注入EventService对象
2. 调用send方法，使用匿名类进行Event参数定义（推荐匿名类）。
3. 在execute方法内定义异步逻辑（耗时操作），并且返回执行结果
4. 在callback方法定义同步逻辑，处理异步任务的结果
```java
Player copy = new Player();
BeanUtils.copyProperties(this.player, copy);
eventService.send(new Event<Player>() {
    @Override
    protected Player execute() {
        mongoTemplate.save(copy);
        return copy;
    }

    @Override
    public void callback(Player result) {
        log.info("Player Save Success -> {}", player.toString());
        if (!online) {
            // 离线情况下，从对象队列中移除
            Client client = tickService.unRegisterClient(result.getOpenid());
            log.info("Client Remove From Queue -> {}", client);
        }
    }
});
```
线程模型
--------
框架提供了2个线程池:
1. Frontend中用于维护与Backend的TCP连接而定义的Socket监控线程池（断线重连）
2. Tick主逻辑线程池。
![](./assets/img/TICK_SERVICE_V1.png)

命令行构建
--------
windows
```cmd
mvnw.cmd clean package
```
or linux:
```bash
mvnw clean package
```
关于LoginService
--------
登录逻辑的Bean，请务必根据业务覆盖注入，并完成下面3个方法的实现
1. auth:参数为登录消息包，请完成认证逻辑后，返回唯一ID，认证失败时请返回null。
2. load:参数为auth方法的返回值，请完成数据加载逻辑（Database检索），返回对象为玩家数据。
3. packet:请返回用于登录的消息包ID。收到该消息包时，会作为参数传递给auth方法。

```java
@Component
@Scope("prototype")
@Slf4j
public class FrontendLoginService implements LoginService<Player> {

    private final PlayerRepository playerRepository;

    @Autowired
    public FrontendLoginService(PlayerRepository playerRepository) {
        this.playerRepository = playerRepository;
    }

    @Override
    public String auth(Packet packet) {
        try {
            Protocol.CF_LOGIN_REQ request = Protocol.CF_LOGIN_REQ.parseFrom(packet.getBody());
            return request.getOpenid();
        } catch (InvalidProtocolBufferException e) {
            log.error(e.getMessage(), e);
        }
        return null;
    }

    @Override
    public Player load(String id) {
        Player player = playerRepository.findUserByOpenid(id);
        if (player == null) {
            // 新玩家
            player = new Player();
            player.setOpenid(id);
            player.setNickName(UUID.randomUUID().toString().replaceAll("-", "").substring(0, 8));
            player.setAvatarUrl("https://open.weixin.qq.com/zh_CN/htmledition/res/assets/res-design-download/icon64_appwx_logo.png");
            playerRepository.insert(player);
        }
        return player;
    }

    @Override
    public short packet() {
        return PacketID.CF_LOGIN_REQ;
    }
}
```

注意事项
--------
1. 项目完全使用的起步依赖和自动注入，如果需要对anchor和anchor-spring-boot-starter逻辑进行修改，请在业务工程里覆盖注入内容，不要直接修改框架源码。
2. 对于异步任务的抽象，解放了业务对于数据库，缓存等组件的依赖，可以根据需求选择MongoDB,Mysql,Redis,Memcache等。
