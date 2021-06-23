# CS144 lab0 详解


<!--more-->

> 实验指导说明：[https://cs144.github.io/assignments/lab0.pdf](https://cs144.github.io/assignments/lab0.pdf)

## `build prerequisite`

> 本人实验环境：阿里云服务器 Ubuntu20.04，其余环境的配置请参考官方提供的[实验环境配置指导](https://web.stanford.edu/class/cs144/vm_howto/)。

实验前需要下载以下软件：
+ g++ (version > 8)
+ clang-tidy (version = 6或7) 
+ clang-format (version = 6或7)
> 其中本人下载的为 clang-tidy-7 和 clang-format-7。
+ cmake (version >= 3)
+ libpcap-dev
+ git
+ iptables
+ mininet (version >= 2.2.0)
+ tcpdump
+ telnet
+ wireshark
+ socat
+ netcat-openbsd
+ GNU coreutils
+ bash
+ doxygen
+ graphviz

可以通过`apt list --installed | grep <software_name>`查看缺少哪些软件，然后再运行`sudo 
apt install <software_name>`安装即可。

## `Fetch a Web page`

使用浏览器访问[http://cs144.keithw.org/hello](http://cs144.keithw.org/hello)，发现只有一行文字：`Hello, CS144!`。在命令行中输入`telnet cs144.keithw.org http`，得到结果如下图：

![cs144-lab0-1](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab0-1.4vs760bsep00.png)

随后依次输入：
+ `GET /hello HTTP/1.1`并敲入回车。这一行指令表示向服务器发送在`HTTP/1.1`下获取路径为`/hello`的数据的请求。
+ `Host: cs144.keithw.org`并敲入回车。这一行指令告诉服务器`host`地址。
+ `Connection: close`并敲入回车。这一行指令告诉服务器客户的请求已经发送完毕，服务器只要发送完数据就可以断开连接。
+ 注意此时还需要敲一下回车。

如果一切都顺利的话，应该能看到下图中的结果：

![cs144-lab0-2](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab0-2.fjvrhnv2vf4.png)

## `Send yourself an email`

这个由于没有官方账号，用QQ邮箱测试也一直连接不上，故跳过。

## `Listening and connecting`

打开两个终端，其中一个终端输入`netcat -lnvp 9090`创建服务器开启监听，如下图：

![cs144-lab0-3](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab0-3.51bsa5rx3u00.png)

另一个终端输入`telnet localhost 9090`连接服务器，如下图：

![cs144-lab0-4](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab0-4.1rb2aqcl19eo.png)

同时另一个终端显示`Connection received on 127.0.0.1 50436`，表示已经连接成功。然后在任意一个窗口输入内容，可以发现另一个窗口显示同样的内容，说明数据成功地在服务器和客户端之间传输。

## `fetching and building the starter code`

~~~shell
git clone https://github.com/cs144/sponge
cd sponge
mkdir build
cd build
# 由于本人前面安装的为 clang-tidy-7 和 clang-format-7，故此时需要进行相应设置
CLANG_TIDY=clang-tidy-7 CLANG_FORMAT=clang-foramt-7 cmake ..
make
~~~

## `Writing webget`

> 在正式敲代码之前务必仔细阅读实验指导中对C++的要求和说明以及`libsponge/util/file descriptor.hh`，`libsponge/util/socket.hh`，`libsponge/util/address.hh`。

本实验的目的就是完成`webget.cc`中的`get_URL`函数，实现对网页数据的获取。首先阅读下`doctests/socket_example_2.cc`中是如何创建`TCPSocket`以及建立连接，发送并读取数据。再了解了相关API后，我们只需要依次进行以下操作：
+ 创建一个`TCPSocket`并与服务器建立连接。
+ 向服务器发送请求，格式参照前面`fetch a web page`部分，注意在`HTTP`中每行的结尾应该为`\r\n`。
+ 发送完请求后，客户端应该关闭`TCPSocket`的写功能，对应前面的`Connection: close`，告诉服务器请求已经发送完毕，服务器只要回复完数据后就可以立刻断开连接。
+ 循环读取从服务器发送过来的信息，直到遇到`EOF (end of file)`。
+ 最后记得需要关闭前面创建的`TCPSocket`。

最终代码实现如下：

~~~CPP
void get_URL(const string &host, const string &path) {
    // Your code here.

    // You will need to connect to the "http" service on
    // the computer whose name is in the "host" string,
    // then request the URL path given in the "path" string.

    // Create a TCP socket and connect to the "http" service on host
    TCPSocket socket;
    socket.connect(Address(host, "http"));

    // Then you'll need to print out everything the server sends back,
    // (not just one call to read() -- everything) until you reach
    // the "eof" (end of file).

    // Send HTTP request to the server
    socket.write("GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\n\r\n");
    // Tell the server that it's not going to send any more requests
    socket.shutdown(SHUT_WR);

    // Print out everything the server sends back
    while (!socket.eof()) {
        cout << socket.read();
    }

    // Close the socket
    socket.close();

    // cerr << "Function called: get_URL(" << host << ", " << path << ").\n";
    // cerr << "Warning: get_URL() has not been implemented yet.\n";
}
~~~

在`build`目录下执行`make`，通过运行`./apps/webget cs144.keithw.org /hello`和`make check_webget`来验证代码正确性。


## `An in-memory reliable byte stream`

简单来说，就是要实现一个用来支持给**输入端写入数据**和**输出端读取数据**的字节流缓冲区。由于缓存区的数据需要满足先进先出的原则，这里我采用的是`deque`(也可以使用`queue`)，同时根据提示以及类中函数说明，我们还需要定义几个变量：缓存区的容量、输入端是否写入完毕、以及写入和读取的字节数。变量定义代码如下：

~~~CPP
    size_t _capacity = 0;
    std::deque<char> _buf{};
    bool _end_input = false;
    size_t _bytes_written = 0;
    size_t _bytes_read = 0;
~~~

函数实现代码如下：

~~~CPP
ByteStream::ByteStream(const size_t capacity) : _capacity(capacity) {}

size_t ByteStream::write(const string &data) {
    if (remaining_capacity() == 0) {
        return 0;
    }

    size_t len2write = min(remaining_capacity(), data.size());
    for (size_t i = 0; i < len2write; i++) {
        _buf.push_back(data[i]);
    }
    _bytes_written += len2write;
    return len2write;
}

//! \param[in] len bytes will be copied from the output side of the buffer
string ByteStream::peek_output(const size_t len) const {
    size_t len2peek = min(buffer_size(), len);
    return string().assign(_buf.begin(), _buf.begin() + len2peek);
}

//! \param[in] len bytes will be removed from the output side of the buffer
void ByteStream::pop_output(const size_t len) {
    size_t len2pop = min(buffer_size(), len);
    for (size_t i = 0; i < len2pop; i++) {
        _buf.pop_front();
    }
    _bytes_read += len2pop;
}

//! Read (i.e., copy and then pop) the next "len" bytes of the stream
//! \param[in] len bytes will be popped and returned
//! \returns a string
std::string ByteStream::read(const size_t len) {
    string ret = peek_output(len);
    pop_output(len);
    return ret;
}

void ByteStream::end_input() { _end_input = true; }

bool ByteStream::input_ended() const { return _end_input; }

size_t ByteStream::buffer_size() const { return _buf.size(); }

bool ByteStream::buffer_empty() const { return buffer_size() == 0; }

bool ByteStream::eof() const { return _end_input && (buffer_size() == 0); }

size_t ByteStream::bytes_written() const { return _bytes_written; }

size_t ByteStream::bytes_read() const { return _bytes_read; }

size_t ByteStream::remaining_capacity() const { return _capacity - _buf.size(); }
~~~

在`build`目录下执行`make`，通过运行`make check_lab0`验证代码正确性。**记得最后运行`make format`将代码格式化。**
