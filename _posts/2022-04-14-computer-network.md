---
layout: default
---

# Computer Network

## A day in the life of a Web page request

假设某台设备通过有线第一次接入校园网，它想获取一个网页，这是它接下来发生的各个环节。

1. 首先假设有个DHCP服务器就在校园网路由器中，主机发起DHCP请求：
   1. 使用UDP传输协议从端口68发往服务器的端口67
   2. IP数据报以广播形式发送，源地址为0.0.0.0，目标地址为255.255.255.255
   3. IP数据报被转载在以太网帧中，其中源地址为网卡的MAC地址，目标地址为FF:FF:FF:FF:FF:FF
   4. 与主机相连的交换机接收到该帧后，广播到所连的全部设备
   5. 路由器接收到该以太网帧后，解析出IP数据报，因为其为广播地址，所以路由器提取出UDP报文，根据端口号发送到DHCP服务器
2. DHCP为该主机分配一个IP地址，并且向其提供DNS服务器地址、网关地址以及子网掩码，省略Offer和Request报文（相似的工作逻辑），向主机发送ACK报文：
   1. 使用UDP传输协议从端口67发往客户端的端口68
   2. IP数据报以单播形式发送，源地址为路由器地址，目标地址为主机地址
   3. 以太网帧的源地址为DHCP服务器的MAC地址，目标地址为主机网卡的MAC地址
   4. 交换机接收到该帧后，由于其自学习特性，它已经知道目的为主机的MAC地址的帧该往那个端口转发，单播到主机
   5. 主机确认其MAC地址与IP地址，提取并设置分配的IP地址、DNS服务器地址、网关地址以及子网掩码，出该局域网的报文将发送到网关地址，是否在局域网依靠子网掩码确认
3. 用户在浏览器中输入网址，发起请求，但首先需要将网址转换为ip地址：
   1. 本机创建一个DNS socket，生成DNS报文，发向本地DNS服务器，使用UDP协议且目的端口53，IP的源地址为本机IP，目标地址为本地DNS服务器IP
   2. 由于本地DNS服务器不在该局域网下，所以主机需要将DNS报文发送给网关，但是主机不知道网关的MAC地址，所以需要先使用ARP协议
   3. 主机发送ARP查询请求，其链路层为广播地址，IP为网关地址，这样帧会被发送到交换机的所有连接设备，并且只有网关的IP对的上，响应报文中加入网关的MAC地址，主机接收到ARP响应报文，获得了网关的MAC地址，因此可以发送DNS查询报文了
   4. 主机发送DNS查询报文，将目的MAC地址设置为网关，目的IP地址设置为本地DNS服务器，源地址均为本机地址，由交换机单播到网关
   5. 网关接收到该以太网帧，根据转发表将其发送到本地DNS服务器的网关路由器，其中目的MAC地址为该网关路由器的地址，经本地DNS服务器的网关路由器转发，最终DNS查询报文到达
   6. 本地DNS服务器根据自身缓存提取查询结果，hostname到ip的映射，发送DNS响应报文到本机（详细过程与发过来类似）
   7. 本机根据UDP报文中的目的地址、目的端口来分解到对应的UDP socket
4. 本机向网站发起HTTP的GET请求：
   1. 使用TCP传输协议，目标端口80，首先要进行三次握手，来确立TCP连接
   2. 本机创建一个TCP socket，发送TCP SYN报文，目的ip地址为网站的ip，目的MAC地址为网关的MAC，网关路由通过到DNS服务器类似的方式将报文发送到HTTP服务器
   3. HTTP服务器发送TCP SYNACK报文，目的ip地址为主机ip，目的MAC地址为其网关MAC，通过与上面一致的方式发送到本机
   4. 本机将SYNACK报文根据目的地址、目的端口、源地址和源端口来分解到对应的TCP socket
   5. 主机发起HTTP GET请求，通过已经建立起来的TCP连接，发送到HTTP服务器
   6. HTTP服务器发送HTTP响应报文，将网站的HTML文件放入内容中，经TCP连接，传回主机

## Chapter 1 Introduction

### What is the Internet?

