---
layout:     post
title:      "C++知识点汇总"
subtitle:   "面试题汇总 一"
date:       2020-01-01
author:     "Felix Zhang"
header-img: "img/in-post/2019-11-25-C++-QNA/bg.jpg"
catalog: true
tags:
   - C++

---

# C++面试题问答汇总

1. ##### 引用和拷贝的区别？引用的优缺点和拷贝的优缺点？

2. ##### 深拷贝和浅拷贝的区别？

   类的函数成员存在__指针成员__时会产生深拷贝和浅拷贝和问题。如果该类没有自定义拷贝构造函数，那么在进行对象拷贝时会调用默认拷贝构造函数，此时进行的是浅拷贝，即只是对指针的拷贝，新拷贝的指针和原指针指向同一内存空间，深拷贝指不仅对指针进行拷贝，而且对__指针指向的内存也进行拷贝__。在调用析构函数时，浅拷贝会导致堆内存被多次释放。关于类对象拷贝的情况，需要注意：

   * 当函数的参数为对象时，实参传递给形参的实际是对实参的一个拷贝，系统通过拷贝构造函数实现。
   * 当函数返回一个类对象时，实际上返回的是对函数内对象的一个拷贝，系统用过拷贝构造函数实现。
   * 浅拷贝带来问题的本质在于析构函数释放多次堆内存，使用std::shared_ptr，可以完美解决这个问题。

   具体例子如下：

   ~~~C++
   #include <iostream>  
   using namespace std;
    
   class Student
   {
   private:
   	int num;
   	char *name;
   public:
   	Student();
   	~Student();
   };
    
   Student::Student()
   {
   	name = new char(20);
   }
   Student::~Student()
   {
       //浅拷贝时此处易发生错误。
   	delete name;
   	name = NULL;
   }
    
   int main()
   {
   	{// 花括号让s1和s2变成局部对象，方便测试
   		Student s1;
   		Student s2(s1);// 复制对象，s2的name和s1的name指向同一内存。
   	}
       //s2和s1依次析构，发生错误
   	system("pause");
   	return 0;
   }
   ~~~

   经过改进后，添加自定义拷贝构造函数。

   ~~~C++
   #include <iostream>  
   using namespace std;
    
   class Student
   {
   private:
   	int num;
   	char *name;
   public:
   	Student();
   	~Student();
   	Student(const Student &s);//拷贝构造函数，const防止对象被改变
   };
    
   Student::Student()
   {
   	name = new char(20);
   	cout << "Student" << endl;
    
   }
   Student::~Student()
   {
   	cout << "~Student " << (int)name << endl;
   	delete name;
   	name = NULL;
   }
   Student::Student(const Student &s)
   {
   	name = new char(20);
   	memcpy(name, s.name, strlen(s.name));
   	cout << "copy Student" << endl;
   }
    
   int main()
   {
   	{// 花括号让s1和s2变成局部对象，方便测试
   		Student s1;
   		Student s2(s1);// 复制对象
   	}
   	system("pause");
   	return 0;
   }
   ~~~

