# CS144 lab1 详解


<!--more-->

> 实验指导说明：[https://cs144.github.io/assignments/lab1.pdf](https://cs144.github.io/assignments/lab1.pdf)

本实验的目的就是将接收方获取到的`TCP段(substrings/segments)`重组成有序的，不重复的，从而被我们在lab0中实现的`byte_stream`读取。虽然看起来很简单，但是其中细节还是不少，写了两天才弄懂实验所有的细节。。。

## 实验中的细节点

![cs144-lab1-2](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab1-2.51pv6at8ro40.png)

需要注意，虽然有的`substring`完成了重组(上图绿色部分)，但是只要没有被`byte_stream`读取，它还属于`stream_assembler`的一部分，因此它的大小也是算在`capacity`中的一部分的。每接收到一个新的`substring`时，我们都要检查下哪些部分可以进行重组(可能包括之前未被重组的部分，上图红色部分)并且被`byte_stream`写入。

此外每次新来一个`substring`的时候需要根据`index`分情况讨论(具体见代码部分)；另外如果数据有一部分被舍弃掉的话，那么传进来的`eof`是无效的，这一点应该挺好理解的。总体来说，如果能弄懂下面这个测试用例的话，应该对本实验理解的差不多了。
~~~CPP
ReassemblerTestHarness test{8};

test.execute(SubmitSegment{"abc", 0});
test.execute(BytesAssembled(3));
test.execute(NotAtEof{});

test.execute(SubmitSegment{"ghX", 6}.with_eof(true));
test.execute(BytesAssembled(3));
test.execute(NotAtEof{});

test.execute(SubmitSegment{"cdefg", 2});
test.execute(BytesAssembled(8));
test.execute(BytesAvailable{"abcdefgh"});
test.execute(NotAtEof{});
~~~

## `map`实现

最开始我用的`map`来保存`substring`的，代码如下：

`stream_reassembler.hh`
~~~CPP
    std::map<size_t, char> index2byte{};
    bool _eof{false};
    size_t _head_index = 0;
    size_t _unassembled_bytes = 0;

    ByteStream _output;  //!< The reassembled in-order byte stream
    size_t _capacity;    //!< The maximum number of bytes
~~~

`stream_reassembler.cc`
~~~CPP
#include "stream_reassembler.hh"

// Dummy implementation of a stream reassembler.

// For Lab 1, please replace with a real implementation that passes the
// automated checks run by `make check_lab1`.

// You will need to add private members to the class declaration in `stream_reassembler.hh`

// template <typename... Targs>
// void DUMMY_CODE(Targs &&... /* unused */) {}

using namespace std;

StreamReassembler::StreamReassembler(const size_t capacity) : _output(capacity), _capacity(capacity) {}

//! \details This function accepts a substring (aka a segment) of bytes,
//! possibly out-of-order, from the logical stream, and assembles any newly
//! contiguous substrings and writes them into the output stream in order.
void StreamReassembler::push_substring(const string &data, const size_t index, const bool eof) {
    // DUMMY_CODE(data, index, eof);
    if (index >= _head_index + _output.remaining_capacity()) {
        return;
    }

    if (index + data.size() < _head_index) {
        if (eof && empty()) {
            _output.end_input();
        }
        return;
    }

    bool flag = true;
    if (index + data.size() > _head_index + _output.remaining_capacity()) {
        flag = false;
    }

    // size_t count = 0;
    for (size_t i = index; i < index + data.size() && i < _head_index + _output.remaining_capacity(); i++) {
        if (i >= _head_index && index2byte.find(i) == index2byte.end()) {
            index2byte[i] = data[i - index];
            // count++;
            _unassembled_bytes++;
        }
    }

    string str = "";
    for (size_t i = _head_index; index2byte.find(i) != index2byte.end(); i++) {
        str += index2byte[i];
        index2byte.erase(i);
        _head_index++;
        _unassembled_bytes--;
    }

    _output.write(str);

    if (flag && eof) {
        _eof = true;
    }

    if (_eof && empty()) {
        _output.end_input();
    }

}

size_t StreamReassembler::unassembled_bytes() const { return _unassembled_bytes; }

bool StreamReassembler::empty() const { return unassembled_bytes() == 0; }
~~~

测试结果(见下图)并不是太理想，虽然实验指导中说了不用过分追求效率，但是也给出了一个预期结果，就是每个测试用时不超过半秒。

![cs144-lab1-1](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab1-1.5nkowm19fds0.png)


## `deque`实现

`map`实现中其实并没用到`map`的很多性质，只用来作为存放`bytes`的容器并且判断某个索引位置是否有数据。这样的话完全就可以换成数组之类的容器，同时用一个`bitmap`来标志某个位置是否有数据即可，最后我采用的是`deque`数据结构，其实只要满足先进先出的数据结构都可以。最后代码如下：

`stream_reassembler.hh`
~~~CPP
    std::deque<char> _buf;
    std::deque<bool> _bitmap;
    bool _eof{false};
    size_t _head_index = 0;
    size_t _unassembled_bytes = 0;

    ByteStream _output;  //!< The reassembled in-order byte stream
    size_t _capacity;    //!< The maximum number of bytes
~~~

`stream_reassembler.cc`
~~~CPP
#include "stream_reassembler.hh"

// Dummy implementation of a stream reassembler.

// For Lab 1, please replace with a real implementation that passes the
// automated checks run by `make check_lab1`.

// You will need to add private members to the class declaration in `stream_reassembler.hh`

// template <typename... Targs>
// void DUMMY_CODE(Targs &&... /* unused */) {}

using namespace std;

StreamReassembler::StreamReassembler(const size_t capacity)
    : _buf(capacity, '\0'), _bitmap(capacity, false), _output(capacity), _capacity(capacity) {}

//! \details This function accepts a substring (aka a segment) of bytes,
//! possibly out-of-order, from the logical stream, and assembles any newly
//! contiguous substrings and writes them into the output stream in order.
void StreamReassembler::push_substring(const string &data, const size_t index, const bool eof) {
    // DUMMY_CODE(data, index, eof);
    if (index >= _head_index + _output.remaining_capacity()) {
        return;
    }

    if (index + data.size() < _head_index) {
        if (eof) {
            _eof = true;
        }
    } else {
        if (index + data.size() <= _head_index + _output.remaining_capacity()) {
            if (eof) {
                _eof = true;
            }
        }

        for (size_t i = index; i < _head_index + _output.remaining_capacity() && i < index + data.size(); i++) {
            if (i >= _head_index && !_bitmap[i - _head_index]) {
                _buf[i - _head_index] = data[i - index];
                _bitmap[i - _head_index] = true;
                _unassembled_bytes++;
            }
        }

        string str = "";
        while (_bitmap.front()) {
            str += _buf.front();
            _buf.pop_front();
            _buf.push_back('\0');
            _bitmap.pop_front();
            _bitmap.push_back(false);
        }

        size_t len = str.size();
        if (len > 0) {
            _unassembled_bytes -= len;
            _head_index += len;
            _output.write(str);
        }
    }

    if (_eof && empty()) {
        _output.end_input();
    }
}

size_t StreamReassembler::unassembled_bytes() const { return _unassembled_bytes; }

bool StreamReassembler::empty() const { return unassembled_bytes() == 0; }
~~~

最后测试结果(见下图)也基本达到了预期结果，所有测试的用时加起来也才到一秒钟。

![cs144-lab1-3](https://cdn.jsdelivr.net/gh/PeiLeiScott/image-hosting@master/cs144-lab1-3.32g6thcyyhc0.png)
