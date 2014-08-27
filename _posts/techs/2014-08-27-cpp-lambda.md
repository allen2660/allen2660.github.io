---
layout: post
title:  C++中的lambda表达式
---

## lambda是什么

许多语言支持匿名函数的概念。这些函数有函数体、但是没有函数名称。lambda隐式定义函数对象类并构造该类类型的函数对象。

## 示例

### 排序的例子

    #include <algorithm>
    #include <iostream>
    #include <vector>   

    using namespace std;    

    void abssort(vector<float> &numbers) {
        std::sort(numbers.begin(), numbers.end(),
                [](float a, float b) {
                    return std::abs(a) < std::abs(b);
                });
    }   

    int main(){
        vector<float> numbers = {1,-1, -0.3, -101.1, 99};
        abssort(numbers);
        for(auto x:numbers){
            cout <<x << " ";
        }
        cout<< endl;
    }

运行结果：
    
    -0.3 1 -1 99 -101.1

### 保持状态的例子

    void lambda_even() {
    vector<int> v;
    for (int i = 0; i < 10; ++i) {
      v.push_back(i);
    }

    // Count the number of even numbers in the vector by 
    // using the for_each function and a lambda.
    int evenCount = 0;
    for_each(v.begin(), v.end(),[&evenCount] (int n) {
       cout << n;
       if (n % 2 == 0) {
          cout << " is even " << endl;
          ++evenCount;
       } else {
          cout << " is odd " << endl;
       }
    });  

    // Print the count of even numbers to the console.
    cout << "There are " << evenCount 
         << " even numbers in the vector." << endl;
    }

运行结果：

    0 is even 
    1 is odd 
    2 is even 
    3 is odd 
    4 is even 
    5 is odd 
    6 is even 
    7 is odd 
    8 is even 
    9 is odd 
    There are 5 even numbers in the vector.

### 其他示例

其他示例，可以看msdn的：[demo](http://msdn.microsoft.com/zh-cn/library/dd293599.aspx)

## 语法

语法，也参考msdn：[lambda语法](http://msdn.microsoft.com/zh-cn/library/dd293603.aspx)

## 小结