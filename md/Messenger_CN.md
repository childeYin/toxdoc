	信使在所有其他模块的顶部。在toxcore的层次结构里,它位于friend_connection的顶部 。

	信使负责使用friend_connection提供的连接发送和接收消息。这个模块给好友提供了连接方法,让它当做一个可用的即时消息系统。例如,信使让用户设置昵称和状态,然后当他们在线的时候发送给好友。它也允许用户发送消息给好友,在friend_connection 底层模块的基础上,建立一个即时消息系统。

	信使提供了两个添加好友的方法。第一种方法是通过她们的长期公钥添加好友,当好友需要被添加,但是因为一些原因,好友请求不应该被发送的时候使用。好友应该仅仅被添加。这个方法大多公认被用来接受好友请求,但是也被用于其他方法中。如果两个朋友通过这个功能,互相添加对方,她们将会互相连接。通过使用这个方法,添加一个好友,仅仅添加到friend_connection 中,且在信使里面为这个好友,创建一个新的好友条目。

	Tox Id 被用来鉴别节点,所以在tox中她们可以作为好友被添加。为了添加好友,tox用户必须有好友的Tox Id。Tox Id包含节点的长期公钥(32字节) ,4字节的随机值(friend_requests) 和2字节的xor 校验和. 发送Tox Id的方法取决于用户和客户端,但是推荐方法是用16进制格式编码它,让用户使用另一个程序,手动发送他给好友。

	Tox ID:
		[long term public key (32 bytes)][nospam (4 bytes)][checksum (2 bytes)]

	校验和通过异或id的前两个字节,下两个字节,下两个字节直到36字节都被异或,结果被追加到Tox Id的尾部。

	用户必须确保Tox Id不被截获,在传输的过程中不被不一样的Tox Id替换,这也就意味着好友 可能连接到恶意的人,而不是用户,虽然采取合理的预防措施,这是在Tox的范围之外。 tox 假设 用户确保她们使用正确的Tox Id,属于预订的人,来添加好友。

	第二种添加好友的方法是通过使用她们的Tox Id和一个发送在好友请求里面的消息。这个方法添加将会尝试给想要添加的节点的Tox Id,发送一个设置消息的好友请求. 这个方法和第一个类似,除了生成好友请求,并发送给其他节点。

	当一个好友连接关联到信使好友上线的时候,一个在线包将会发送给她们。 好友只设置在线 如果在线包被接收了。

	同理,当好友离线的时候,信使将停止给好友发送好友请求,如果她发送她们,她们对于这个好友也是过剩的信息。

	如果好友连接关联到她们离线,或者如果从好友那接收到离线包,好友将设置离线。

	信使包使用对好友的在线好友连接,发送给好友。

	信使需要去检查无序包是否在被好友接收到的列表中,例如,实现对文本信息的接收,net_crypto可以被使用。 net_crypto 包数字,用于发送包,应该被记录,然后 net_crypto检查后接收的数字,查看它是否在这个包数字的后边。如果他是,好友接收它。长时间记录net_crypto包数字 可能会导致溢出,所以检查应该发生在用相同的好友连接发送的在 2^32个 net_crypto包内。

	动作消息和正常的文本消息的消息接受是被实现的,通过将每个消息的net_crypto包数字,以及接收编号,添加到每个好友在发送他们的时所具有的链接列表的顶部来实现。每个信使循环,条目从底部读取,条目被移除且传递给客户端,直到到达的条目没有被其他的包接收,当这种情况发生时停止。


List of Messenger packets:

ONLINE: 
	
	length: 1 byte
	[uint8_t (24)]

	当连接建立,发送在线包给好友,告诉它们在他们的好友列表中,标识我们在线。这个包和离线包是被需要的,可以和群组中不是好友的人,建立friend_connections。两个包被用来区分不同的节点,通过群聊连接用户,真实的朋友在好友列表中被标识在线状态。 

	一旦接受这个包,信使将会作为在线,展示节点。

OFFLINE:
	
	length: 1 byte
	[uint8_t (25)]

	当删除好友的时候,发送给好友。防止删除的好友看到我们在线,如果我们通过群聊连接她们 。

	一旦接受这个包,信使将展示这个节点离线。

NICKNAME:
	
	length: 1 byte to 129 bytes.
	[uint8_t (48)][Nickname as a UTF8 byte string(Min length: 0 bytes,Max length: 128 bytes)]

	用来给其他的节点发送节点昵称。她们每次的上线和修改昵称,这个包都应该发送给每一个朋友。

