我在实现 NLT(Nest Louds Trie) 的 Iterator 时，使用了这样的写法（最小化问题代码）：
```c++
#include <stdio.h>
template<class T>
struct A {
    template<class U>
    struct X {
        void foo(U* p) { printf("p->a = %d\n", p->a); }
    };
};
template<class T>
struct B {
    typedef typename A<T>::template X<B<T> > X;
//  friend typename    X; // g++ fail
    friend typename B::X; // g++ ok
private:
    int a = 1234;
};
int main() {
    B<int> a;
    B<int>::X().foo(&a);
    return 0;
}
```
在 Visual C++ 2017 中编译、测试完成之后，开始在 gcc 中编译、测试，然后 `g++ fail` 那行就出现了诡异的编译错误：
```c++
template_friend.cpp:12:21: error: expected nested-name-specifier before ‘X’
  friend typename    X; // g++ fail
                     ^
template_friend.cpp:12:21: error: ‘X’ is neither function nor member function; cannot be declared friend
template_friend.cpp:12:21: error: ‘int B<T>::X’ conflicts with a previous declaration
template_friend.cpp:11:43: note: previous declaration ‘typedef typename A<T>::X<B<T> > B<T>::X’
  typedef typename A<T>::template X<B<T> > X;
                                           ^
template_friend.cpp: In instantiation of ‘void A<T>::X<U>::foo(U*) [with U = B<int>; T = int]’:
template_friend.cpp:19:20:   required from here
template_friend.cpp:6:45: error: ‘int B<int>::a’ is private within this context
   void foo(U* p) { printf("p->a = %d\n", p->a); }
                                          ~~~^
template_friend.cpp:15:10: note: declared private here
  int a = 1234;
          ^~~~
```
还好，这段代码稍作修改就可以正确地通过编译（参见紧邻的 `g++ ok` 那行代码）。

我们已经在 gcc 官方网站报告了 [这个 bug](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=86876).