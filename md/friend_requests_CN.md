	当tox用户用tox添加某人,toxcore将会尝试向那个人,发送添加朋友请求.添加好友请求包含了一个发送者的长期公钥,nospam数字和信息.

	传送长期公钥是好友请求的主要目标,因为对等节点需要找到它和发送者建立实例连接.如果他接受好友请求,长期公钥会被接收者添加到他的好友列表.

	nospam是数字,它被用来防止使用有效的好友请求来自于垃圾邮件网络.它确保只有看到对等节点tox id的人能够发送他们的好友请求.nospam是tox Id的一个组件.

	nospam是被对等节点设置的一个数字或者一个数字列表.只有接收包含被对等节点设置nospam的好友请求,将其发送到客户端,被用户接受或者被拒绝.nospam阻止在网络中的随机节点给不存在的朋友发送好友请求.nospam对于安全来说并不足够长,这也就意味着极具弹性的攻击者可以设法向某人发送spam好友请求.4字节足够用来阻止来自于网络中的随机对等节点.如某人找到了tox id,决定给它发送上百个spam好友请求,nospam 也允许tox用户发出不同的tox ids,甚至改变tox ids.改变spam可以停止spam的好友请求,并且对用户好友列表没有任何负面影响.例如,如果用户改变他们的公钥来阻止他们接受好友请求,它意味着她们可能基本上放弃所有他们的当前好友,因为好友和公钥是绑定的.一旦好友互相添加,nospam不在适用,也就意味着改变他,不会产生负面影响.

	Friend request:
	[uint32_t nospam][Message (UTF8) 1 to ONION_CLIENT_MAX_DATA_SIZE bytes]

	Friend request packet when sent as an onion data packet:
	[uint8_t (32)][Friend request]

	好友请求包,当作为一个net_crypto数据包发送(如果我们因为组聊天直接连接对等节点,但是和他们不是好友)

	当一个好友的tox id和消息被添加到toxcore,好友被添加进friend_connection,然后 toxcore 尝试发送好友请求.

	当发送一个好友请求,如果两个对等节点都在相同的聊天组里,toxcore将会检查发送好友请求的对等节点,是否已经使用net_crypto连接.如果是这个情况,好友请求将会使用那个连接,作为 net_crypto包发送,如果不是,它将会作为onion数据包发送.

	onion数据包包含发送者的真实公钥,如果net_crypto连接是建立的,它意味着对等节点知道我们的真实公钥.这就是为什么好友请求不需要包含对等节点的真实公钥.

	在toxcore,好友发送请求以指数级增长 2秒,4秒,8秒等.这就是好友请求重新发送但是重新发送的间隔这么大,基本她们也到期了.发送者没有办法知道,对等节点是否拒绝了好友请求,这也是为什么好友请求需要存活在一些方法中.记录最小的超时时间的时间间隔,如果toxcore不能发送好友请求，它会不停的重试，直到它可以发送它。不能发送好友请求的一个原因是， 在onion中没有找到好友,所以不能发送onion数据包给它们.

	好友请求被多次发送,意味着toxcore为了确保阻止相同的好友请求发送给客户端,它保存了它接受好友请求的最新的真实公钥列表,丢弃任何来自于列表中存在的真实公钥的好友请求.在toxcore中,列表是简单的循环链表.有很多方法可以提高和改进效率,作为循环链表它不是高效的,然而它在toxcore中,到目前为止,运行良好.

	来自于已经被添加到好友列表中的公钥的好友请求,应该被丢弃.