STATUSMESSAGE:

	length: 1 byte to 1008 bytes.
	[uint8_t (49)][Status message as a UTF8 byte string(Min length: 0 bytes,Max length: 1007 bytes)]

	用来给其他节点发送节点的消息状态。她们每次的上线和改变消息状态,这个包应该发送给每一个好友。

USERSTATUS:
	
	length: 2 bytes
	[uint8_t (50)][uint8_t status (0 = online,1 = away,2 = busy)]

	用来给其他节点发送用户状态。她们每次的上线和改变用户状态,这个包应该发送给每一个好友。

TYPING:
	
	length: 2 bytes
	[uint8_t (51)][uint8_t typing status (0 = not typing,1 = typing)]

	用来告诉好友,用户当前是否在输入。

MESSAGE:
	
	[uint8_t (64)][Message as a UTF8 byte string (Min length: 0 bytes,Max length: 1372 bytes)]

	用来发送正常的文本信息给好友。

ACTION:
	
	[uint8_t (65)][Action message as a UTF8 byte string (Min length: 0 bytes,Max length: 1372 bytes)]

	用来发送动作消息给好友(像IRC 动作)

MSI:

	[uint8_t (69)][data]
	为Tox AV usage保留.

File Transfer Related Packets:

FILE_SENDREQUEST:

	length: 4 bytes if control_type isn't seek. 8 bytes if control_type is seek.
	[uint8_t (81)][uint8_t send_receive (0 if the control targets a file being sent (by the peer sending the file control),1 if it targets a file being received)][uint8_t file number][uint8_t control_type (0 = accept,1 = pause,2 = kill,3 = seek)][uint64_t seek parameter (only included when control_type is seek)]

	注意如果它包含随机参数,将会用 endian/network格式发送。

