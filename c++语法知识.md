本文的主要内容是有关c++的语法上面的知识

###XXX will be initialized after
写了一点c++代码，发现编译器提示XXX will be initialized after这样的警告。出现警告的代码类似于下面的情况。

```
#include<iostream>
using namespace std;

class Test
{
private:
    int a;
    int b;
public:
    Test() : b(1), a(2) {}
};

int main()
{
    Test t;
    return 0;
}
```

内部的原因没有去深究，反正将初始化列表按照声明的顺序写也就是改成`Test() : a(2), b(1) {}`就好了。