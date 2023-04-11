---
title: Modern C++ Learning ( Core Language Features )
date: 2023-04-12 18:32:00 +0800
categories: [Blogging, Learning]
tags: [c++]
---

本篇是对 C++11-C++23 的学习整理，主要参考 [cppreference](https://en.cppreference.com/w/cpp/11)

## `auto` & `decltype`

c++11 引入了 `auto` 和 `decltype` 关键字，提供了在编译期进行类型推导的能力

- `auto`: 用于推导出变量的类型 (`decltype(auto)` 同理，但遵循 decltype 的推导规则，主要解决 `auto` 无法表达引用类型的问题)
  - `const` 和 `&` 需要手动添加

  ```cpp
  // auto x = expr
  auto i = 0; // i is int
  auto& i0 = i; // i0 is int&
  auto&& i1 = 0; // i1 is int&&
  auto* p_i0 = &i; // p_i0 is int*
  auto p_i1 = &i; // p_i1 is int*

  auto j = 0.0; // j is double
  decltype(auto) j0 = j; // j0 is double
  decltype(auto) j1 = (j); // j1 is double&
  // The placeholder decltype(auto) must be the sole constituent of the declared type

  // new expression
  double* p = new double[]{1, 2, 3}; // creates an array of type double[3]
  auto p = new auto('c');            // creates a single object of type char. p is a char*
  auto q = new std::integral auto(1);         // OK: q is an int*
  auto q = new std::floating_point auto(true) // ERROR: type constraint not satisfied
  auto r = new std::pair(1, true); // OK: r is a std::pair<int, bool>*
  auto r = new std::vector;        // ERROR: element type can't be deduced

  // auto& f();
  template<class T, class U>
  auto add(T t, U u) { return t + u; } // the  return type is the type of operator+(T, U)

  // in the parameter declaration of a non-type template parameter
  template<auto n> // C++17 auto parameter declaration
  auto f() -> std::pair<decltype(n), decltype(n)> // auto can't deduce from brace-init-list
  {
      return {n, n};
  }

  // function
  auto (*p)() -> int; // declares p as pointer to function returning int
  auto (*q)() -> auto = p; // declares q as pointer to function returning T, where T is deduced from the type of p
  ```

- `decltype ( entity )` & `decltype ( expression )`: 用于推导表达式类型
  - decltype is useful when declaring types that are difficult or impossible to declare using standard notation, like lambda-related types or types that depend on template parameters.
  - 如果加上括号，编译器会将其视为 expression，否则视为 entity
    - if the value category of expression is xvalue, then decltype yields T&&;
    - if the value category of expression is lvalue, then decltype yields T&;
    - if the value category of expression is prvalue, then decltype yields T.
  
  ```cpp
  const int &i = 0;
  decltype(i) j = 0; // j is const int&
  
  // perfect forwarding of a function call must use decltype(auto)
  // in case the function it calls returns by reference --> 如果写 auto，则无法表示引用类型
  template<class F, class... Args>
  decltype(auto) PerfectForward(F fun, Args&&... args) 
  { 
      return fun(std::forward<Args>(args)...); 
  }

  template<typename T, typename U>
  auto add(T t, U u) -> decltype(t + u) { // 这里只能后置返回类型，因为t和u的作用域在后面
    return t + u;
  }
  ```

## trailing return type

- 使用 `auto` 和 `decltype` 情况下的后置返回类型 --> 受到用于类型推导的变量作用域的限制，只能后置

  ```cpp
  template<typename T, typename U>
  auto add(T t, U u) -> decltype(t + u) {
    return t + u;
  }
  ```

- 函数指针作为返回值，需要后置返回类型 --> 函数指针的返回类型建议后置，不然容易产生歧义

## defaulted and deleted functions

- `default`: 用于指定函数的默认实现
- `delete`: 用于指定函数的删除实现
  - default 和 delete 都是函数定义，与其他同名函数之间构成重载关系
  - 五之法则：用户定义的析构函数、复制构造函数、复制赋值运算符的存在会阻止移动构造函数和移动赋值运算符的隐式定义 --> 任何想要移动语义的类必须声明全部五个特殊成员函数。

  ```cpp
  struct Test {
    Test(){}
    Test(const Test&) = delete;
  };
  int main(){
    Test t1;
    Test t2(std::move(t1)); // error: use of deleted function ‘Test::Test(const Test&)’
    // 显示弃置的复制构造实际上也阻止了移动构造函数的隐式定义
  }
  ```

## final specifier and override specifier

这个特性比较简单，主要用于让我们在写继承类和重写虚函数时更加安全

- `override`: 用于指定虚函数必须重写基类的虚函数

  ```cpp
  struct Base {
    virtual void doSomething(int i) const {
      std::cout << "This is from Base with " << i << std::endl;
    }
  };
  ​
  struct Derivied : Base {
    virtual void doSomething(int i) {
      std::cout << "This is from Derived with " << i << std::endl;
    }
  };
  ​
  void letDoSomething(Base& base) {
    base.doSomething(419);
  }
  ​
  int main() {
    Derived d;
    letDoSomething(d);  //输出结果： "This is from Base with 419"
  }
  ```

  上述问题你或许遇到过，`Derived::doSomething` 函数把 `Base::doSomething` 的 `const` 给搞丢了。所以他们两者并没有相同的函数签名，前者也没有如我们预想的对后者进行覆写。

  加上 `override` 之后，编译器就会报错，告诉我们这个函数并没有覆写基类的虚函数。

  ```cpp
  struct Derivied : Base {
    virtual void doSomething(int i) override {
      std::cout << "This is from Derived with " << i << std::endl;
    }
  };
  ```

  此外，作为一个覆写方法，使用 `override` 关键字后可以省略 `virtual` 关键字。

- `final`: 用于指定类或者虚函数不可被继承或者重写

  ```cpp
  struct Base {
    virtual void doSomething(int i) const {
      std::cout << "This is from Base with " << i << std::endl;
    }
  private:
    virtual int dontChangeMe() const = 0;
  };

  struct Derived: Base {
    void doSomething(int i) const override {
      std::cout << "This is from Derived with " << i << std::endl;
    }
  private:
    int dontChangeMe() const final  { return 419; }
  };

  struct Derived2: Derived {
  private:
    // int dontChangeMe() const override { return 61; }
  };
  ```

## lvalue, rvalue, xvalue, glvalue, prvalue

### 概念

- expression 表达式
  - glvalue (generalized lvalue) 泛左值：指一个通过评估能够确定对象/位域/函数的标识的表达式 --> 具名对象，可以取地址
    - lvalue
    - xvalue (expiring value) 将亡值：表示资源可以被重用的对象和位域，通常是因为他们接近其生命周期的末尾，另外也有可能是经过右值引用的转换产生的。 --> 临时对象，可以取地址
  - rvalue
    - prvalue (pure rvalue) 纯右值：指一个通过评估能够用于初始化对象和位域，或者能够计算运算符操作数的值的表达式 --> 临时对象，不可以取地址
    - xvalue

注意：值类别都是表达式的属性，我们常说的左值和右值实际上指的是表达式。

例如，`i++`是先拷贝一个临时值作为返回值，即得到的是将亡值，`++i` 是返回的自己，即左值。

### xvalue

本质上说，产生 xvalue 的途径有两种，第一种是使用类型转换将泛左值转换为其右值引用；第二种是临时量实质化（c++17引入），指的是纯右值转换为一个临时对象的过程。例如：

```cpp
struct X {
  int a;
  X() : a(255) {}
  X(const X&) {}
  ~X(){}
};
X getX() { return X(); }
X getX2() { return getX(); }
int main() {
  int b = getX2().a;
}
```

上述 `X()` 是一个纯右值，但是在 `b = X().a` 这个表达式中，`X()` 被转换为一个临时对象，这个临时对象就是一个 xvalue。

### std::move

`std::move` 的作用是将一个泛左值转换为一个将亡值，这个将亡值可以被绑定到一个右值引用上，从而实现对一个左值的移动操作（通过移动构造函数）。

注意，单纯地将一个左值转换到另外一个左值没有什么意义，正确的使用场景是在一个右值被转换为左值后需要再次转换为右值。例如：

```cpp
struct X {
  int a;
  X() : a(255) {}
  X(const X&) {}
  X(X&&) {}
  ~X(){}
};
void moveX(X&& x) {
  // 无论实参是左值还是右值，其形参都是左值
  X x2(std::move(x));
  // X x2(static_cast<X&&>(x));
}
int main (){
  moveX(X());
}
```

### 万能引用 & 引用折叠

- 常量左值引用既可以引用左值又可以引用右值，可惜其具有常量性质
- c++11 引入了万能引用，即 `T&&`，它可以引用左值也可以引用右值，但是它的具体类型取决于实参的类型

```cpp
void foo(int &&i){} // 右值引用
template<class T>
void bar(T &&t){} // 万能引用

int get_val() { return 0; }
int &&x = get_val(); // 右值引用
auto &&y = get_val(); // 万能引用
```

引用折叠规则：

| 模板类型 | T 实际类型 | 最终类型 |
| --- | --- | --- |
| T& | R | R& |
| T& | R& | R& |
| T& | R&& | R& |
| T&& | R | R&& |
| T&& | R& | R& |
| T&& | R&& | R&& |

### std::forward

实现完美转发的时候需要利用引用折叠规则，并使用 `static_cast<T &&>` 进行类型转换，从而保留引用的类型。

和移动语义一样，c++11 提供了一个更为便捷的方法 `std::forward`。

### 编译器提供的的隐式移动

需要提供`-fno-elide-constructors`编译选项，关闭返回值优化，不然看不到什么输出。

```cpp
struct X {
  X() = default;
  X(const X&) { std::cout << "copy" << std::endl; }
  X(X&&) { std::cout << "move" << std::endl;}
  ~X(){ std::cout << "destructor" << std::endl; }
};
X f(X x) { return x; }
int main() {
  X x = f(X{});
}
```