FILE_DATA:

	length: 2 to 1373 bytes.

	[uint8_t (82)][uint8_t file number][file data piece (Min length: 0 bytes,Max length: 1371 bytes)]

	Group Chat Related Packets:
		INVITE_GROUPCHAT 96
		ONLINE_PACKET 97
		DIRECT_GROUPCHAT 98
		MESSAGE_GROUPCHAT 99
		LOSSY_GROUPCHAT 199

	文件在tox中使用文件传输传输。

	实例化一个文件传输,好友创建和发送一个file_sendrequest包给好友,表示它想要实例化一个文件传输。

	FILE_SENDREQUEST包首部分是文件编号。文件编号是数字,被用来标识这个文件传输。当文件编号被代表使用1字节数字,tox发送给好友的最大的文件数是256.256文件传输每个好友是足够的,如果有很多文件需要被发送,客户端可以使用技巧,像队列文件。

	给每个好友传出256个文件,意味着最大的同时文件传输是512,在两个用户中,如果同时接收和传出文件传输被计算在一起。

	文件编号被用来识别文件传输,tox实例必须确保对于同一个好友创建一个新的传出文件传输,正在使用的文件编号不被其他的传出文件传输所使用。文件编号被选择通过文件发送者和没有更改的entire期间的文件传输。文件编号被使用,通过FILE_CONTROL 和FILE_DATA两个包,来识别来自于哪个文件传输。

	文件传输请求的第二部分是文件类型。这个简单的数字,标识文件的类型。例如 tox.h定义了文件类型,0作为正常的文件和 1作为avatar,意味着tox客户端应该使用file,作为一个avatar. 文件类型不影响文件怎样传输,或者文件传输的行为。它被tox client设置创建文件传输和发送一个好友未接收。

	文件大小表示文件总得大小将被传输的。一个文件大小UINT64_MAX(最大值 在 unint64_t)意味着文件大小是不确定的或者未知的。例如,如果一个人想要使用tox文件传输传输数据流,她们可以设置文件大小为 UINT64_MAX.一个文件大小是0是可用的,其行为与正常的文件传输完全相同。

	文件id是32字节,可以用来作为文件传输的唯一标识。例如,avatar传输器,使用它作为avatar的哈希,所以接收者可以检查她们是否已经有avatar,对好友更节省带宽。它也被用来标识打断的文件传输,通过toxcore重启(更多的信息查看文件传输片段 在tox.h)。文件传输实现了不需要关心文件id是什么,它只被上游的东西使用。

	文件传输的最后一部分是文件名字,用来告诉接收者文件的名字。

	当一个FILE_SENDREQUEST包被接收的时候,实现可用和发送信息给tox客户端,决定她们是否可以接受文件传输。

	拒绝或者取消文件传输,她们将会发送FILE_CONTROL包 和 control_type 是2(kill)

	FILE_CONTROL包用来控制文件传输。FILE_CONTROL包用来接受或者取消暂停,暂停,杀死/取消 和查找文件传输。control_type参数表示file control包做什么。

	send_receive和文件编号用来标识特殊的文件传输。文件编号对于其他的来说,传出或者传入文件不是相互联系的。send_receive参数被用来识别,文件编号是否属于正在发送文件或者被用户正在接收的文件。如果send_receive 是0,文件编号对应一个文件 被发送通过用户发送file control包。如果send_receive是1,它对应一个文件被接受,通过用户发送file control 包。

	control_type标识FILE_CONTROL包的目的。control_type是0,意味着FILE_CONTROL包被用来告诉好友,文件传输是被接受或者我们是取消暂停先前暂停(被我们)的文件传输。control_type是1被用来告诉其他来暂停文件传输。

	如果一方暂停文件传输,那一方必须是取消暂停的一方。两边都暂停文件传输,两边都取消暂停,在文件可以被恢复之前。例如,如果发送方暂停文件传输器,接收方必须不能取消暂停它。取消暂停一个文件传输,control_type是0.在传输中且她们被接收中文件只被暂停。

	control_type 2用来杀死,取消或者拒绝一个文件传输。当FILE_CONTROL被接收,目标文件传输被认为已经死亡,将立即擦拭和它的文件编号可以被重用。节点发送FILE_CONTROL必须也被擦拭,目标方文件传输来自于其他的方。在任何时候,control type都可以被传输的两边使用。

	control_type 3,seek control type被使用,告诉文件的发送方启动发送从文件里面不是0的位置开始,接受FILE_SENDREQUEST包之后,它可以被正确使用。 接受文件之前,FILE_CONTROL和control_type是0。当这个control_type被使用,一个额外的big endian格式的8字节数字被追加在FILE_CONTROL,并不代表其他的control类型。这个数字标识文件开始传输的位置,文件发送者开始发送的文件。这个control type的目标是确保文件通过core重启,可以被恢复。tox客户端可以知道,如果她们有通过正在使用的file id,接受一个文件的一部分,她们使用这个包告诉其它方从最后一个接受的字节开始发送 。如果查找位置是大于或者等于文件的大小的,查找包是不可用的,接收方会丢弃它。

	接受一个文件tox将会发送一个查找包,如果它是需要的,然后发送一个control_type是0的FILE_CONTROL包,告诉文件发送者,文件被接收。

	一旦文件传输被接收,文件发送方将开始从文件的开头(或从FILE_CONTROL寻找包的位置,如果接收到)的顺序块中发送文件数据。

	文件数据使用FILE_DATA包发送。文件编号对应文件块从属的文件传输。一旦接收到文件数据大小不等于最大大小(1371字节)的块,接收器假定文件传输结束。这就是发送者怎样告诉接收者文件传输是完成的在文件传输中,文件的大小是未知的(设置UINT64_MAX)。接收者 也假设如果接受的数据总量等于文件大小 在FILE_SENDREQUEST中的数量,则文件发送是完成的,并且成功被接收。在这之后,接收者立即释放文件编号,让一个新的传入的文件传输可以使用文件编号。实现应该丢弃任何额外数据接收,当接收到的大于开始的时候的文件大小 。

	FILE_DATA包应该尽可能快的被发送, net_crypto连接可以根据用堵塞状况处理它。

	如果好友下线, 在toxcore中,所有的文件传输将被清除的。这让它对于toxcore更简单,它不需要处理恢复文件传输。它也对于客户端来说,更加简单,恢复文件传输保持相同的, 即使客户端重启或者toxcore丢失对好友连接,因为一个坏的网络连接。

	信使也关心保存的好友列表和其他好友信息,让他关闭或者启动toxcore,同时保持你的所有的好友,你的长期公钥和重新连接网络需要的必要信息

	重要的信息信使存储包含：长期私钥,我们当前的随机数,我们好友的公钥和一些好友请求,用户是当前发送的。网络DHT节点,tcp 重放,一些洋葱节点被存储,用来重新连接。

	附加说明,大部分选项数据可以被存储,例如,好友的用户名,我们当前的名字,好友消息的状态,我们的状态信息等等都可以被存储。 toxcore保存的确切格式稍后解释。

	如果客户端已启用TCP服务器,则从toxcore messenger模块运行TCP服务器。 tcp server是通常作为bootstrap节点包的一部分,但是它可以被使用在客户端中。如果它在toxcore中可用,信使将会把运行的tcp server添加到tcp重放里面。

	Messenger是将可以基于公钥连接到朋友的代码转换为实时即时通讯的模块。

















