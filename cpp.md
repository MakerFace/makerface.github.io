# C++ 学习笔记

> [代码案例库](https://github.com/0voice/cpp_new_features)

1. **auto：自动推断(定义时)变量类型**

   ```
    auto a = vet.begin();
   ```
2. **decltype：自动推断(定义后)变量类型**

   ```
    int a = 0;
    decltype(a) b = 1; // int b = 1;
    struct C{ double x;} c;
    decltype(c.x) d; // double d;
    decltype(c) e; // C e;
    using C_ = decltype(c);
   ```

   **推断函数返回值类型**

   ```
    template<typename T, typename U>
    auto add(T x, U y) -> decltype(x+y){
        return x + y; // T 必须有operator+()方法
    }
   ```

   > **补充：c++14新特性**
   >
   > ```
   >  decltype(auto) ref_a = (a); // type of ref_a is int&, an alias of a
   > ```
   >
   > **ref_a是a的别名，C++11则无法通过编译**
   >

   **当传入带括号的参数时**

   ```
    int a;
    decltype((a)) b; // int &，a是左值
    decltype((0)) c; // int，0是纯右值
    decltype((get(a))); // int&&，get方法返回临终值，右值
   ```
3. **r-value和l-value，x-value, prvalue**

   1. **左值：在内存中有确定的存储地址的对象表达式的值称为左值。**
      **左值可以作为右值使用，**`b = i`就是将i的左值作为右值使用的。
      **C++11及之后左值也包含了将亡值，称为泛左值。**
   2. **传统右值：C++11之前，所有非左值。**
   3. **右值：C++11及之后，右值包括纯右值和将亡值(expiring)。**
      **纯右值：字面量值（0、"hello"等）；不具名临时对象，如函数返回值等**
      **将亡值：返回****右值引用**的函数返回值；转换为**右值引用**的转换函数返回值。

      > **可以通过取址&区分左右值**
      >
      > **右值无法取地址**
      >

      ```
       int j=0;
       int &ref_j = j; // left reference
       int &&rref_j = j; // right reference
       int &ref_j_plus = j+1; //error
       int &&rref_j_plus = j+1;//right!
       int &ref_zero = 0; // error!
       int &&ref_zero = 0; //right!
      ```

      **右值的用处：**

      **一、获取临时对象**

      ```
       int getZero(){ return 0; }
       int val = getZero(); // create temporary int and destory it
       int &&val = getZero(); // temporary int, alive with val
      ```

      **二、右值引用自动推断**

      ```
       template<typename T>
       void fun(T&& t){}
       fun(10); // t == right value
       int x = 10;
       fun(x); // t == left value
      ```

      **自动推断左右值。可以利用该机制实现移动语义以及完美转发。**
   4. **移动语义**

      ```
       class T{
       private:
           int* d;
       public:
           T():d(new int(0)){}
           ~T(){ delete d; } // 析构函数阻止隐式移动构造函数
       }
      ```

      **如果用一个T对象a去构造另一个对象b，或函数返回T时。**

      ```
       T a;
       T b(a); // 调用默认复制构造函数，按位拷贝，导致悬置指针：b先析构，a在析构时出现double free问题，原因是a, b指向同一片堆区。做普通形参、返回值等都会引起这个问题。
       T getT() { return T() };
      ```

      **则必须实现深度复制构造函数**

      ```
       T(const T& a):d(new int(*a.d)){}
      ```

      **引发的一个问题是，生成临时对象时，也会调用深度复制构造函数，然后在调用复制构造函数给有名对象。**

      ```
       T real = getT(); // 调用两次复制构造函数
      ```

      **因此出现移动构造函数(move constrctors)，将被moved object置空，防止悬置。**

      ```
       T(T &&t) noexcept : d(std::move(t.d));
       T(T&&) = default;
       T(T&&) = delete;
      ```
   5. **完美转发**
      `std::forward<T>(t)`

      ```
       template<typename T>
       void func(T& t){ cout << "lvalue\n";}
       
       template<typename T>
       void func(T&& t) { cout << "rvalue\n";}
       
       template<typename T>
       void call(T &&t){
           func(std::forward<T>(t));
       }
      ```

      > **`<T>`**不能丢，C++11编译报错。
      >
4. **default和delete**
   **default使用默认函数**
   **delete禁用默认函数（例如不可拷贝的实现）**
5. **override和final**
   **override关键词显示覆盖父类虚方法**
   **final表明不可被继承或者虚函数不可覆盖**
6. **尾置返回类型**

   ```
    []() -> type {}
   ```
7. **&，&&**
   **在类成员函数中，&，&&可用来修饰this指针。**

   ```
    class echoer{
        // void echo() const{} // 注意实现了该函数，下面两个就没法重载了
        void echo() const & { cout << "echo left value\n"; }
        void echo() const&& { cout << "echo righ value\n"; }
    };
    echoer e;
    e.echo(); // left value
    echoer().echo(); // right value
   ```
8. **作用域枚举，与枚举类**
   **传统enum隐式转换为int**

   ```
    enum class Color:char{ RED, BLACK };
   ```
9. **is_same**
   **C++14以上增加** `is_same_v()`函数返回 `is_same`的value。因此做适配

   ```
    namespace std
    {
        template <typename T, typename U>
        bool is_same_v(T, U)
        {
            return is_same<T, U>::value;
        }
    }
    static_assert(std::is_same_v<int, int32_t>)
   ```
10. **lock_guard，unique_lock**
    **锁管理遵循RAII(Resource Acquisition Is Initialization， 资源获取即为初始化)，利用构造函数和析构函数，在函数返回时自己主动析构，降低了死锁风险。**

    ```
     std::mutex lock_;
     int getRes(){
         std::lock_guard<std::mutex> lock(lock_); // RAII
         return res;
     }
    ```

    **unique_lock比lock_guard更灵活，效率低一点，内存占用也会增加。**
11. **explicit**
    **用户定义的显式转换函数。可以保证在不需要****调用隐式转换函数**（包括构造函数）的时候，无法隐式调用。

    ```
     struct X{
         operator int() const { return 7; } // implicit conversion
         explicit operator int*() const { return nullptr; } // explicit conversion
     }x;
     struct Y{
         explicit Y(X x){
             
         }
     }
     auto = 1+x; // 7, implicit conversion
     (int*)x; // nullptr
     auto y = Y(x); // convert x to y
    ```
12. **this_thread**

    1. **yield：该函数取决于实现，依赖于OS中的调度器机制和系统状态。先进先出实时调度器中，将会挂起当前线程，并放入可运行的同优先级队列队尾，如果队列中只有一个线程，则不产生任何效果。使用场景：忙等。sleep等待固定时间，接受终端，yield根据CPU时间片确定，不接受中断。**
    2. **get_id**
    3. **sleep_for**
    4. **sleep_unitl**
13. **正则表达式**

    ```
     const char* reg_exp = "[,.\\t\\n;:]";
     std::regex rgx(reg_exp);
     std::cmatch match;
     const char* target = "Polytenchnic University of Turing";
     if(regex_search(target, match, rgx)){
     const size_t n = match.size();
     for (size_t a = 0; a < n; ++a){
     string str(match[a].first, match[a].second);
     cout << str << "\n";
     }
     }
    ```
