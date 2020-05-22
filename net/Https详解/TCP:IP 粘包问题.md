场景

在TCP通信的时候，连续多次发送数据，经常会遇到一些“奇怪”的问题，具体代码如下：

服务器端：
```c
//
//  ServerSocket.m
//  TCP粘包
//
//  Created by qinmin on 2017/11/5.
//  Copyright © 2017年 qinmin. All rights reserved.
//

#import "ServerSocket.h"
#import <sys/socket.h>
#import <netinet/in.h>
#import <arpa/inet.h>

#define kMAXLINE                 4096

@interface ServerSocket()
{
    int         _socketHandle;
    BOOL        _isFinish;
    NSInteger   _serverPort;
}
@end

@implementation ServerSocket

#pragma mark - LiferCycle
- (instancetype)initWithPort:(NSInteger)port
{
    if (self = [super init]) {
        _isFinish = YES;
        _serverPort = port;
    }
    
    return self;
}

#pragma mark - PublicMethod
- (void)startServer
{
    if (!_isFinish) {
        return;
    }
    
    _isFinish = NO;
    if ([NSThread isMainThread]) {
        dispatch_async(dispatch_get_global_queue(0, 0), ^{
            [self createServerSocket];
        });
    }else {
        [self createServerSocket];
    }
}

- (void)stopServer
{
    if (_isFinish) {
        return;
    }
    
    _isFinish = YES;
    close(_socketHandle);
    
    if (_serverDidStopBlock) {
        _serverDidStopBlock();
    }
}

#pragma mark - PrivateMethod
- (void)createServerSocket
{
    int connnectHandle;
    struct sockaddr_in servaddr;
    char buff[kMAXLINE];
    
    if ((_socketHandle = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        NSLog(@"socket error %s", strerror(errno));
        return;
    }
    
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(_serverPort);
    
    if(bind(_socketHandle, (struct sockaddr*)&servaddr, sizeof(servaddr)) == -1) {
        NSLog(@"bind error: %s",strerror(errno));
        return;
    }
    
    if(listen(_socketHandle, 10) == -1) {
        NSLog(@"listen error: %s",strerror(errno));
        return;
    }
    
    if (_serverDidStartBlock) {
        _serverDidStartBlock();
    }
    
    // 目前只处理一个socket连接
    if((connnectHandle = accept(_socketHandle, (struct sockaddr*)NULL, NULL)) == -1) {
        NSLog(@"accept socket error: %s",strerror(errno));
        return;
    }
    
    size_t n;
    while (!_isFinish && (n = recv(connnectHandle, buff, kMAXLINE, 0)) > 0) {
        buff[n] = '\0';
//        NSLog(@"recv msg from client: %s\n", buff);
        NSLog(@"recv msg from client length: %ld", n);
    }
    close(connnectHandle);
}

@end

```

客户端：

```c
//
//  ClientSocket.m
//  TCP粘包
//
//  Created by qinmin on 2017/11/5.
//  Copyright © 2017年 qinmin. All rights reserved.
//

#import "ClientSocket.h"
#import <sys/socket.h>
#import <netinet/in.h>
#import <arpa/inet.h>

#define kMAXLINE    4096

static void* clientSocketSendQueueKey;
static void* clientSocketRecvQueueKey;

@interface ClientSocket()
{
    int                 _sockfd;
    NSString            *_serverIP;
    NSInteger           _serverPort;
    dispatch_queue_t    _clientSocketSendQueue;
    dispatch_queue_t    _clientSocketRecvQueue;
    BOOL                _isStop;
}
@end

@implementation ClientSocket

#pragma mark - LiferCycle
- (instancetype)initWithServerIP:(NSString *)IP port:(NSInteger)port
{
    if (self = [super init]) {
        _serverIP = IP;
        _serverPort = port;
        _isStop = YES;
        _clientSocketSendQueue = dispatch_queue_create("client.socket.send.queue", NULL);
        _clientSocketRecvQueue = dispatch_queue_create("client.socket.recv.queue", NULL);
        dispatch_queue_set_specific(_clientSocketSendQueue, &clientSocketSendQueueKey, NULL, NULL);
        dispatch_queue_set_specific(_clientSocketSendQueue, &clientSocketRecvQueueKey, NULL, NULL);
    }
    
    return self;
}

#pragma mark - PublicMethod
- (void)startConnect
{
    if (!_isStop) {
        return;
    }
    
    dispatch_block_t block = ^() {
        _isStop = NO;
        [self createClientSocket];
    };
    
    if (dispatch_queue_get_specific(_clientSocketSendQueue, &clientSocketSendQueueKey)) {
        dispatch_sync(_clientSocketSendQueue, block);
    }else {
        dispatch_async(_clientSocketSendQueue, block);
    }
}

- (void)stopConnect
{
    if (_isStop) {
        return;
    }
    
    dispatch_block_t block = ^() {
        close(_sockfd);
        _isStop = YES;
    };
    
    if (dispatch_queue_get_specific(_clientSocketSendQueue, &clientSocketSendQueueKey)) {
        dispatch_sync(_clientSocketSendQueue, block);
    }else {
        dispatch_async(_clientSocketSendQueue, block);
    }
}

- (void)sendData:(NSData *)data
{
    dispatch_block_t block = ^() {
        const char *sendLine = data.bytes;
        NSUInteger lineLength = (data.length > kMAXLINE ? kMAXLINE : data.length);
        ssize_t len = 0;
        while (!_isStop && (len = send(_sockfd, sendLine, lineLength, 0)) > 0) {
            sendLine += lineLength;
            NSUInteger left = data.length - lineLength;
            if (left <= 0) {
                break;
            }
            
            lineLength = (left > kMAXLINE ? kMAXLINE : left);
        }
        
        if (len < 0) {
            NSLog(@"send msg error: %s", strerror(errno));
        }
    };
    
    if (dispatch_queue_get_specific(_clientSocketSendQueue, &clientSocketSendQueueKey)) {
        dispatch_sync(_clientSocketSendQueue, block);
    }else {
        dispatch_async(_clientSocketSendQueue, block);
    }
}

- (void)recvData
{
    dispatch_block_t block = ^() {
        char buff[kMAXLINE];
        size_t n = 0;
        while (!_isStop && (n = recv(_sockfd, buff, kMAXLINE, 0)) > 0) {
            buff[n] = '\0';
            // NSLog(@"recv msg from client: %s\n", buff);
            NSLog(@"recv msg from client length: %ld", n);
            
            if (_clientDidRecvDataBlock) {
                _clientDidRecvDataBlock([NSData dataWithBytes:buff length:n]);
            }
        }
    };
    
    if (dispatch_queue_get_specific(_clientSocketRecvQueue, &clientSocketRecvQueueKey)) {
        dispatch_sync(_clientSocketRecvQueue, block);
    }else {
        dispatch_async(_clientSocketRecvQueue, block);
    }
}

#pragma mark - PrivateMethod
- (void)createClientSocket
{
    struct sockaddr_in servaddr;
    
    if((_sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        NSLog(@"socket error: %s", strerror(errno));
        return;
    }
    
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(_serverPort);
    if(inet_pton(AF_INET, _serverIP.UTF8String, &servaddr.sin_addr) <= 0) {
        NSLog(@"inet_pton error");
        return;
    }
    
    if(connect(_sockfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0) {
        NSLog(@"connect error: %s",strerror(errno));
        return;
    }
}

@end
```

