---
layout: post
title:  右值引用
---

## 概念

### 右值

首先，右值一般是指不能被改变的值。

### 右值引用语法符号

左值的语法符号为`T& t`，右值的语法符号是`T&& t`。下面的函数定义接受的参数就是右值：

    void process_value(int& i) { 
      std::cout << "LValue processed: " << i << std::endl; 
     }  

     void process_value(int&& i) { 
      std::cout << "RValue processed: " << i << std::endl; 
     }  

     void forward_value(int&& i) { 
      process_value(i); // i在这里又变成了一个左值。
     }  

     int main() { 
      int a = 0; 
      process_value(a); 
      process_value(1); 
      forward_value(2); 
     }

运行结果 :

     LValue processed: 0 
     RValue processed: 1 
     LValue processed: 2

### 转移语义

`转移语义`是和`拷贝语义`相对应的。转移语义可以将资源（堆等）从一个对象转移到另一个对象，从而减少不必要的临时对象的创建、拷贝、销毁，提高性能。

以下面的简单的String例子来说：

     class MyString { 
     private: 
      char* _data; 
      size_t   _len; 
      void _init_data(const char *s) { 
        _data = new char[_len+1]; 
        memcpy(_data, s, _len); 
        _data[_len] = '\0'; 
      } 
     public: 
      MyString() { 
        _data = NULL; 
        _len = 0; 
      }     

      MyString(const char* p) { 
        _len = strlen (p); 
        _init_data(p); 
      }     

      MyString(const MyString& str) { 
        _len = str._len; 
        _init_data(str._data); 
        std::cout << "Copy Constructor is called! source: " << str._data << std::endl; 
      }     

      MyString& operator=(const MyString& str) { 
        if (this != &str) { 
          _len = str._len; 
          _init_data(str._data); 
        } 
        std::cout << "Copy Assignment is called! source: " << str._data << std::endl; 
        return *this; 
      }     

      virtual ~MyString() { 
        if (_data) free(_data); 
      } 
     };     

     int main() { 
      MyString a; 
      a = MyString("Hello"); // @1
      std::vector<MyString> vec; 
      vec.push_back(MyString("World")); //@2 
     }

运行结果：

    Copy Assignment is called! source: Hello 
    Copy Constructor is called! source: World

运行结果说明，在@1和@2两处地方，发生了资源的拷贝，而这种拷贝是没有必要的（因为拷贝的源是右值，这个拷贝结束后，那个右值就会被析构了）。所以这个时候如果定义了右值拷贝构造/右值赋值操作符，且在其中直接复用源右值的内存，就可以减少开销：

    MyString(MyString&& str) { 
        std::cout << "Move Constructor is called! source: " << str._data << std::endl; 
        _len = str._len; 
        _data = str._data; 
        str._len = 0; 
        str._data = NULL; 
    }

    MyString& operator=(MyString&& str) { 
        std::cout << "Move Assignment is called! source: " << str._data << std::endl; 
        if (this != &str) { 
            _len = str._len; 
            _data = str._data; 
            str._len = 0; 
            str._data = NULL; 
        } 
        return *this; 
    }

重新运行结果：

    Move Assignment is called! source: Hello 
    Move Constructor is called! source: World

### std::move

这个操作符将左值转化为右值。下面是使用这个操作符实现的swap：

    template <class T> swap(T& a, T& b) 
    { 
        T tmp(std::move(a)); // move a to tmp 
        a = std::move(b);    // move b to a 
        b = std::move(tmp);  // move tmp to b 
    }

### Perfect Forwarding

参见：[精确传递](http://www.ibm.com/developerworks/cn/aix/library/1307_lisl_c11/)

## 示例

## 总结