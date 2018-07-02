	DHT(Distributed Hash Table) 关心的是找到节点的ip/port,如果可能的话,用UDP打洞技术对它们建立直接的路由.DHT仅在UDP上可以运行,所以只有在UDP工作的时候它才能被使用.

	在Tox DHT中的每一个节点的都有一个被称作临时DHT公钥的地址.这个地址是临时的,并且每次tox实例关闭或者重新启动的时候都会被消除.同盟的DHT公钥用onion module可以被找到,DHT用来找到它们并且通过UDP协议直接连接它们.
	
	DHT在Tox网络中是使用UDP协议进行自我分配节点.自我分配发生在用DHT贯穿每一个节点连接x个节点的时候,toxcore 实现了32个(32 in the toxcore implementation),最靠近它们自己的公共密钥.通过距离方法来定义最靠近的. 2个节点的距离可以被定义为XOR(异或),在俩个32个字节big endian格式的DHT公钥中 .[题外话:big endian 低地址存放最高有效字节 MSB,little endian 低地址存放最低有效字节 LSB],距离越小,也就是越接近的成员.

	一个DHT公钥为1的节点比起它接近公钥为5的节点可能更接近一个公钥为0的节点,因为 1 XOR 0 = 1,1 XOR 5 = 4,1是比较小的,也就意味着与其说1接近于5,不如说1更接近于0.

	如果在网络中的每一个节点都知道最接近它DHT公钥的节点的DHT公钥,然后找到一个公钥特定为X的节点,仅仅需要循环的询问DHT的节点,谁的公钥最接近于X.如果那个节点在线,最终节点会找到在DHT中最接近的那个节点,

#### Packed node format:

	uint8 ip_type(2 代表IPv4,10代表IPv6,130 代表TCP IPv4,138 代表TCP IPv6.首字节代表协议 (0 是 UDP,1 是 TCP),3 bits 代表 nothing,4 bits 代表 address family))][ip (in network byte order),如果是ipv4,长度为4 bytes,如果是ipv6长度为16 字节][port (in network byte order),length=2 bytes][char array (node_id),length=32 bytes]

	包装节点格式(packed node format )是一种小且易于解析格式的存储节点信息的一个方法.用来存储不止一种节点,简单的在前一个节点上追加另一个：[packed node 1][packed node 2]

	包装节点格式在Tox中被用在了很多地方.ip_type 2到10数字 被用在标示ipv4 或ipv6 UDP节点.130被用来标识ipv4 TCP 转发,138被用来标识ipv6 TCP转发. 使用这些数字的原因是因为在我的linux机器上ipv4,ipv6是2到10这些数字(the AF_INET and AF_INET6 defines). TCP的代表数字仅是UPD代表数字+128. 在ipv4地址上,ip是4字节(ip_type 在2-130),在ipv6 地址上,ip是16字节(ip_type 是 10-138 ).这也被32字节的节点公钥遵循.

	当用包装节点格式发送节点的时候,只有UDP ip_types(ip_type 2,ip_type 10) 被用在DHT 模块. 这是因为TCP ip_types 被用来发送TCP转发信息,DHT只是UDP.

#### Ping(request and response):

	字节值00表示请求,01表示响应,发送者的DHT公钥是32字节,随机24字节的随机数.用随机数,发送者的DHT私钥,接收者的DHT公钥来进行加密.[1字节 0表示请求,1表示发送][8字节 ping_id]ping_id是8字节数字,响应必须包含相同的数字作为请求.

	主要的DHT包装类型是ping请求和响应,用来检查另一个节点是否活着,并得到节点包,最多用来查询4个他们知道最接近请求节点的另一个DHT节点.

	ping请求的首字节是0. 这也被发送者的DHT公钥和随机数遵循(This is then followed by the DHT public key of the sender and a nonce).加密部分包含一个值为0的字节其次是一个8字节的ping ip,在响应中将会被发送回来.原因是加密部分的1字节值在请求和响应中使用了相同的密钥,由于加密技术怎么工作可以通过建立一个不对请求解密的响应来防止可能的攻击.ping id 用来确保稍后响应接收到的是这个ping的响应,不是前一个ping的转发响应,将会允许攻击者让ping的发送者相信他们ping的节点还是上涨的. ping_id还用于节点不能只是向节点发送ping响应包,为了让DHT模块实现重置它的超时.它用来确保节点在发送响应之前实际上接收到数据请求包.

	ping响应的首字节是1.其次是发送响应的DHT公钥和随机数.在发送响应中加密部分包含了一个字节为1的值其次是8字节的ping id.
	
	所有的发送请求接收都会被解密.如果成功解密请求,将会对同一个节点返回一个响应.

