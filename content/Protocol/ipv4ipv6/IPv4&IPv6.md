在计算机网络中，IPv4（Internet Protocol version 4）和IPv6（Internet Protocol version 6）是用于标识和定位网络中设备的协议。它们在地址长度上有所不同：

1. **IPv4：**
   - IPv4地址长度为32位，通常以四个十进制数表示，每个数范围在0到255之间，以点分十进制（dotted-decimal）表示法呈现，例如：192.168.0.1。

2. **IPv6：**
   - IPv6地址长度为128位，通常以八组十六进制数表示，每组数有四个十六进制数字，以冒号分隔，例如：2001:0db8:85a3:0000:0000:8a2e:0370:7334。

由于IPv6地址长度更长，它提供了更多的地址空间，以解决IPv4地址空间不足的问题。IPv6的广泛采用也有助于支持更多的互联设备，并提供更好的网络性能和安全性。

``` C++
#define IN6ADDR_ANY_INIT { { { 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 } } }
#define IN6ADDR_LOOPBACK_INIT { { { 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,1 } } }

#define INET_ADDRSTRLEN 16
#define INET6_ADDRSTRLEN 46

```

``` C++
#include <iostream>
#include <arpa/inet.h>
using namespace std;
// #define	INADDR_NONE		((in_addr_t) 0xffffffff)
int main() {
    const char *ipAddress = "";

    // 将点分十进制的 IPv4 地址转换为二进制形式
    in_addr_t binaryAddress = inet_addr(ipAddress);

    if (binaryAddress == INADDR_NONE) {
        cerr << "Invalid IP address " <<binaryAddress<< endl;
    } else {
        cout << "Binary IP address: " << binaryAddress << endl;
    }
   
    char ipv4_addr[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &binaryAddress, ipv4_addr, INET_ADDRSTRLEN);
    printf("IPv4 Address: %s\n", ipv4_addr);
    
    return 0;
}
 
// Invalid IP address 4294967295
// IPv4 Address: 255.255.255.255
```

``` C++
#include <iostream>
#include <arpa/inet.h>
using namespace std;
int main() {
    const char* ipv6Address = "";
    struct in6_addr result;
    if (inet_pton(AF_INET6, ipv6Address, &result) == 1) {
        cout << "IPV6 address is valid" << endl;
    }
    else {
        cout << "invalid IPv6 addrss" << endl;
    }

    struct in6_addr ipv6;
    char ipv6_addr[INET6_ADDRSTRLEN];
    inet_ntop(AF_INET6, &result, ipv6_addr, INET6_ADDRSTRLEN);
    printf("IPv6 Address: %s\n", ipv6_addr);
    return 0;
}

// invalid IPv6 addrss
// IPv6 Address: ::
```

