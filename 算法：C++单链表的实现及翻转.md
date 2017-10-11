### 算法：C++单链表的实现及翻转

撸个简单的单链表顺便学习一下c++

#### int与int&的区别

int为定义变量，int & 为引用。

- 引用在定义的时候必须赋值，否则会编译出错。
- 引用在定义时不会分配内存只是初始化变量的一个别名。
- 引用只能在初始化的时候引用一次，不能转而引用其他变量。

##### 基础引用

```c++
    int x = 1;
    int y = 2;
    int &p = x;

    cout << "x:address->" << &x << " x:value->" << x << endl;
    cout << "p:address->" << &p << " p:value->" << p << endl;

    p = y;

    cout << "x:address->" << &x << " x:value->" << x << endl;
    cout << "p:address->" << &p << " p:value->" << p << endl;
```

输出结果：

```c++
x:address->0x7fff59683988 x:value->1
p:address->0x7fff59683988 p:value->1
x:address->0x7fff59683988 x:value->2
p:address->0x7fff59683988 p:value->2
```



##### 引用传递

如果将`int&`作为函数参数，则表明此处是引用传递，形参是实参的一个别名。

```c++
    int x = 1;
    int y = 2;
    cout << "x:address->" << &x << " x:value->" << x << endl;
    cout << "y:address->" << &y << " y:value->" << y << endl;
    cout << "---调用func---" << endl;
    func(x, y);
    cout << "x:address->" << &x << " x:value->" << x << endl;
    cout << "y:address->" << &y << " y:value->" << y << endl;
    
    void func(int &p, int q) {
    int t;
    t = p;
    p = q;
    q = t;
}
```

输出结果：

```c++
x:address->0x7fff54ea2978 x:value->1
y:address->0x7fff54ea2974 y:value->2
---调用func---
x:address->0x7fff54ea2978 x:value->2
y:address->0x7fff54ea2974 y:value->2
```



##### 不禁陷入深思，指针和引用有啥区别？

1. 指针是个实体，sizeof指针得到的是对象地址的大小。而引用只是别名，sizeof引用得到的是所指向的变量的大小
2. 指针和引用自增自减运算意义不同
3. 引用必须指向有效的变量，指针可以为NULL
4. 引用不可变，指针可变
5. 使用引用时无需解引用（*），而指针需要解引用
6. 从内存分配上看，程序要为指针分配内存区域，而引用则不需要



### 接下来看单链表的实现和翻转



```c++
#ifndef SINGLYLINKEDLIST_LINKEDLIST_H
#define SINGLYLINKEDLIST_LINKEDLIST_H

#include <iostream>

using namespace std;

class LinkedList {
public:
    LinkedList();

    virtual ~LinkedList();

    void create();

    void insert(const int &d);

    void del(const int &d);

    void print();

    void revers();

private:
    struct Node {
        int data;
        Node *next;

        Node(const int &d) : data(d), next(NULL) {}
    };

    Node *head;

    void clear() {
        Node *p = head;
        while (p->next != NULL) {
            Node *q = p->next;
            delete p;
            p = q;
        }
    }
};


#endif //SINGLYLINKEDLIST_LINKEDLIST_H
```



```c++
LinkedList::LinkedList() {
    create();
}

LinkedList::~LinkedList() {
    clear();
}

void LinkedList::create() {
    head = new Node(0);
}

void LinkedList::insert(const int &d) {
    Node *p = new Node(d);
    p->next = head->next;
    head->next = p;
}

void LinkedList::del(const int &d) {
    Node *p = head;
    while (p->next != NULL) {
        if (p->next->data == d) {
            p->next = p->next->next;
            break;
        }
        p = p->next;
    }

}

void LinkedList::print() {
    for (Node *p = head->next; p; p = p->next) {
        cout << p->data << endl;
    }
}

void LinkedList::revers() {
    Node *p = head->next;
    while (p->next != NULL) {
        Node *q = p->next;
        p->next = p->next->next;
        q->next = head->next;
        head->next = q;
    }
}
```



