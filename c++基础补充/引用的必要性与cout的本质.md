#### 引用

###### 存在原因

既然引用的底层就是指针 为什么还需要引用呢？

1. **语法上** **重载运算符必须依赖引用**  举个例子

   ```c++
   // 假如没有引用，只能用指针：
   Vector a, b, c;
   c = *(&a + &b);             // 极其反人类  而且这样是完全错误的 指针相加减表示指针算数偏移 不能重载为加法
   std::cout << *ptr;          // 链式调用 std::cout << a << b 无法自然书写
   
   // 有了引用：
   Vector operator+(const Vector& a, const Vector& b);
   c = a + b;                  // 像内置原生类型一样自然
   std::cout << a << b;
   ```

2. **语义安全** 引用必须初始化 并且永远不为空 并且不可重新绑定  更安全

3. 完美支持现代c++的**移动语义** 通过**右值引用**实现**移动语义与完美转发** **消除**了沉重的**深度拷贝**

###### 详解

***重载运算符为啥必须依赖引用？***

1. **数组下标运算符**读写问题 

   ```c++
   如果没有引用 只有返回值
       先来讲讲一个细节，编译器是在函数编译阶段，根据函数签名就决定好了底层汇编指令的，而不是运行的时候根据调用的语句去猜
       所以如果是值返回的话，编译器看到返回类型是int，就会生成一条拷贝指令，把原对象的值放到EAX寄存器里面（32位），并且打上右值的标签
       如果是引用返回的话，编译器看到返回类型是int&，本质还是指针，CPU不会复制数据内容，而是直接把内部元素的内存地址通过RAX寄存器（64位）传出来，并且打上左值的标签
       
   int operator[](int index); // 按值返回
   这样返回的是一个临时副本（右值） 是在给一个即将销毁的临时整数赋值 编译器会直接报错
       
   假如没有引用，为了能修改，只能返回指针
       
   也就是	int* operator[](int index); // 返回内部元素的地址
   那么干啥都要加个解引用符号↓↓↓
   *v[0] = 10;       // 相当于 *(v.operator[](0)) = 10;
   int x = *v[0];    // 连读取一个值，也必须解引用！
   
   有了引用：
   int& operator[](int index) { 
   return data_[index]; // 返回底层元素的别名
   }
   写法非常方便并且和原生C语言数组一模一样
   ```

2. **加法连加** 

   ```c++
   加法必须支持链式操作  a+b+c  编译器解析的顺序就是 先算(a+b)的临时结果  算出来的临时结果再和c相加
   
   假设没有引用 为了避免大对象拷贝  只能传指针
   // 传指针版本
   Matrix operator+(const Matrix* a, const Matrix* b);
   但是！ Matrix result = &(&a + &b) + &c; // 语法完全崩溃 
   
   有了引用：
   Matrix operator+(const Matrix& a, const Matrix& b);
   (a+b)的时候没有拷贝开销 并且计算出来的临时对象直接作为const Matrix&绑定到下一个加法中  
   ```

   

3. **流输出运算符<<：链式调用cout<<a<<b**

   每天写的

   ```c++
   std::cout << a << b << std::endl;
   ```

   本质上是

   ```c++
   ((std::cout.operator<<(a)).operator<<(b)).operator<<(std::endl);
   ```

   前一个**<<运算**必须**把cout这个对象**原原本本地**交还给下一个<<**继续使用

   1. 为什么不能按值返回？

      标准库直接把它的**拷贝构造函数禁用了**(delete) 禁止拷贝  因为**内部有文件描述符、缓冲区锁和状态** 只能是同一个对象

   2. 为什么不能返回指针？

      返回指针的话 就变成了 (cout << a) -> operator<<(b);或者*(cout << a) << b;  完全破坏了流畅性

   3. 只有返回引用std::ostream&

      ```c++
      std::ostream& operator<<(std::ostream& os, const MyData& data) {
          os << data.val;
          return os; // 返回同一个输出流的引用（别名）
      }
      ```



#### cout

###### 详解

1. cout不是一个函数 而是一个全局对象 类型是std::ostream（标准输出类）

2. **ostream**大致模样

   ```c++
   class ostream {
   public:
       // 针对各种基础数据类型，重载了大量的 operator<<
       ostream& operator<<(int val);
       ostream& operator<<(double val);
       ostream& operator<<(const char* str);
       ostream& operator<<(bool val);
       // ... 其他类型
   };
   ```

   所以cout<<10; 其实就是 cout.operator<<(10);

3. **operator<<**内部在干嘛

   以operator<<(int val)为例：（伪代码）

   ```c++
   ostream& ostream::operator<<(int val) {
       // 1. 将整数转换为字符串（比如把数字 10 转成字符 '1', '0'）
       char buffer[32];
       int len = convert_int_to_string(val, buffer);
   
       // 2. 将字符写入到底层的缓冲区（streambuf）
       // 最终会通过 Linux 的 write 系统调用写给文件描述符 1 (stdout)
       this->rdbuf()->sputn(buffer, len);
   
       // 3. 最关键的一步：把“自己”（当前这个 ostream 对象的引用）返回出去！
       return *this; 
   }
   ```

   1. 底层走的是linux系统调用 先缓存在ostream缓冲区中 遇到刷新条件时调用write

   2. 为什么能连着写？核心就在 **return *this;**   屏幕输出完之后会**返回cout本身的引用  接着打印后面的** 再返回...就像接力棒一样 

   3. **std::endl**本质上是一个**函数指针**

      ```c++
      // endl 是一个接收 ostream& 并返回 ostream& 的普通函数
      std::ostream& endl(std::ostream& os) {
          os.put('\n');      // 1. 输出换行符
          os.flush();        // 2. 强行把缓冲区的数据冲刷到屏幕上（调用 write）
          return os;         // 3. 同样返回流对象
      }
      ```

      在ostream里面：

      ```c++
      class ostream {
      public:
          // 接收一个签名为 ostream&(ostream&) 的函数指针
          ostream& operator<<(ostream& (*func)(ostream&)) {
              // 直接调用传进来的这个函数，并把自己 (*this) 作为参数传进去
              return func(*this); 
          }
      };
      ```

4. 如何让**自己的结构体**也能使用cout<<

   写一个**全局重载函数**：

   ```c++
   #include <iostream>
   
   // 全局重载 operator<<
   // 必须返回 std::ostream&，以支持链式输出
   std::ostream& operator<<(std::ostream& os, const Twist& t) {
       os << "[Linear: " << t.linear_x << " m/s, Angular: " << t.angular_z << " rad/s]";
       return os; // 返回流的引用
   }
   
   int main() {
       Twist cmd{1.5, 0.2};
       
       // 编译器会自动调用：operator<<(std::cout, cmd)
       // 然后紧接着调用：.operator<<(std::endl)
       std::cout << cmd << std::endl; 
       
       return 0;
   }
   ```

   