#### Get nodes(Request)

	[字节值02  发送者的DH公钥长度为32] 随机24字节数 发送者的私钥和接收者的公钥用同一密钥加密,请求节点公钥(我们想要找到的DHT公钥)长度为32字节,ping_id 在响应中必须没有修改的返回 长度为8字节 
	
	可用的转发：一个发送节点包

	节点请求的首字节是2.其次跟着发送者的DHT公钥和随机数. 请求加密部分是发送者查找或者想要找到在DHT中最接近它的DHT密钥.其次跟随着8字节的ping id用于发送请求.

    通过发送节点答复得到节点的请求和响应(Get node requests are responded to by send node responses).发送节点的响应应该包含4个在他们的已知节点列表里面最近的可以使用(没有超时)的节点.

#### Send_nodes(response)

    字节值04 发送者的DHT公钥长度为32,24字节随机数,用随机数加密发送者的私钥,uint8_t numbser of nodes在这个包中. 节点在包节点格式,长度=(ipv4 39,ipv6 41)*(节点数字( 最大值4)字节) ping_id 长度8字节

    发送节点响应的首字节是4. 其次跟随着发送者的DHT公钥和随机数. 响应的加密部分是包它存储在分组节点的数目的1字节数,最多4个包节点格式的节点,8字节的ping id会在请求中发送. 

    Toxcore 存储着离他DHT公钥最接近的32个节点,在它DHT同盟列表中最接近每一个公钥的8个节点(它尝试找到链接的DHT公钥列表)每60秒请求它们一次来判断它们是否存活中.节点可以存在多个列表中,例如对等节点的DHT公钥非常接近查找的同盟的DHT公钥.它可以每隔20秒对列表中的随机节点发送得到节点请求.通过查找离他公钥最近节点的公钥,被查找的公钥在那些DHT 同盟列表中.(随机让它不可预测,可预测的或者知道哪一个节点将会请求下一个将会产生一些攻击中断网络,更容易作为它增加可能的攻击载体).122秒后节点没有响应,节点将会被移除.在得到可用的ping响应或者从发送节点包是从它们那接收到,节点被增加到列表中.如果节点已经存在于列表中且ip地址更改了,该节点会被更新.如果列表不是满的或者节点的DHT公钥比在列表中查找到的最近节点的DHT公钥更接近,节点只可以被添加到列表中.当节点被添加到满的列表的时候,它将会替换最远的节点.

    如果32个节点数在增加,它将会增加需要检查的包的数量,是否它们每一个还存活着,将会  增加带宽使用,但是可靠性将会增加.如果节点数量减少,可靠度和带宽使用会降低.其原因是可靠度和节点的数量之间的关系,如果我们假设不是每一个节点都打开它的UDP端口或者在锥形NAT的后面,它意味着这些节点每一个必须可以存储一定数量的背后限制节点NATs,以便于其它人在列表中可以找到背后的那些限制节点.例如如果7/8节点是背后的限制NATs,使用8个节点将会不够,因为这些节点不能在网络中找到的可能性太高了.

	如果ping超时,延误是高的,它将会减少带宽使用但是会增加仍然存储在列表存储的不可连接的节点的数量,减少这些延迟会有相反的影响.如果最接近每一个公钥的8个节点增加到16个,它将会增加带宽的使用,可能增加对称NATs的打孔效率(more ports to guess from,see Hole punching),可能会增加可靠度. 降低这个数字会有相反的影响.

	当接收到发送节点包的时候,toxcore 将会检查每一个接收到的节点是否被添加到了任何一个列表中,如果节点可以用,toxcore 将会对他发送ping包,如果它不能用,将会被忽略.当接收到节点包的时候,toxcore 将会在它的节点列表中找到最接近包里公钥的4个节点,在发送节点响应中发送它们. The timeouts and number of nodes in lists for toxcore where picked by feeling alone and are probably not the best values. This also applies to the behavior which is simple and should be improved in order to make the network resist better to sybil  attacks.(这句话实在是不知道该怎 么翻译了)

#### DHT Request packets:

	值为32的字符 接收者的DHT公钥(32字节) 发送者的DHT公钥(32字节) 随机数 (24字节) 加密信息

	DHT请求包可以发送跨越它们知道的一个一对一DHT节点. 他们被用来发送对同盟的加密数据,以至于我们不能在DHT中直接连接.

	一个接收DHT请求包的DHT节点将会检查 节点是否有接收者的公钥 也就是他们的DHT公钥,如果是,他们将会解密处理包,如果不是,他们将会检查是否他们知道DHT公钥(是否它在它们接近节点的列表中)如果不是,他们将会删除包,如果是,他们将会重新发送 包到那个DHT节点. 

    加密的信息是用接收者的DHT公钥,发送者的私钥和随机数(随机的24字节)加密的.

    DHT请求包被用来DHTPK包 (see onion)NAT ping 包.

	NAT ping 包 (在DHT处在请求包里) uint8_t 值254 0字符 8字节随机数,uint8_t 值254 1字符 8字节随机数 和请求发送的是一样的.NAT ping 包被用来查看在线上我们不能直接连上,准备好打孔的同盟.