3. ##### 智能指针是什么？如何解决内存泄露问题？

   智能指针帮助我们处理资源泄漏，悬空指针和比较隐晦的由一场造成的资源泄露。

   ****

   考虑下边简单的代码：

   ~~~C++
   int main()
   {
       int *ptr = new int(0);
       return 0;
   }
   ~~~

   上边这个程序，如果忘记释放掉ptr指向的不再使用的内存，就会导致资源泄露（resource leak），即内存泄漏。将代码改进：

   ~~~C++
   int main()
   {
       int *ptr = new int(0);
       delete ptr;
       return 0;
   }
   ~~~

   上边程序虽然申请了释放内存，但是ptr会变成空悬指针（dangling pointer，即野指针），悬空指针不同于空指针（nullptr），会为系统带来诸多隐患，如无法用if语句判断是否为野指针。

   ~~~C++
   int main()
   {
       int *ptr = new int(0);
       delete ptr;
       ptr = nullptr;
       return 0;
   }
   ~~~

   考虑到其他问题，如果new申请内存失败，会抛出异常，所以要用上try catch语句或直接判断

   ~~~C++
   #include <iostream>
   using namespace std;
   
   int main()
   {
       int *ptr = new int(0);
       if(!ptr)
       {
           return 0;
       }
       if(hasException())
       {
           throw exception();
       }
       
       delete ptr;
       ptr = nullptr;
       return 0;
   }
   ~~~

   对于上述的任何一个异常，hasException返回true，程序会抛出一个异常，导致程序中断，而申请的异常并没有被释放掉。继续改进：

   ~~~C++
   #include <iostream>
   using namespace std;
   
   int main()
   {
       int *ptr = new int(0);
       if(!ptr)
       {
           return 0;
       }
       if(hasException())
       {
           delete ptr;
           ptr = nullptr;
           throw exception();
       }
       return 0;
   }
   ~~~

   而使用智能指针，上述的问题都会解决。

   > 自C++11起，C++标准提供两大类型的智能指针：
   >
   >   　　1. Class shared_ptr实现共享式拥有（shared ownership）概念。多个智能指针可以指向相同对象，该对象和其相关资源会在“最后一个引用（reference）被销毁”时候释放。为了在结构复杂的情境中执行上述工作，标准库提供了weak_ptr、bad_weak_ptr和enable_shared_from_this等辅助类。
   >
   >   　　2. Class unique_ptr实现独占式拥有（exclusive ownership）或严格拥有（strict ownership）概念，保证同一时间内只有一个智能指针可以指向该对象。它对于避免资源泄露（resourece leak）——例如“以new创建对象后因为发生异常而忘记调用delete”——特别有用。
   >
   > 　　注：C++98中的Class auto_ptr在C++11中已不再建议使用。

   ****

   **shared_ptr**

   > 几乎每一个有分量的程序都需要“在相同时间的多处地点处理或使用对象”的能力。为此，我们必须在程序的多个地点指向（refer to）同一对象。虽然C++语言提供引用（reference）和指针（pointer），还是不够，因为我们往往必须确保当“指向对象”的最末一个引用被删除时该对象本身也被删除，毕竟对象被删除时析构函数可以要求某些操作，例如释放内存或归还资源等等。

   所以我们需要“当对象再也不被使用时就被清理”的语义。Class shared_ptr提供了这样的共享式拥有语义。也就是说，多个shared_ptr可以共享（或说拥有）同一对象。**对象的最末一个拥有者有责任销毁对象，并清理与该对象相关的所有资源。**shared_ptr的目标就是，在其所指向的对象不再被使用之后（而非之前），自动释放与对象相关的资源。以下代码节选自《C++标准库（第二版）》。

   ~~~C++
   #include <iostream>
   #include <string>
   #include <vector>
   #include <memory>
   using namespace std;
    
   int main(void)
   {
       // two shared pointers representing two persons by their name
       shared_ptr<string> pNico(new string("nico"));
       shared_ptr<string> pJutta(new string("jutta"),
               // deleter (a lambda function) 
               [](string *p)
               { 
                   cout << "delete " << *p << endl;
                   delete p;
               }
           );
    
       // capitalize person names
       (*pNico)[0] = 'N';
       pJutta->replace(0, 1, "J");
    
       // put them multiple times in a container
       vector<shared_ptr<string>> whoMadeCoffee;
       whoMadeCoffee.push_back(pJutta);
       whoMadeCoffee.push_back(pJutta);
       whoMadeCoffee.push_back(pNico);
       whoMadeCoffee.push_back(pJutta);
       whoMadeCoffee.push_back(pNico);
    
       // print all elements
       for (auto ptr : whoMadeCoffee)
           cout << *ptr << " ";
       cout << endl;
    
       // overwrite a name again
       *pNico = "Nicolai";
    
       // print all elements
       for (auto ptr : whoMadeCoffee)
           cout << *ptr << " ";
       cout << endl;
    
       // print some internal data
       cout << "use_count: " << whoMadeCoffee[0].use_count() << endl;
    
       return 0;
   }
   
   /*
   注：shared_ptr本身提供默认内存释放器(default deleter)。但其并不能释放数组内存空间，我们可以对其进行改造，提供自己的内存释放器：
   shared_ptr<int> pJutta2(new int[10],
   	//deleter(a lambda function)
   	[](int *p)
   	{
   		delete[] p;
   	}
   );
   默认内存释放器调用如下：
   shared_ptr<int> p(new int[10], default_delete<int[]>());
   */
   ~~~

   

   ****

   **unique_ptr**

   >unique_ptr是C++标准库自C++11起开始提供的类型。它是一种在异常发生时可帮助避免资源泄露的智能指针。一般而言，这个智能指针实现了**独占式拥有概念**，意味着它可确保一个对象和其相应资源同一时间只被一个指针拥有。一旦拥有者被销毁或变成空，或开始拥有另一个对象，先前拥有的那个对象就会被销毁，其任何相应资源也会被释放。

   ~~~C++
   #include <memory>
   using namespace std;
   
   int main()
   {
       unique_ptr<int> ptr(new int[0]);
       return 0;
   }
   ~~~

   

   *****

   **智能指针如何实现？**