- *hosts* or *end systems*: 主机、笔记本等联网设备
- *communication links*: 各种网线（双绞线、同轴电缆、光纤）
- *ISP (Internet Service Provider)*: 网络运营商
- *routers* and *link-layer switches*: 用于转发报文的设备
- *TCP (Transmission Control Protocol)*: 传输控制协议，位于传输层
- *IP (Internet Protocol)*: IP协议，位于网络层
- *RFC (Requests for comments)*: 互联网行业标准的文档
- *socket interfaces*: 套接字接口，所有应用层都需要调用其实现对互联网的访问

### Internet Edge

- *access networks*: 终端设备接入的第一个路由器*edge router*所组成的网络
- 列举一些家用的接入网
  - *DSL (Digital Subscriber Line)*
    - 每一户独立带宽
    - 频分复用实现多用途：4KHz以下是电话，4-50KHz是上传，50K-1MHz是下载
  - *coaxial cable, fiber, HFC (Hybrid Fiber Coax)*: 电缆光纤混合
    - 多户共享带宽
    - 相比DSL带宽大
    - 家中是电缆，汇总到光纤
  - *FTTH (Fiber to the Home)*: 光纤到户
    - 多户共享带宽
    - 相比电缆其带宽更大
- 列举一些用于通信的物理媒介
  - *guided media*: 有线连接
    - *twisted-pair copper wire*: 双绞线，用于DSL
    - *coaxial cable*: 同轴电缆
    - *fiber optics*: 光纤
  - *unguided media*: 无线连接
    - *terrestrial radio channels*: 地面无线信道
    - *satellite radio channels*: 卫星无线信道，分为同步卫星（始终在某个点，目前主要使用）和低轨卫星（位置会变，需要多颗覆盖）

### Internet Core

- *packet switching*: 分组交换，通过路由器等设备将数据包发送到目的地的过程
  - *store-and-forward transmission*: 一般路由器都是采用先接收再转发的工作逻辑
  - *output buffer*: 输出缓存，路由器将数据包发送到链路中不是瞬时的，受限于它的带宽，因而总有一些包需要等待发送，就会被放在缓存中。
  - *packet loss*: 由于缓存长度不可能是无限长的，受限于路由器硬件，所有待发送的包占满缓存后，后面进来的包就会被丢掉。
  - *forwarding table*: 转发表，路由器依靠内部的转发表来决定数据包发送到那个端口
- *circuit switching*: 电路交换，会为用户提供独占带宽，保证通信质量
  - *FDM (Frequency-Division Multiplexing)* or *TDM (Time-Division Multiplexing)*: 通过频分或者时分复用的方式，使得一条物理线路可以被公平地划分为多条子线路。

### Network of Networks

- *access ISP*: 接入网运营商，最低级
- *regional ISP*: 地区运营商，次低级
- *IXP (Internet Exchange Points)*: 供不同顶级运营商交换网，次高级
- *tier-1 ISP*: 顶级运营商，最高级
- *content provider*: 内容分发者，比如谷歌，与顶级运营商同级

### Delay and Throughput

- 列举总体时延的四个组成部分
  - *processing delay*: 处理时延，传输过程中路由器所作的一系列必要操作（比如校验和、转发表查询等）所产生的时间开销
    - 具体大小与每台路由设备有关
    - 在单台设备上可认为是固定的
  - *queuing delay*: 排队时延，在路由器中输出缓存等待转发所产生的时间开销
    - 具体大小取决于线路的拥塞程度
    - *traffic intensity*: 描述线路的拥挤程度，即当前传输量与理论最高传输量的比值，越接近1说明越拥挤
    - 线路越拥塞会导致排队时延指数上涨，甚至丢包
  - *transmission delay*: 传输时延，路由器将数据包发送到链路所产生的时间开销
    - 具体大小取决于线路的带宽和数据包的大小
    - 假设数据包大小为L，传输带宽为R，最终开销为L/R
  - *propagation delay*: 传播时延，数据包在线路中传播所产生的时间开销
    - 以近似光速传播，具体大小取决于线路长度
    - 假设线路长度为s，传播速度为c，最终开销为s/c
- *throughput*: 吞吐量，代表端到端的实际传输速度，以比特级别为单位，包含协议开销和丢包重传，一条链路的吞吐量取决于其中最慢的一段的速度

