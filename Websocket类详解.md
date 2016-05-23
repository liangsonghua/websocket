要理解socket就要先理解http和tcp的区别，简单说就是一个是短链，一个是长链，一个是去服务器拉数据，一个是服务器可以主动推数据。<br/>

而socket就是应用层与TCP/IP协议族通信的中间软件抽象层，它是一组接口。在设计模式中，Socket其实就是一个门面模式，它把复杂的TCP/IP协议族隐藏在Socket接口后面，对用户来说，一组简单的接口就是全部，让Socket去组织数据，以符合指定的协议。-来自网络。<br/>
# 一、握手 #

## 1、客户端发送请求 ##

websocket协议提供给javascript的API就是特别简洁易用。

    var url = "ws://172.0.0.1:8888";
    var ws = new WebSocket(url);
    ws.onopen = function(){
      console.log("握手成功，打开socket连接了。。。");
      console.log("ws.send(Websocket opened)");
      ws.send(("Websocket opened!"));
    };
    ws.onmessage = function(e){
      console.log("message:" + e.data);
    };
    ws.onclose = function(){
      console.log("断开socket连接了。。。");
    };
    ws.onerror = function(e){
      console.log("ERROR:" + e.data);
    };
  
# 2、服务器端 #

封装的类为WebSocket，address和port为类的属性。

## （1）建立socket并监听 ##

listen函数使用主动连接套接口变为被连接套接口，使得一个进程可以接受其它进程的请求，从而成为一个服务器进程。在TCP服务器编程中listen函数把进程变为一个服务器，并指定相应的套接字变为被动连接。其中的能存储的请求不明的socket数目。

    function createSocket()
     {
	    $this->master=socket_create(AF_INET, SOCK_STREAM, SOL_TCP)
	      or die("socket_create() failed:".socket_strerror(socket_last_error()));
	      
	    socket_set_option($this->master, SOL_SOCKET, SO_REUSEADDR, 1)
	      or die("socket_option() failed".socket_strerror(socket_last_error()));
	      
	    socket_bind($this->master, $this->address, $this->port)
	      or die("socket_bind() failed".socket_strerror(socket_last_error()));
	
	    socket_listen($this->master,20)
	      or die("socket_listen() failed".socket_strerror(socket_last_error()));
	    
	    $this->say("Server Started : ".date('Y-m-d H:i:s'));
	    $this->say("Master socket  : ".$this->master);
	    $this->say("Listening on   : ".$this->address." port ".$this->port."\n");
       }

然后启动监听，同时要维护连接到服务器的用户的一个数组（连接池），每连接一个用户，就要push进一个，同时关闭连接后要删除相应的用户的连接。
   
     public function __construct($a, $p)
     {
	    if ($a == 'localhost')
	      $this->address = $a;
	    else if (preg_match('/^[\d\.]*$/is', $a))
	      $this->address = long2ip(ip2long($a));
	    else
	      $this->address = $p;
	    
	    if (is_numeric($p) && intval($p) > 1024 && intval($p) < 65536)
	      $this->port = $p;
	    else
	      die ("Not valid port:" . $p);
	    
	    $this->createSocket();
	    array_push($this->sockets, $this->master);
    }

## （2）建立连接 ##

维护用户的连接池

    public function connect($clientSocket)
     {
	    $user = new User();
	    $user->id = uniqid();
	    $user->socket = $clientSocket;
	    array_push($this->users,$user);
	    array_push($this->sockets,$clientSocket);
	    $this->log($user->socket . " CONNECTED!" . date("Y-m-d H-i-s"));
     }
## （3）回复响应头 ##

首先要获取请求头，从中取出Sec-Websocket-Key，同时还应该取出Host、请求方式、Origin等，可以进行安全检查，防止未知的连接。

    public function getHeaders($req)
    {
	    $r = $h = $o = null;
	    if(preg_match("/GET (.*) HTTP/"   , $req, $match))
	      $r = $match[1];
	    if(preg_match("/Host: (.*)\r\n/"  , $req, $match))
	      $h = $match[1];
	    if(preg_match("/Origin: (.*)\r\n/", $req, $match))
	      $o = $match[1];
	    if(preg_match("/Sec-WebSocket-Key: (.*)\r\n/", $req, $match))
	      $key = $match[1];
	      
	    return array($r, $h, $o, $key);
  }

之后是得到key然后进行websocket协议规定的加密算法进行计算，返回响应头，这样浏览器验证正确后就握手成功了。

    protected function wrap($msg="", $opcode = 0x1)
     {
	    //默认控制帧为0x1（文本数据）
	    $firstByte = 0x80 | $opcode;
	    $encodedata = null;
	    $len = strlen($msg);
	    
	    if (0 <= $len && $len <= 125)
	      $encodedata = chr(0x81) . chr($len) . $msg;
	    else if (126 <= $len && $len <= 0xFFFF)
	    {
	      $low = $len & 0x00FF;
	      $high = ($len & 0xFF00) >> 8;
	      $encodedata = chr($firstByte) . chr(0x7E) . chr($high) . chr($low) . $msg;
    }
    
    return $encodedata;			
    }

其中我只实现了发送数据长度在2的16次方以下个字符的情况，至于长度为8个字节的超大数据暂未考虑。 

    private function doHandShake($user, $buffer)
    {
	    $this->log("\nRequesting handshake...");
	    $this->log($buffer);
	    list($resource, $host, $origin, $key) = $this->getHeaders($buffer);
	    
	    //websocket version 13
	    $acceptKey = base64_encode(sha1($key . '258EAFA5-E914-47DA-95CA-C5AB0DC85B11', true));//固定的升级key的算法
	    
	    $this->log("Handshaking...");
	    $upgrade  = "HTTP/1.1 101 Switching Protocol\r\n" .
	          "Upgrade: websocket\r\n" .
	          "Connection: Upgrade\r\n" .
	          "Sec-WebSocket-Accept: " . $acceptKey . "\r\n\r\n";  //必须以两个回车结尾
	    $this->log($upgrade);
	    $sent = socket_write($user->socket, $upgrade, strlen($upgrade));//向socket写入升级数据
	    $user->handshake=true;
	    $this->log("Done handshaking...");
	    return true;
    }

