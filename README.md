# DNSTest
解析dns的三种方法
DNS解析(网络切换的问题解决)
上次提到过由于电信的问题需要自己手动去解析dns，这里介绍的是如何拦截
每一个请求做解析，但是没有说具体的解析方法，下面简单的记录一下：

res_query方法

int res_query(char *domain_name, int class, int type, char *answer_buffer, int answer_buffer_length)
1
int res_query(char *domain_name, int class, int type, char *answer_buffer, int answer_buffer_length)
这是比较常见的系统调用，使用该方法的时候需要在Xcode中添加libresolv.dylib，然后包含resolv.h头文件即可，具体代码如下：


    unsigned char res[512];
    int nBytesRead = 0;

    //调用系统方法
    nBytesRead = res_query("www.baidu.com", ns_c_in, ns_t_a, res, sizeof(res));

    ns_msg handle;
    ns_initparse(res, nBytesRead, &handle);

    NSMutableArray *ipList = nil;
    int msg_count = ns_msg_count(handle, ns_s_an);
    if (msg_count > 0) {
        ipList = [[NSMutableArray alloc] initWithCapacity:msg_count];
        for(int rrnum = 0; rrnum 
1
2
3
4
5
6
7
8
9
10
11
12
13
14
    unsigned char res[512];
    int nBytesRead = 0;
 
    //调用系统方法
    nBytesRead = res_query("www.baidu.com", ns_c_in, ns_t_a, res, sizeof(res));
 
    ns_msg handle;
    ns_initparse(res, nBytesRead, &handle);
 
    NSMutableArray *ipList = nil;
    int msg_count = ns_msg_count(handle, ns_s_an);
    if (msg_count > 0) {
        ipList = [[NSMutableArray alloc] initWithCapacity:msg_count];
        for(int rrnum = 0; rrnum 
然而该方法有一个问题，在网络从2/3G和WI-FI之间切换的时候，该方法经常不能正常工作，或者需要等待较长的时间，

gethostbyname

struct hostent *gethostbyname(const char *hostName);
1
struct hostent *gethostbyname(const char *hostName);
具体代码如下：


    struct hostent *host = gethostbyname("www.google.com.hk");

    struct in_addr **list = (struct in_addr **)host->h_addr_list;

    //获取IP地址
    NSString *ip= [NSString stringWithCString:inet_ntoa(*list[0]) encoding:NSUTF8StringEncoding];

    NSLog(@"ip address is : %@",ip);
1
2
3
4
5
6
7
8
    struct hostent *host = gethostbyname("www.google.com.hk");
 
    struct in_addr **list = (struct in_addr **)host->h_addr_list;
 
    //获取IP地址
    NSString *ip= [NSString stringWithCString:inet_ntoa(*list[0]) encoding:NSUTF8StringEncoding];
 
    NSLog(@"ip address is : %@",ip);
该方法在碰到切换网络的时候，出现失败的情况比上面的方法好多了，但偶尔也还是会出现，是时候采用苹果自己的方法了。

CFHostStartInfoResolution

Boolean CFHostStartInfoResolution (CFHostRef theHost, CFHostInfoType info, CFStreamError *error);
1
Boolean CFHostStartInfoResolution (CFHostRef theHost, CFHostInfoType info, CFStreamError *error);
具体实现方法如下:


    Boolean result,bResolved;
    CFHostRef hostRef;
    CFArrayRef addresses = NULL;

    CFStringRef hostNameRef = CFStringCreateWithCString(kCFAllocatorDefault, "www.google.com.hk", kCFStringEncodingASCII);

    hostRef = CFHostCreateWithName(kCFAllocatorDefault, hostNameRef);
    if (hostRef) {
        result = CFHostStartInfoResolution(hostRef, kCFHostAddresses, NULL);
        if (result == TRUE) {
            addresses = CFHostGetAddressing(hostRef, &result);
        }
    }
    bResolved = result == TRUE ? true : false;

    if(bResolved)
    {
        struct sockaddr_in* remoteAddr;
        for(int i = 0; i sin_addr));
            }
        }
    }
    CFRelease(hostNameRef);
    CFRelease(hostRef);
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
    Boolean result,bResolved;
    CFHostRef hostRef;
    CFArrayRef addresses = NULL;
 
    CFStringRef hostNameRef = CFStringCreateWithCString(kCFAllocatorDefault, "www.google.com.hk", kCFStringEncodingASCII);
 
    hostRef = CFHostCreateWithName(kCFAllocatorDefault, hostNameRef);
    if (hostRef) {
        result = CFHostStartInfoResolution(hostRef, kCFHostAddresses, NULL);
        if (result == TRUE) {
            addresses = CFHostGetAddressing(hostRef, &result);
        }
    }
    bResolved = result == TRUE ? true : false;
 
    if(bResolved)
    {
        struct sockaddr_in* remoteAddr;
        for(int i = 0; i sin_addr));
            }
        }
    }
    CFRelease(hostNameRef);
    CFRelease(hostRef);
具体的demo可以到这里看看