### Protocol

- *Internet protocol stack*: 互联网5层协议栈
  - 从上到下，依次是应用层、传输层、网络层、数据链路层和物理层
  - 作为下层的本层会将来自上层的完整报文作为载荷*payload*，并添加本层的首部*header*，从而组成本层的完整报文；发送时从上到下是一层层包裹的过程，而接收时从下到上是一层层剥离的过程

## Chapter 2 Application

### Network Application Architectures

- **server-client**架构
  - *server*: 保持在线监听请求的主机
  - *client*: 向服务器发送请求的主机
  - *data center*: 由多台主机构成的虚拟服务器，在网络中处于同一位置但分流了负载
- **P2P (Peer to Peer)**架构
  - *peers*: 在网络中处于同等地位的互相连接的主机，不依赖于某一特定的服务器进行通信
  - 代表软件：*BitTorrent*
  - 特点：自成规模（不需要外加服务器，有更多的*peer*加入就可以提升性能），高度去中心化

### Processes Communicating

- *process*: 进程，执行中的某个程序，实际通讯的单位
  - 在网络中分为*server process*和*client process*，前者负责接收和回复请求，后者负责发送请求，哪怕是在**P2P**架构中某一时刻也可以按照收和发这样划分。
- *socket*: 套接字，是由操作系统提供给应用层的网络接口，即*API (Application Programming Interface)*
  - 套接字连接了应用层和传输层，应用程序可以设定传输层所使用的协议以及报文长度
  - 每个套接字都有对应的端口*port*，端口会作为*IP*地址更细的划分，决定了发到本机的消息该给哪个应用；较小的端口号往往留给了一些常用应用，比如，Web是80，SMTP是25，SSH是22等等

### Transport Services

- *reliable data transfer*
  - *TCP*: 一种提供了可靠的数据传输的传输层协议
  - *UDP*: 一种不可靠的传输层协议，但相对简单，时延低，适合损失容忍应用*loss-tolerant applications*
- *throughput*
  - *bandwidth-sentitive applications*: 带宽要求高的应用，实时语音/视频
  - *elastic applications*: 带宽要求低的应用，传输文件、浏览网页
- *time-sensitive*: 时间敏感性，实时语音/视频

### HTTP (HyperText Transfer Protocol)

- *web pages*: 网页由多个对象*objects*组成，对象有：基础的一个HTML文件，进阶的在HTML中可能存在多处链接，它们指向图片、JS脚本、CCS样式文件等等
- HTTP协议是服务于*server-client*架构的，浏览器是*client*，远程web服务器是*server*
- HTTP使用TCP作为运输层协议
- HTTP是一种无状态协议*stateless protocol*，即进行一次通信后不会有状态量发生改变，服务器不知道客户端是第几次访问服务
- 最早的版本是HTTP/1.0，目前主流的是HTTP/1.1，未来的是HTTP/2

### Persistent Connections

- *persistent connections* or *non-persistent connections*: 这里持久化和非持久化的区别在于有没有每次通信关闭TCP连接，持久化连接可以节省一次建立TCP的时间
- *RTT (Round-Trip Time)*: 一次请求的往返时间，由于三次握手的存在，通常一次HTTP请求需要2个*RTT*
- *HTTP Pipelining*: 一次请求，连续发送多个文件，特别适用于多个小文件多次请求的情况
- HTTP/1.0仅提供不持久性连接，HTTP/1.1提供持久性连接，默认是持久化连接+管道化

### HTTP Request Message

- *request line*: 请求行
  - *method*: HTTP方法
    - GET: 请求页面
    - POST: 带有大量选项的请求，但请求页面通常用GET，GET的选项放置于首部行，而非响应实体
    - HEAD: 用于debug，会返回请求但不返回具体对象
    - PUT: 上传文件到服务器上
    - DELETE: 删除服务器上的某个文件
  - *URL*: 请求的链接
  - *version*: HTTP协议版本
- *header lines*: 首部行
  - *host*: 网页对象所存储的位置
  - *connection*: 是否持久化
  - *user-agent*: 用户代理，即浏览器类型
  - *accept-language*: 网页文本的首选语言，比如中文优先