# 二、数据传输 #

## 1、客户端 ##

客户端websocket的API非常容易，直接使用websocket对象的send方法即可。

    ws.send(message);

## 2、服务器端 ##

客户端发送的数据是经过浏览器支持的websocket进行了mask处理的，而根据规定服务器端返回的数据不能进行掩码处理，但是需要按照协议的数据帧规定进行封装后发送。因此服务器需要接收数据必须将接收到的字节流进行解码。

    protected function unwrap($clientSocket, $msg="")
    { 
	    $opcode = ord(substr($msg, 0, 1)) & 0x0F;
	    $payloadlen = ord(substr($msg, 1, 1)) & 0x7F;
	    $ismask = (ord(substr($msg, 1, 1)) & 0x80) >> 7;
	    $maskkey = null;
	    $oridata = null;
	    $decodedata = null;
	    
	    //关闭连接
	    if ($ismask != 1 || $opcode == 0x8)
	    {
	      $this->disconnect($clientSocket);
	      return null;
	    }
	    
	    //获取掩码密钥和原始数据
	    if ($payloadlen <= 125 && $payloadlen >= 0)
	    {
	      $maskkey = substr($msg, 2, 4);
	      $oridata = substr($msg, 6);
	    }
	    else if ($payloadlen == 126)
	    {
	      $maskkey = substr($msg, 4, 4);
	      $oridata = substr($msg, 8);
	    }
	    else if ($payloadlen == 127)
	    {
	      $maskkey = substr($msg, 10, 4);
	      $oridata = substr($msg, 14);
	    }
	    $len = strlen($oridata);
	    for($i = 0; $i < $len; $i++)
	    {
	      $decodedata .= $oridata[$i] ^ $maskkey[$i % 4];
	    }		
	    return $decodedata; 
     }

其中得到掩码和控制帧后需要进行验证，如果掩码不为1直接关闭，如果控制帧为8也直接关闭。后面的原始数据和掩码获取是通过websocket协议的数据帧规范进行的。

# 三、连接关闭 #

## 1、客户端 ##

    ws.close();

## 2、服务器端 ##

需要将维护的用户连接池移除相应的连接用户。

    public function disconnect($clientSocket)
    {
	    $found = null;
	    $n = count($this->users);
	    for($i = 0; $i<$n; $i++)
	    {
	      if($this->users[$i]->socket == $clientSocket)
	      { 
	        $found = $i;
	        break;
	      }
	    }
	    $index = array_search($clientSocket,$this->sockets);
	    
	    if(!is_null($found))
	    { 
	      array_splice($this->users, $found, 1);
	      array_splice($this->sockets, $index, 1); 
	      
	      socket_close($clientSocket);
	      $this->say($clientSocket." DISCONNECTED!");
	    }
    }
# 四.启动服务 #
     public function run()
    {
	    while(true)
	    {
		    $socketArr = $this->sockets;
		    $write = NULL;
		    $except = NULL;
		    socket_select($socketArr, $write, $except, NULL);  //select the socket with message automaticly
		    //if handshake choose the master
		    foreach ($socketArr as $socket)
		    {
			    if ($socket == $this->master)
			    {
			       $client = socket_accept($this->master);
			       if ($client < 0)
				    {
				      $this->log("socket_accept() failed");
				      continue;
				    }
				    else
				    {
				       $this->connect($client);
				      }
			    }
			    else
			    {
			      $this->log("----------New Frame Start-------");
			       $bytes = @socket_recv($socket,$buffer,2048,0);
				    if ($bytes == 0)
				    {
				      $this->disconnect($socket);
				    }
				    else
				    {
				       $user = $this->getUserBySocket($socket);
					    if (!$user->handshake)
					    {
					   		 $this->doHandShake($user, $buffer);
					    }
					    else
					    {
					   		 $this->process($user, $buffer);
					    }
				     }
			    }
		    }
	    }
    }
**重点：**<br/>
一.是while(true)挂起进程，不然执行一次后进程就退出了。<br/>
二.是socket_select和socket_accept函数的使用。<br/>
三.是客户端第一次请求时握手。<br/>
## socket_select ##
 **socket_select ($sockets, $write = NULL, $except = NULL, NULL);**

$sockets可以理解为一个数组，这个数组中存放的是文件描述符。当它有变化（就是有新消息到或者有客户端连接/断开）时，socket_select函数才会返回，继续往下执行。 <br/>
$write是监听是否有客户端写数据，传入NULL是不关心是否有写变化。  <br/>
$except是$sockets里面要被排除的元素，传入NULL是”监听”全部。 <br/> 
最后一个参数是超时时间  <br/>
1. 如果为0：则立即结束  <br/>
1. 如果为n>1: 则最多在n秒后结束，如遇某一个连接有新动态，则提前返回  <br/>
1. 如果为null：如遇某一个连接有新动态，则返回 <br/>
## socket_accept ##
创建，绑定，监听后accept函数将会接受socket要来的连接，一旦有一个连接成功，将会返回一个新的socket资源用以交互，如果是一个多个连接的队列，只会处理第一个，如果没有连接的话，进程将会被阻塞，直到连接上。如果用set_socket_blocking或socket_set_noblock()设置了阻塞，会返回false;返回资源后，将会持续等待连接。