数据发送

```c
- (void)viewDidLoad
{
    [super viewDidLoad];

    NSData *data = [NSData dataWithContentsOfFile:[[NSBundle mainBundle] pathForResource:@"demo" ofType:@"txt"]];
    
//    NSLog(@"%ld", data.length);
    
    _client = [[ClientSocket alloc] initWithServerIP:@"127.0.0.1" port:6666];
    _server = [[ServerSocket alloc] initWithPort:6666];
    
    __weak typeof(self) wself = self;
    [_server setServerDidStartBlock:^{
        __strong typeof(self) sself = wself;
        [sself.client startConnect];
        [sself.client sendData:data];
        [sself.client sendData:data];
        [sself.client sendData:data];
        [sself.client sendData:data];
    }];
    
    [_server startServer];
}
```
可以看出只有第一次是完整的数据大小，其它每次接收的数据都不是待发送数据的真实长度。

## 粘包问题

在做TCP通信的时候，如果需要在一条连接上连续发送不同结构的数据时，可能遇到其中的某些包完整，某些包不完整，也可能遇到某些包包含多个数据。这就是典型的TCP粘包现象。TCP粘包现象是指在使用TCP通信的时候，一个完成的消息可能会被TCP拆分成多个包进行发送，也有可能把多个小的包封装成一个大的数据包进行发送。

## 提高网络利用率

**Nagle 算法**

TCP 中为了提高网络的利用率，经常使用一个叫做Nagle的算法。该算法是指发送端即使还有应发送的数据，但如果这部分数据很少的话，则进行延迟发送的一种处理机制，也就是仅在下列任意一种条件下才能发送数据，如果两条件都不满足，那么暂时等待一段时间以后再进行数据发送。

1、已发送的数据都已经收到确认应客时。<br>
2、可以发送最大段长度(MSS) 的数据时。<br>

在使用 TCP 协议发送数据的时候，即使只发送一个字节，但是数据还是需要封装成TCP/IP包来发送。因此，最少需要加入一个 20 字节的 TCP 首部，20 字节的 IP 首部，这样发送的流量其实是数据的40倍左右。Nagle 算法就是为了解决频繁发送小包所导致的流量浪费和网络阻塞问题。

## 延迟确认应答

接收数据的主机如果每次都立刺回复确认应答的话，可能会返回一个较小的窗口。那是因为刚接收完数据，缓冲区已满。当某个接收端收到这个小窗口的通知以后，会以它为上限发送数据，从而又降低了网络的利用率。为此，引入了一个方法，那就是收到数据以后并不立即返回确认应答，而是延迟一段时间的机制，尝试减少接收方所发送的 ack 数量。<br>
1、在没有收到2x最大段长度的数据为止不做确认应答；<br>
2、其他情况下，延迟发送确认应答；

## 粘包原因

1、由Nagle算法造成的发送端的粘包。发送端需要等缓冲区满才发送数据出去，这就有可能把多个小的包封装成一个大的数据包进行发送。<br>

2、接收端接收不及时造成的接收端粘包。TCP会把接收到的数据存在自己的缓冲区中,然后通知应用层取数据。当应用层不能及时的把TCP的数据取出来,就会造成缓冲区中存放了多个MSS数据。

## 解决办法

1、每次发送数据，就与对方建立连接，然后双方发送完一段数据后，就关闭连接。这种算法的局限在于每次都要进行三次握手四次挥手，既浪费流量，又使数据传输延时性增大，socket不能很好的复用。<br>

2、特殊切割符来分割包。这种方式必须严格要求包体中不会出现该特殊字符，因此，需要控制使用范围。<br>

3、每个包都是固定长度。这种方式会造成包的体积很难确定，浪费流量等问题。<br>

4、发送端使用了TCP强制数据立即传送的操作指令push。可能引发频繁发送小包所导致的流量浪费和网络阻塞问题。<br>

5、自定义协议，支持可变长度的包。可定制性强，对编码要求增加。