- *entity body*: 响应实体，GET中通常为空，POST中会放置键值对

### HTTP Response Message

- *status lines*: 状态行
  - *version*: HTTP协议版本
  - *status code*: 状态码
    - 200 请求成功
    - 301 网址永久迁移
    - 400 服务器不理解请求的通用代码
    - 404 请求的文档不存在
    - 505 该HTTP版本不支持
  - *phrase*: 对于状态码含义的描述
- *header lines*: 首部行
  - *connection*: 是否持久化
  - *date*: HTTP请求生成的时间
  - *server*: 服务器类型，对应请求中的*user-agent*
  - *last-modified*: 上次修改时间，与缓存息息相关
  - *content-length*: 在响应实体中的内容长度
  - *content-type*: 在响应实体中的内容类型
- *entity body*: 响应实体，存放被请求的内容

### Cookies

- cookies使HTTP这个无状态协议获得了状态，比如购物车这个功能就离不开cookies
- 在首次访问服务器时，服务器会生成一个随机数作为客户端的cookies，服务器记录该cookies的持有者在网站的一举一动，返回的cookies在响应首部行的*set-cookie*一项中
- 在再次访问服务器时，客户端将在自己的请求首部行中加入*cookie*一项，由于服务器已经记录了该随机数，所以可以识别该持有者，加载记录的状态

### Web Caching

- *web cache* or *proxy server*: 网络缓存，降低时延并减轻网络拥塞
- 网络缓存既是服务器也是客户端，因为用户向其请求，其向目标网络服务器发送请求，它的工作逻辑是：如果本地有用户所请求的对象，且没有过期，则直接返回对象；如果没有，则向目标服务器请求对象，在接收到后缓存一份到本地并发送回用户
- *conditional GET*: 有条件GET，用于解决缓存因时间太长导致其过期（落后于服务器上的最新版本）的问题，向请求头中加入*if-modified-since*字段，其中存放的日期时间是缓存获取时的响应报文中的*last-modified*字段中的日期时间；当服务器未进行更新时，其*last-modified*对应的值不变，所以服务器可以判断是否需要更新；若需要则正常GET，反之返回*304 Not Modified*，响应实体为空，告知缓存有效

### HTTP/2

- *history*: `HTTP/1.0` -> 1997 `HTTP/1.1` -> 2015 `HTTP/2` -> future `HTTP/3` (比如QUIC协议)
- *motivation*: 在`HTTP/1.1`中持久化连接技术的应用节省了创建链接的开销，但遇到了另一个问题：行首阻塞问题*Head of Line (HOL) blocking*
  - 比如在某个网页上有一个大的视频文件和一堆小文件排在后面，由于持久化连接中是按顺序传输的，所以后面的小文件需要等待前面的大文件传完才能进行传输，这就增大了**感知时延**
  - 在`HTTP/1.1`中通常采用并行传输的方式来缓解这个问题，并行传输还有一个好处是可以在TCP拥塞控制中享受到更多的带宽，这是因为拥塞控制是按连接均匀分配带宽的，连接数多了，分到的带宽也会变多
  - `HTTP/2`的目的就是缓解HOL阻塞，使得浏览器不用开太多连接，太多的连接不利于整体的拥塞控制
- *method*: 改变了数据的格式和传输方式
  - *HTTP/2 framing*: 为了避免前面的大文件阻塞后面的小文件，`HTTP/2`将文件拆解成许多个帧（由专门设计的子层实现），轮流发送，这样小文件就会快速的加载出来，降低了感知时延。
  - *response message prioritization*: 客户端发送并发的请求时，可以对请求标记优先级，这样服务器可以先响应高优先级的请求，并且请求间可能存在的依赖关系也可以使用ID来标记
  - *server pushing*: 在服务器知道一些必要请求的情况下，可以在客户端未请求的情况下，提前给出响应，节省因等待客户端的请求报文所造成的时延

### E-Mail Protocol