4. **容器在处理下标时和数组有何不同？**

   容器中的下表必须是无符号类型，且是直接引用，数组中可为负数，且是当前的相对引用。

5. **< cname >和“name.h”中的函数名字有什么区别？**

   前者包含在std命名空间中，后者没有。

6. ##### new和delete的区别有哪些？

   * new按类型申请内存，malloc按申请的大小分配内存。
   * new返回的是对象的指针，malloc返回的是void*，通常需要强制类型转换。
   * new不仅分配内存，还会调用构造函数，malloc不会。
   * new需要用delete销毁，此时会调用析构函数；malloc需要用free来销毁，不调用析构。
   * new无法像malloc一样在分配失败的时候用realloc扩容。
   * new是一个操作符可以重载（如何创建一个类只在栈上创建对象）但malloc是一个库函数。
   * new如果分配失败了会抛出bad_malloc的异常，但malloc失败了会返回NULL

7. ##### 函数传值的方式有哪些？

8. ##### C++的三大特性？

9. ##### 虚函数的工作原理？

10. ##### 如果是多重继承只有一个虚表指针吗？

11. ##### 函数模板与类模板的区别？

12. ##### 类型萃取是什么？

13. ##### lambda函数是什么，其使用场景是什么？

14. ##### C++内存分区，未初始化的全局变量放在那里？如果编译了二进制文件这其中有他的位置吗？

15. ##### 进程间的通信方式？互斥锁和自旋锁？

16. ##### 野指针是什么？有什么工具可以检测吗？

17. ##### 一个结构体，能用_memcpy_判断两个结构体存放的东西是否一样吗？

18. ##### 左值引用和右值引用有什么区别？

19. ##### 四种cast强制类型转换的区别和使用？

20. ##### C++中STL是线程安全还是不安全的？

21. ##### C++有哪些创建线程的方法？

22. ##### 讲一下windows下的消息机制。

23. ##### C++中的那种行为体现了接口的特性？

24. ##### 三种继承方式的使用场景？你如何做出判断有没有自己的原则？

25. ##### 既然私有继承派生类不能访问，为什么要继承？

26. ##### 如何访问其他类的私有成员？

27. ##### 既然私有成员不想让外界访问，为什么设计友元的方式去访问其私有成员？

28. ##### 函数重载与函数模板的区别？

29. ##### 函数模板如何调用到对应的重载版本？

30. ##### 函数模板在二进制文件中存放了全部重载版本吗？

31. ##### 函数重载max，是直接比较大小吗？

32. ##### 虚函数表是谁维护的？

33. ##### 虚函数表存放的是什么？

34. ##### 派生类虚函数表的具体结构？

35. ##### 基类指针如何动态得知道自己指向的动态对象？

36. ##### 抽象类可以有成员变量吗？

37. ##### c的struct与c++的struct的区别？

38. ##### const的作用在函数前面和函数后面有什么不一样？

39. ##### struct与class的区别？什么时候用stuct什么时候用class？

40. ##### 类函数哪些需要设置程序函数？你怎么判断一个函数需要声明为虚函数？

41. ##### 基类指针操作基类对象，和基类指针操作派生对象，和派生类指针操作基类对象和派生类指针操作派生类对象，操作同名函数时的表现如何？

42. ##### 如果同名函数为序函数又是怎么样的?

43. ##### C++的虚拟内存分布？

44. ##### STL中vector中扩容的原理和具体实现方式？

45. ##### map和unordered_map的底层实现和性能区别？获取元素和增加删除元素怎么实现？

46. ##### 线程同步的方式是什么？

47. ##### 什么是内存泄漏？如何解决内存泄漏问题？如果new了一个内存，在delete之前这个进程被系统杀死了，这样算内存泄露吗？

48. ##### 静态库与动态库的区别？

49. ##### 如果在调试过程中，出现了需要循环很多次才出现的错误应该怎么调试？

50. ##### 手撕单例模式线程安全写法？

51. ##### 如果多个线程同时判断到当前对象未创建，应该怎么解决？