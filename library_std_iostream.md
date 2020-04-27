

## 流

- 连续的字节序列，且支持随机访问，视为流
- iostream 流类
  - istream 输入流类
  - ostream 输出流类

- 流对象
  - cin， istream流对象，标准输入
    - getline函数，从给定的istream读取一行数据存入string对象
    - ">>"运算符，从istream对象读取输入数据
  - cout, ostream流对象，标准输出
  - cerr，ostream流对象，标准错误
    - "<<"运算符，向ostream对象写入输出数据
  - 流对象不能拷贝，以IO操作函数以引用方式传递和返回流对象

- 流状态
  - iostate，条件状态类
    - bad 状态
    - fail 状态
    - eof 状态
    - good 状态
  - 操作流状态的函数
    - 检测状态
      - eof(), failt(), bad(), good(), 标志位置位则返回true，仅当全部没有置位good返回true
      - clear()，复位，将流的状态设置为有效good状态
      - clear(flags), 按给定的flags对应复位/（位集合)
      - setstate(flags)， 置位
      - rdstate(), 返回当前流状态，类型为iostate
  - 一个流发生错误，其上后续的IO操作都会失败，只有当一个流出于无错状态时，才可从它读取数据和写入数据
```cpp
while (cin >> word) { } // 在使用一个流前检查它是否出于良好状态，将刘对象的状态当成条件

auto old_state = cin.rdstate();
cin.clear();
process_input(cin);
cin.setstate(old_state); // 恢复

cin.clear(cin.rdstate() & ~cin.failbit & ~ci.badbit); // 将failbit和badbit复位，保持eofbit不变
```

- 流缓冲区
  - 输出缓冲区
    - 每一个输出流都管理一个缓冲区，用来保存程序写入的数据，操作系统将程序的多个输出操作组合成一个单一的系统级写入操作 (缓冲I/O)
    - 刷新流缓冲区，即将缓冲区中的内容写入文件或设备
      - 自动刷新缓冲区
        - 程序正常结束
        - 输出缓冲区满
        - 输入流关联到输出流，写输入流时
      - 显式刷新缓冲区
        - flush()，直接刷新，不输出额外字符
        - ends()，添加空字符，然后刷新
        - endl()，添加空行符，然后刷新
    - 启用或关闭缓冲I/O方式
      - unibuf，将输出流设置为无缓冲，任何输出都立即刷新
      - nonunibuf，默认带缓冲模式
```cpp
cout << unibuf; // 无缓冲，输出立即刷新
cout << nonunbuf;

// 给流对象指定流缓冲区, 关联流对象和流缓冲对象

```

- 流关联
  - 将一个流与另一个流进行关联，一端写入，数据将被发往与其关联的另一端
  - 每个流同时最多关联到一个流，但多个流可以同时关联到一个ostream
  - 关联输入流和输出流
    - 从输入流读操作都会先刷新关联的输出流
```cpp
cin.tie(&cont);
ostream *old_tie = cin.tie(nullptr); // 解除关联，cin对象不与任何流关联
cin.tie(&cerr); // 读取cin，刷新cerr
cin.tie(old_tie);
```


## 文件流

- 继承自iostream类，用于从给定文件读取数据和向给定文件写入数据
- fstream
  - ifstream
    - 默认文件打开模式为in
  - ofstream
    - 默认文件打开模式为out，即不保留文件的原内容，重头开始写入
    - app模式为，在文件的末尾进行添加，保留原内容
- 特有操作
  - open(file_name, open_mode)
    - 将文件流对象与一个打开的文件关联，可指定文件模式
    - 如果已经关联到一个打开文件，在关闭当前打开文件后，才可以重新打开其他文件
    - 任一时刻，一个文件流对象只能关联到一个文件，但多个文件流对象可以同时打开同一个文件
  - close()，关闭文件，即文件流对象与文件断开关联，当文件流对象析构时，自动调用close
  - is_open()，返回布尔值，指出文件流对象所文联的文件是否成功打开或尚未关闭
- 文件打开模式
  - trunc模式，显式截断文件，不保留文件内容, out模式为隐式截断
  - ate和binary模式，可与其他文件模式组合使用，可用于任何类型的文件流对象
```cpp

#include <fstream>

std::ifstream in(file_name);
std::ofstream out; // 未关联任何文件
out.open(file_name); // 输入输出流打开同一个文件
out.close();
out.open(file_name, std::ofstream::app); // 以追加模式重新打开文件
```


## string流

- 从string对象读取数据和向string对象写入数据，将string当作一个IO流(内存I/O)
- 可用于对从文件读取的行数据的流处理，如提取每个单词
- stringstream
  - istringstream
  - ostringstream
    - 可以当输出流使用，使用str()可以当成string对象使用
- 特有操作
  - str()，返回string对象的拷贝
  - str(s)，将s拷贝到string对象
```cpp
#include <sstream>
#include <fstream>
#include <string>
#include <iostream>

int main() {
    std::ifstream in(file_name);
    std::string line, word;
    while (getline(in, line)) {
        std::istringstream  record(line);
        record >> word;
        std::cout << word << " ";
    }
}

std::ostringstream formatted;
formatted << format(nums);
cout << formmatted.str();
```