#### hole punching


	我们假设人们使用三种NAT类型中的一种tox：

	Cone NATs : 在NAT后面对每个UDP socket许可一个接口,来自于任何ip/port 发送到许可的端口任何数据包,将会被internet转发到它后面的socket.

	Restricter Cone NATs: 在NAT 后面对每一个UDP socket许可一个端口,然而,它将只会转发从ips包到UDP socket .

	Symmetric NATs: 最不好的一种NAT类型,他们对每一个 ip/port 许可新的端口包发送的,他们 对待每一个新的成员发送UDP包,作为链接,将会只转发从连接到ip/port的包.

    在正常的cone NATs中打洞通过简单的DHT功能实现.
    
    如果在DHT中多于8个成员的一半最接近同盟,返回同盟的 ip/port ,我们对每一个返回的ip/ports发送ping请求,但是没有得到响应.如果我们对 4个ip/ports 发送4个ping请求,据说是同盟但是没有得到响应.这对tox来说足够它开始打洞了. 数字8和4 被用在toxcore,单独基于感觉被选择,因此可能不是最好的数字.在打洞开始之前,对称节点将会通过他们知道的同盟节点对同盟发送NAT ping包. 如果NAT ping 用接收到的相同的随机数响应,打洞将会开始.   

	如果接收到NAT ping请求,我们将会首先检查它是不是来自于同盟.如果它不是来自于同盟,它将会被删除.如果它来自于同盟,通过知道同盟发送请求的节点,用请求中相同的8字节响应发回.如果不是来自于知道的同盟的节点,包将会被删除.
	
	接收到NAT ping响应 意味着同盟是在线积极的查找我们,那是他们知道节点,知道我们唯一的途径 . 这是重要的因为只有同盟积极的尝试连接我们,打洞才将会工作.

	NAT ping在toxcore每隔3秒发送一次请求,如果6秒中没有接收到响应,打洞将会停止. 在长时间的间隔发送他们可能会增加其他节点下线,在打洞中,ping 包发送到僵死的成员,但是降低带宽的使用.降低间隔时间可能将带来相反的影响.
	
	这是两个关于toxcore处理打洞的例子. 第一个例子是 如果有每4个以上的对称节点返回相同的ip和端口,第二个是4个以上的对称成员返回一个ip但是端口不同.

	第三个例子是可能发生的,就是对称节点返回不同的ip和端口. 这只能发生,如果同盟是后面限制NAT,不能打洞或者对称节点最近连接另一个网络,一些对称节点依旧有一些老的存储. 因此,我们可以 把什么都不做作为第一选择,建议使用对等体返回常见ip,忽略其他的ip/ports.
	
	这个例子中,对称节点返回相同的ip和port,它意味着其他的同盟是限制Cone NAT. 这些NATs的类型可以打洞,通过得到同盟,对我们公共的ip/port发送一个包 . 这意味着打洞可以简单的到达ip/port,我们应该继续发送DHT ping包到ip/port,直到我们得到一个ping响应. 同盟查找我们在DHT中将会找到我们,尝试用打洞发送一个包到我们的公共IP/port从而建立一个连接.
	
	对于这个例子不能返回一样的端口号,这意味着其他节点是对称的NAT. 这些对称的NAT在序列中打开端口号,所以端口号可能会被其他的节点返回 像 1345,1347,1389,1395. hole punch,当他们尝试发送我们ping请求的时候,这些 NAT方法是尝试猜更多可能被其他节点使用的端口号,发送一些ping请求到这些端口号.
	
	Toxcore 仅仅是尝试每一个返回的所有的端口号,(ex: 对于4个前面端口 它可能会尝试1345,1347,1389,1395,1346,1348,1390,1396,1344,1346...)越走越远,尽管这也在工作,方法应该可以得到改善. 使用这个方法,toxcore将会最多尝试48个端口号,每三秒钟尝试一次,直到两个连接上了. 5次尝试以后,toxcore将会双倍尝试,且开始从1024端口号尝试(48次)和先前一个猜测的端口号. 这是因为我有注意到  这个看起来对一些对称的NATs修正了它,很可能因为它们中的大部分重新开始计数为1024.

	增加每秒尝试端口号的数量会让打洞走的更快,但是可能由于大量的包,在短的时间内被发送到不同的ips导致DoS NATs .降低它可能会让速度打洞变慢 .

	这个例子中工作有不同的NATs. 例如如果A和B尝试连接对方,A有对称的NAT B有受限制锥形NAT.A将检查B有一个受限制锥形 NAT且保持发送ping包到它的一个ip/port,B将检查A有对称NAT且将会尝试发送包到它猜测的端口号. 如果B管理 猜测端口号A发送包,他们将连接在一起.


补充一些知识： 

- ip 32位字节中,ip首部加入了长度为8 bit的数值,称作协议域.1表示ICMP,2表示IGMP,6 表示TCP,17表示UDP.

- ip首部+UDP首部+UDP数据=ip数据报
  
- Sybil attack : the Sybil attack in computer security is an attack wherein a reputation system is subverted by forging identities in peer-to-peer networks. 
  
- NAT  network address translation  https://en.wikipedia.org/wiki/Network_address_translation
  
- hole punching

	https://en.wikipedia.org/wiki/Computer_networking
  
- peer 对称节点,简称节点
  