- *e-mail*: 电子邮件是一种异步通讯方式，接收者不需要与发送者同时在线
- *user agent*: 用户操作的邮箱客户端/网页
- *mail server*: 真正保存邮件，传递邮件的服务器，每个账户的邮件会被分别保存，这也就是邮箱的概念
- *SMTP (Simple Mail Transfer Protocol)*:
  - SMTP是邮箱服务器之间的通讯协议，从用户代理到邮箱服务器的通讯协议可以是SMTP或者HTTP，由于SMTP是一个发送协议，所以，从邮箱服务器到用户代理是不能使用SMTP的，通常使用HTTP或者IMAP(Internet Mail Access Protocol)协议
  - SMTP使用TCP作为传输层协议，采用持久化连接，端口号为25
  - 由于SMTP协议的出现过于早期，其只能发送7位ascii码，所以在发送前需要先进行编码，在接收后需先进行解码
  - 为什么不直接发送到收件人的主机？有两点原因：
    - 邮箱服务器的存在可以更好的找到收件人，不用每台主机收集信息
    - 邮箱服务器可以每半个小时发送一次请求（每台主机定时重发会很麻烦），保证邮件的到达

### DNS (Domain Name System)

- *hostname*: 主机名，为了方便人们记忆，通常会起个主机名（例如www.baidu.com），但网络中用于定位主机的是IP地址；DNS就是负责主机名与IP地址的转换
- *hostname aliasing* and *mail server aliasing*: 由于主机名或者邮箱服务器的正式名称通常很长，所以一般会使用别名来帮助使用者记忆，而且对于大型网站，通常有多个服务器，通过统一别名的方式有助于负载均衡
- *overview*: DNS报文以UDP协议从端口53发送与接收，大量的应用层协议依赖于DNS来映射IP地址，所以DNS服务器一般以分布式的形式部署，降低DNS的时延
- *hierarchy*:
  - *root server*: 根服务器，世界上总共有13台，但是其镜像有1000台以上分布在全球各地
  - *TLD (Top-Level Domain) server*: 顶级域名服务器，如`org`、`edu`、`com`、`jp`等，位于根服务器之下
  - *authoritative server*: 权威服务器，每个组织自己的DNS服务器，保存该组织网址或邮箱的IP地址，位于顶级域名服务器之下
- *local DNS server*: 本地DNS服务器，不属于DNS服务器的架构体系下，类似于一个DNS的用户代理，通常位于离主机特别近的位置，主机与其交互，它再与DNS服务器的三层架构交互
  - 迭代式查询，本地DNS服务器依次向根服务器、顶级域名服务器和权威服务器查询，前两个提供的是下级DNS服务器的位置，最后一个提供的是IP地址
  - 递归式查询，本地DNS服务器向根服务器查询，再有根服务器向顶级域名服务器查询，最后向权威服务器查询，然后结果一步一步回传，最后由根服务器交给本地DNS服务器
- *DNS caching*: DNS缓存的存在降低了查询时延，减轻了网络流量，本地DNS服务器会缓存查询过的hostname到ip地址记录，在下次收到相同的hostname请求时可以直接响应
- *RRs (resource records)*: DNS服务器中存储的记录，格式通常为`Name, Value, Type, TTL`
  - *TTL*: 剩余存活时间，决定了缓存的过期时间
  - *Type*: 不同的类型中的键值对含义不同
    - *Type=A*: 标准的hostname(`Name`)到ip地址(`Value`)映射
    - *Type=NS*: 域名(`Name`)到权威服务器(`Value`)的映射
    - *Type=CNAME*: 别名(`Name`)到主机正式名称(`Value`)的映射
    - *Type=MX*: 别名(`Name`)到邮箱正式名称(`Value`)的映射

### P2P (Peer-to-Peer) File Distribution

- *scalability*: 相比于传统的*client-server*架构，*P2P*的优势在于每个加入的用户在接收文件的同时也在上传文件，用户越来越多的情况下，发送带宽会随着接收带宽增长，反观传统架构中，只有服务器一方在上传，其带宽受限，导致总分发时间随着客户端保持线性增长。
- *BitTorrent*: 一个流行的*P2P*文件传输协议
  - *chunks*: 大文件会被分为256KB的小块，用户间通过交换这些小块来完成一个大文件的分发，由于分成了许多独立的小块，所以用户在一次分发过程中可以随时退出再重新加入
  - *tracker*: 相当于一个服务器，每个用户都要周期性地向*tracker*汇报自己是否还在洪流中

