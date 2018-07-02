#### friend_connection
	
	friend_connection是一个位于DHT顶端的模块,onion和net_crypto模块满足三个连接在一起的需求。
	
	同盟在friend_connection被它们真实的公钥代表。当一个同盟添加在friend_connection,onion搜索名单为那个同盟创建。这就意味着 onion模块将会开始查询这个同盟和发送同盟的DHT公钥,它连接上tcp重放,如果一个连接只能使用TCP。
	
	一旦onion返回节点的DHT公钥,DHT公钥被保存,添加到DHT同盟列表和新的net_crypto连接被创建。任何tcp重放被onion返回为这个同盟传递到 net_crypto连接。
	
	如果DHT和同盟建立直接的UDP连接,friend_connection将会通过同盟的ip/port到net_crypto,并保存它,如果连接断开,使用它对同盟重新连接。
	
	如果net_crypto同盟有一个不一样的DHT公钥,这发生在同盟重启它们的客户端net_crypto将会通过新的DHT公钥到onion模块并且 将移除旧的DHT公钥,并用新的替换它。 当前 net_crypto连接进程将会被杀死使用正确的DHT公钥的新的连接将会被创建。
	
	当net_crypto连接对同盟在线的时候,friend_connection将会通知onion模块同盟是在线,它可以停止发送资源来查找同盟。当同盟下线的时候, friend_connection 将会通知onion模块,它再次开始寻找同盟。

	2种类型的数据包(活动数据包和tcp重放包)用net_crypto连接发送给同盟,在friend_connection层处理。活动数据包是 有包的id或者数据的首字节(只有一个字节在包里)是16.它们使用为了检查其他的同盟是否依旧在线。在toxcore , 这个包每8秒发送一次。如果这些包没有在32秒中被接收到, 连接超时并且连接被杀死。这些数字似乎引起了一个问题, 32秒不是太长。如果一个同盟超时,toxcore不会错误的看到它们的在线时间太长 通常当一个同盟下线的时候,它们几乎离线瞬间发送一个数据包断开连接。
	
	当停止通过创建新的net_crypto连接尝试重新对同盟的连接的时候定义为超时,在toxcore旧的超时和DHT节点的超时是一样(122秒)。 然而,它计算了从DHT公钥接收对同盟或者同盟的net_crypto连接在上线之后下线的的最后时间。 当超时的时候,最大的时间被用来计算。net_crypto连接将会被重新创建(如果连接失败)直到超时。

	在toxcore中,friend_connection每5分钟对每个连接同盟发送一个包含3个重放的列表(在tcp_connection,相同的数字作为tcp重放连接的目标数字). 直接发送重放之前,它们联系当前的net_crypto->TCP_connection连接。这有助于连接两个同盟一起作为同盟使用重放,接收包将会对 它们接收到它的net_crypto连接发送重放。当两边做这个的时候,它们将会可以使用重放连接每一个。包id或者共享重放包的首字节是17。这是随后 是存储在包的格式节点的tcp重放。 

	[uint8_t packet id (17)][TCP relays in packed node format (see DHT)]

	如果本地包ips被作为包的一部分接收,本地ip将会被发送重放包的节点的ip替换。这是因为我们假设这是尝试连接tcp重放最佳的方法。 如果节点发送重放使用本地ip,则发送本地ip应当被使用连接重放。
	
	对于所有的其它包的数据,通过friend_connection传递到上游的messager模块。它从net_crypto分开加密有损和无损数据包。

	friend connection满足对同盟建立连接的需求, 给上游的messager 层 一个简单的接口来接收和发送信息。且知道一个同盟是否在线,来 添加和移除同盟 






