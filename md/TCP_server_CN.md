	TCP服务器在tox有一个实现目标,像在不能直接连上彼此的或者因为一些原因限制使用TCP协议连接彼此的客户端之间进行TCP重放.TCP_server是典型只在真实的服务器机器上运行的,但是任何一个Tox客户端可以承载一个作为api 运行一个通过tox.h api曝光的客户端.
	
	toxcore用TCP客户端模块连接TCP server主机.

	在toxcore,TCP服务器可以在epoll 在linux,或者使用未优化但是轻便的socket polling,来实现当前任何一个工作.

	在TCP客户端和服务器之间的TCP连接加密,用来防止外人仅仅是看到有人连接tcp服务器,知道像谁连接谁的信息.当有人通过某物连接服务器,这是有用的,例如像tor.它也防止有人在流中注入数据和制造它,所以我们可以假设任何接收到的数据都不是篡改过的,正是客户端发送的数据.

	当客户端第一次连接tcp服务器的时候,他对ip/port 挂起了tcp连接 ,tcp服务器处于监听状态.一旦连接建立,他会发送握手包,服务器用对他自己的响应,安全连接建立.连接据说是没有确认的,在服务端标记连接确认之前,客户端必须发送一些加密的数据给服务端.因为像这样工作可以防止节点发送一个握手包之后立即超时的攻击类型.为了防止这个,服务器在标记确认连接之前,客户端收到它的握手包,服务器必须等待一些时间.它们可以通过加密连接和对方对话.

	TCP 服务器机器重要的行为作为两个节点的重放.当TCP客户端连接服务器的时候,他告诉服务器 他想服务器去连接他哪一个客户端.

	两个客户端如果都指示服务器他们想要连接彼此.服务器将会只让他们相互连接.这是用来防止如果是来自于某人连接到服务器的检查,而不是来自于同盟.

	TCP服务器支持通过它盲目的对客户端和公钥为x客户端发送包,然而TCP server没有给任何反馈 或者任何提示,如果包到达或者没有达到,像这样它只是有用的去发送数据对同盟,可能不知道我们连接的当前的TCP服务器,当我们知道他们是的时候.

	当一个节点发现TCP重放和其他节点的DHT公钥的时候,其他节点还没有发现它的DHT公钥.在那个例子OOB将被使用,直到其他节点知道节点是连接重放和通过它建立连接.

	为了让toxcore工作在TCP,TCP服务器只能支持重放从TCP客户端来的onion包,且发送任何响应到TCP客户端.

	建立一个安全的连接和TCP服务器发送一个128字节的数据或者握手包给服务器.

	[DHT public key of client (32 bytes)][nonce for the encrypted data [24 bytes]][encrypted with the DHT private key of the client and public key of the server and the nonce:[public key (32 bytes)][base nonce the TCP client wants the TCP server to use to encrypt the packets sent to the TCP client (24 bytes)]]

	前32位字节是公钥(DHT公钥),是TCP客户端对服务器声明它自己.其次24字节是随机数,TCP客户端用和包中的前32字节公钥有关联的密钥来加密这个包的其余部分.

	这个包加密部分包含临时公钥,在连接期间,它将会被用来加密,之后将会丢弃. 它也包含了基本的随机数 将会稍后用来加密发送给客户端的数据包.

	如果服务器解密成功在握手包中的加密的数据,响应以下的96字节长度的握手响应：

	[nonce for the encrypted data [24 bytes]][encrypted with the private key of the server and the DHT public key of the client and the nonce:[public key (32 bytes)][base nonce the TCP server wants the TCP client to use to encrypt the packets sent to the TCP server (24 bytes)]]--懒得翻译了,因为我懒

	客户端已经知道服务器的长期公钥,所以它在响应中忽略,代替只有随机数存在的没有加密的部分.加密部分的响应有相同的元素像加密部分请求的.一个临时公钥绑在这个连接 基本的随机数将会稍后被用于当解密接收到的包来自于TCP客户端这个连接唯一的. 

	在 toxcore 基本的随机数 生成 随机数, 像所有的其他的随机数一样,他必须预防生成重复的随机数.例如如果随机数用0 对于两边 用相同的key 去加密包 他们发送给对方,两个包会用相同的随机数加密.这些可能重播回给发送者可能会导致问题. 一个类似的机制在net_crypto中使用.

	这个客户端将会知道 连接临时公钥和服务器的随机数,服务器将会知道连接的随机数和临时公钥的客户端.客户端将会发送加密包给服务器,包的内容不合适,它必须正常被服务器处理(ex: 如果它是一个ping 发送ping响应,第一个包必须是任何一个可用的加密的数据包).唯一的事情就是包正确被客户端加密,因为它意味着客户端有正确接收服务器发送给它握手响应,客户端发送服务器 的握手是真实的从客户端来的,而不是来自于攻击播回包.

	服务器必须预防资源被超时客户端持续攻击,如果它们没有发送任何加密包,所以服务器证明给连接是正确建立的服务器 .

	toxcore对客户端没有超时,在大的circular表中,代替它存储连接的客户端,次数会让它超时.如果它们的实体在列表中,会被新的连接替代了,因为 这是 它预防在当前连接节点上有来自于 有一个本地的碰撞的TCP洪水攻击 .然而更好的途径做这个且唯一原因toxcore用这种方法做了,因为写它非常简单. 当连接确认他们 是被移除的某些地方.

	当服务器确认连接 他必须 查看连接的节点列表 看他是否准备好连接客户端,用相同声明的公钥. 如果这个例子 服务器必须杀掉 前一个连接,因为这意味着前一个客户端 超时 ,重新连接. toxcore 设计 它 非常不喜欢发生的 两个legitimate不同节点 将会使用相同的公钥 所以这是一个正确的行为.



	Encrypted data packets look like this to outsiders:

	对于外来者,加密数据包看起来就像这样:

	[[uint16_t (length of data)][encrypted data]]

	In a TCP stream they would look like:
	[[uint16_t (length of data)][encrypted data]][[uint16_t (length of data)][encrypted data]][[uint16_t (length of data)][encrypted data]]...


	Both the client and server use the following (temp public and private (client and server) connection keys) which are each generated for the connection and then sent to the other in the handshake and sent to the other. They are then used like the next diagram shows to generate a shared key which is equal on both sides.


	Client:                                     Server:
	generate_shared_key(                        generate_shared_key(
	[temp connection public key of server],    [temp connection public key of client],
	[temp connection private key of client])    [temp connection private key of server])
	=                                           =
	[shared key]                                [shared key]

	生成的共享密钥对于两边来说都是一样的,被用来加密和解密被加密的数据包.

	每个加密的数据包发送到客户端将会用共享密钥,随机数加密,等价于(客户端的基本随机数 + 包的数量 发送 所以 对于首个包(开始是0) 它是 随机数+0,第二个是随机数+1,等等. 注意 随机数像所有的其他数字发送到network,在toxcore,是big enian 格式,所以当增加他们 通过1 至少 签名)

	每个从客户端接收到的包 将会用共享密钥和随机数解密 (服务端 基于随机数+包的发送个数 所以 第一个包 开始是0 随机数+0 ,第二个包是随机数+1 等等,note 随机数 像 其他所有的数字 发送 over 网络  在toxcore   是数字  big endian 格式,当增加他们 通过1,最近签名字节是最后一个)  


	加密数据包有两个最大的大小限制 2+2048 bytes, 在toxcore tcp server 实现2048 bytes 足够确保所有的toxcore包可以通过, 留有一些空余的空间以防止协议在 在未来需要改变. 2 bytes 代表了数据的大小, 2048是加密部分的最大字节.这也就意味着最大的大小是2050 字节 .
	在当前的toxcore,最大的发送加密数据包是2+1417,也就是总共2419字节.

	握手机制背后的逻辑：
	1. 需要证明给服务器 ,我们自己的私钥以及涉及到的公钥 ,我们用来证明我们自己的.
	2. 需要建立一个安全的连接 完美的促进安全.
	3. 预防任何重播,模仿或者其他的攻击

	How it accomplishes each of those points:
	1.如果客户端不拥有私钥相关的公钥,他们将不能创建握手包.
	2. 在握手包的加密部分,客户端和服务端生成的临时session keys,在session期间,被用来加密解密包 .
	3. a.攻击者修改握手包的任何一些字节:解密失败,可能没有攻击.
		b.攻击者从客户端捕获握手包,稍后向服务端重播他们,攻击者将不会得到服务端的确认连接
		c. 攻击者捕获服务端的响应,下次发送他们给客户端,他们尝试连接服务端 客户端将不会确认连接
		d. 攻击者尝试假冒服务端,他们不能解密握手包且不能产生响应
		e. 攻击者尝试冒充客户端,服务端不能解密握手包

	加密包背后的逻辑：
	1.TCP是一个流协议,我们需要包
	2.任何攻击都必须被阻止

	How it accomplishes each of those points:

	1. 在每个加密的包前边增加2字节,象征着长度. 我们假设正常的TCP为了让它工作将会传送这些字节. 如果TCP没有增多,意味着,它是在攻击下和为了看到下一个点.
	2. a.修改长度也将会导致连接超时或者解密失败
	   b.修改任何加密字节将会导致解密失败
	   c.注入任何字节会导致解密失败
	   d.尝试重新组合包会导致解密失败,因为有序的随机数
	   e.从流中移除任何包将会导致解密失败,因为有序的随机数

	The folowing represents the various types of data that can be sent inside encrypted data packets.



	//TODO: nice graphics

	0 - Routing request.
	[uint8_t id (0)][public key (32 bytes)]
	1 - Routing request response.
	[uint8_t id (1)][uint8_t (rpid) 0 (or invalid connection_id) if refused,connection_id if accepted][public key (32 bytes)]
	2 - Connect notification:
	[uint8_t id (2)][uint8_t (connection_id of connection that got connected)]
	3 - Disconnect notification:
	[uint8_t id (3)][uint8_t (connection_id of connection that got disconnected)]
	4 - ping packet
	[uint8_t id (4)][uint64_t ping_id (0 is invalid)]
	5 - ping response (pong)
	[uint8_t id (5)][uint64_t ping_id (0 is invalid)]
	6 - OOB send
	[uint8_t id (6)][destination public key (32 bytes)][data]
	7 - OOB recv
	[uint8_t id (7)][senders public key (32 bytes)][data]
	8 - onion packet (same format as initial onion packet) but packet id is 8 instead of 128)
	9 - onion packet response (same format as onion packet with id 142 but id is 9 instead.)
	16 and up - Data
	[uint8_t connection_id][data]

	当转播许多包,TCP服务器设置的方式尽量减少浪费,可能到两个tox之间去,在转播上,因此客户端必须创建 对其他他客户端的连接. 连接数字是uint8_t 必须等于或者大于 16 ,为了让它是可用的.因此uint8_t 有一个256的最大值,它意味着不同的连接的到其他客户端,每个连接可以有最大值为240. 因为可用的connection_ids 是大于16 ,它们是数据包的首字节. 目前只有0到9 是保留的,然而我们保留了一些额外的情况下,我们需要不破坏它去扩展协议 .

	Routing request(sent by client to server):
	对我们想用公钥(公钥是一个公开节点用来声明它们自己的)连接节点的服务器发送一个路径请求.服务器必须用Routing response 对这个响应 .

	Routing response (Sent by server to client):

	对 routing request 的响应,如果 routing request 成功（ 可用的connection_id）告诉客户端 
	告诉他们connection 的id. 如果在routing request 中发送了公钥,那么在响应中也将会发送这些公钥. 所以客户端可以在同一时间对服务器发送很多请求,我们不需要代码跟踪哪些响应属于哪些公钥.

	routing request应该会失败的唯一原因是连接到达了同时连接的最大数. 在情况下,路由请求失败响应中的公共密钥将是在失败的请求公钥.

	Connect notification (Sent by server to client):
	告诉客户端现在连接的 connection_id,意味着另一个是在线的,数据可以使用这个connect_id发送 .

	is client has simply disconnected from the TCP server.

	Disconnect notification (Sent by client to server):
	在通知中,当客户端想让服务器忘记关于这个连接涉及到的connect_id 的时候发送断开连接通知.服务器必须移除这个连接,必须对其他的连接重用这个connection_id. 如果这个连接是被连接的 服务器必须对其他的客户端发送断开连接通知.其他的客户端必须认为这个客户端已经简单的从tcp 服务器断开连接 .

	Disconnect notification (Sent by server to client):
	服务器发送给客户端,告诉它们连接的connection_id 已经断开连接. 当其他的客户端的连接断开或者当他们告诉服务器杀死连接,它也发送断开连接的通知.

	ping 和pong包（可以被客户端和服务器两个发送,都将会响应）ping 包 被用来知道其他的连接的对方是否存活着.TCP 当建立的时候没有理智的超时（1周是不理智的） 所以我们有义务用我们自己的方法去检查另一边是否存活. ping ids 可以是除了0以外的任何东西,这是因为 当它接收到pong 响应,toxcore 设置变量存储ping_id,也就是发送的0,意味着0是无效的ping_id. 

	服务器应该每x秒发送ping 包（toxcore TCP_server 每隔30秒发送它们,如果没有在10秒内得到响应,节点超时）. 服务器应该用pong 包直接响应ping 包

	服务器应该用pong 包和ping 包中相同的ping_id响应ping 包. 服务器应该检查每个pong包 包含在ping中相同的ping_id,如果没有,pong 包必须被忽略,

	00B send(Send by client to server):
	如果节点的私钥等于他们在连接中声明自己的密钥,00B 发送包中的数据将会作为 00B recv 包被发送到节点.如果没有节点是连接的,包将会被丢弃.toxcore TCP_server 实现了1024的最大00B数据长度.1024被选中是因为它对于net_crypto包涉及的握手是足够大对于 协议的任何修改都不需要打断TCP server. 然而它并不是足够的对于最大的net_ctypto包与发送一个既定的net_crypto连接来禁止通过00B包发送这些。

	OOB recv (Sent by server to client):
	OOB recv are sent with the announced public key of the peer that sent the OOB send packet and the exact data.

	OOB packets can be used just like normal data packets however the extra size makes sending data only through them less efficient than data packets.

	Data:
	Data packets can only be sent and received if the corresponding connection_id is connection (a Connect notification has been received from it) if the server receives a Data packet for a non connected or existent connection it will discard it.

	Why did I use different packet ids for all packets when some are only sent by the client and some only by the server? It's less confusing.

