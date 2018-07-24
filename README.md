# moderncpp
=====================================

A list of modern c++ features / learning notes

### lvalue vs rvalue

lvalue could be at both left and right side of assignment operator(=) but rvalue could only be at right side
lvalue's address can be access via & operator, otherwise it's rvalue
a declared rvalue could be both lvalue and rvalue. If it has a name, it's a lvalue, otherwise rvalue.

```
  void foo(X && x) {
      X obj = x; // calling X(const X &rhs) - it has a name, it's lvalue even declared as rvalue
  }
  X&& go();
  X obj = go(); // calling X(X && rhs) - it has no name, rvalue
```
[reference](http://thbecker.net/articles/rvalue_references/section_01.html)

## std::move() ##
    turns argument into a rvalue even if it's not, by hiding name

    SubClass(SubClass && rhs)
        : Base(std::move(rhs) {
        }

## std::forward() ##
forwarding problem: lval or rval attribute might gets changed while forwarding as parameter.using std::forward will keep this attribute.

example:

    class Test {

    public:
        Test(){}

        Test(const Test&) {
            cout << "copy ctor" << endl;
        }
        Test(Test&&) {
            cout << "move ctor" << endl;
        }
    };
    
    template <typename T>
    void func(T t) {
    }

    template <typename T>
    void func2(T&& t) {
        func(t); // t is always lvalue and copy ctor is called
    }

    template <typename T>
    void func3(T&& t) {
        func(std::forward<T>(t)); // t's lval or rval attribute keeps unchanged. So copy ctor or move ctor could be called.
    }

## universal reference ##
  defined by Scott Meyes at: http://channel9.msdn.com/Shows/Going+Deep/Cpp-and-Beyond-2012-Scott-Meyers-Universal-References-in-Cpp11
  1. T&&
  2. Type of T defined via deduction
  try perfect forward as much as possible in this case:
  template <typename T>
  void doSomething(T&& t) {
      do(std::forward<T>(t));
  }

## std::initializer_list ##
used to access list initialized by braces.

int sum(const std::initializer_list<int> & list) {  
  int s = 0;
  for (const auto &i : list) {
    s += i;
  }
  return s;
}
std::cout << "{1, 3, 5} = " << sum({1,3,5}) << std::endl; //output: {1, 3, 5} = 9

## Top/Low level const ##
- Top level const : pointer itself is const
- low level const : when a pointer points to const object
example:
int n = 0;
const int *p1 = &n; // low level - pointer to const int
int *const p2 = &n; // top level - const pointer to int
const int c_int = 1; // top level
const int& c_ref = n; // low level

## auto ##
auto deduce type from initializing expression, if expression is reference or there is top level const/volatile, they are ignored.

int x = 1;
const int *p = &x; // low level const
auto p2 = p; // const not removed
*p2 = 2; // error

const int i = 0;
auto& r_i = i;
r_i = 1; // error: const qualifier was not removed

int i = 1;
auto&& ri_1 = i; // i is lvalue, int & is deduced -> int & && : collapsed to int &
auto&& ri_2 = 2; // 2 is rvalue, no additional addons -> int &&

## decltype ##
basics: to inspect exact declared type of an expression.
example:
int a = 0;
decltype(a) b = a;
template <typename T, typename S>
auto calc(T t, S s) -> decltype(t * s) {
   return t*s;
}

auto f = [](int a, int b) -> int {
   return a + b;
}
decltype(f) g = f;
g(1, 3) // 4

## forward ##
//forward unknown return value to another function
template <typename T1, typename T2>
void func(T1 t1, T2 t2) {
   auto && r = f1();
   f2(std::forward<decltype(f2())> (r)); // r will be forwarded as lval or rval according to decltype(...)
}

// decltype add reference, 
// The following example, which was provided by Stephan Lavavej, demonstrates that:
template <typename T>
auto array_access(T& array, size_t pos) -> decltype(array[pos]) {
  return array[pos];
}

std::vector<int> vect = {42, 43, 44};
int* p = &vect[0];

array_access(vect, 2) = 45;
array_access(p, 2) = 46;

//used in lamda
auto create_vec = [](auto const &x, size_t n) {
    return std::vector<std::decay_t<decltype(x)>>(n, x);
};
vector<int>vec = create_vec(5, 10);//{5...5} 10 times

## nullptr ##:
 nullptr is of type std::nullptr_t and could be converted to any pointer type.
 NULL is old MACRO defined 0, which has ambigous between 0 and (void *)0.
void f(char *){}
void f(int) {}
void g(int) {}
  
f(nullptr);
f(NULL);// error, ambiguous
f(0);
g(NULL);//warning, passing NULL to non-pointer type


## Scoped Enum ##
with class to make enum a strong type not implicitly convert to int to prevent potential issue. For example in a switch case below, if no enum type, a case value from another enum could be wrongly checked.

enum class Color {
  WHITE,
  RED,
};

enum class Location {
  US,
  EUROPE,
};

void foo(Location l) {
  switch (l) {

    case Location::US: // force usage of class prefix
      break;
    case Color::RED: //error, could not convert 'RED' from 'Color' to 'Location'
      break;
    default:
      return;
  }
}

alternative in boost - boost::scoped_enum

#include <boost/detail/scoped_enum_emulation.hpp>

BOOST_SCOPED_ENUM_START(Color)
{
    green,
    red,
    cyan
};
BOOST_SCOPED_ENUM_END
void foo(BOOST_SCOPED_ENUM(Color) a) {}

BOOST_SCOPED_ENUM(Color) sample( Color::red );
sample = Color::green;
foo( color::cyan );

## Delegating constructor ##
calling another ctor from ctor, avoiding "init()" in all related ctors.
class Test {
public:
	Test() {
	}

	Test(int a) : Test() {
	}
};

## constexpr ##
Compile time resolution/computation, replace template in some scenarios.

    constexpr int add(int a, int b) {
        return a + b;
    }
    
    //constexpr not work, compile error, since vector size is not decided at compile time
    /*
    constexpr int sum_size(const std::vector<int> &a, const std::vector<int> &b) {
        return a.size() + b.size();
    }
    */
    // constexpr works, size of array is decided at compile time
    template<std::size_t N, std::size_t M>
    constexpr int sum_size_of_array(const std::array<int, N> &a, const std::array<int, M> &b) {
        return a.size() + b.size();
    }
    
    constexpr int factorial(unsigned n) {
        return (n <= 1) ? 1 : n * factorial(n-1);
    }
    
	constexpr int a = 5; // compiler is not bound to set value of a at compile time, could decide later
	constexpr static int b = 1; // b is static, lifetime is whole program, compiler bound to compute value at compile time
	constexpr static int c = 2;
	constexpr static int d = b + c;
   constexpr static int val = add(1, 3);
   static constexpr auto fac = factorial(5);
   
## "extern" template ##
templated function will be specialized repeatedly in each compilation unit, which results in extra compilation time. Redundant specialized will be simpled dropped as waste. 
extern template void func<int>(); //"extern" will indicates that specialisation of this template function is in another compile unit

## explicit ##
for conversion operator, like safe bool idiom:
class BoolTest {
public:
	BoolTest() : a(1) {
	}
	explicit operator bool() {
	    return a == 1;
	}

	int a;
};
// following code won't compile
    BoolTest t;
    // without explicit, bool() is called and return 1 to continue
    // with explicit, no implicit conversion occurs
    if (t < 1) {
        
    }

## Type alias ##
key word "using"
    template<typename T>
    struct TestA {
        using value_type = T;
        using ptr_type = T*;
    };
    
    template<typename Key, typename Value>
    struct TestKV {
    };
    //partial specialization
    template<typename Value>
    using str_value_type = TestKV<int, Value>;

    TestA<int>::value_type i;
    TestA<char>::ptr_type p;

## noexcept ##
noexcept will indicate to compiler that this function will not throw any exception. In some case, std::move rather than std::copy will be used for example when a vector reaches its capacity and should reallocate chunck of memory and copy/move objects to new allocated memory. This keyword will make the difference in this case.
by default, generated copy/move ctor/assignment will have "noexcept".

## default ##
use implementation generated by compiler, no explicit impl required.

## delete ##
any invoke to this function is invalid.
struct OnlyDouble {
    template<class T> void f(T) = delete;
    void f(double d) {
    }
};
OnlyDouble d;
d.f(1.0d);
//d.f(1.0f); //this won't compile

## std::reference_wrapper ##
(std::ref, std::cref)
It facilitates pass-by-reference in a pass-by-value context:

template<typename T>
void test(T t) {
    t++;
}
    int value = 1;
    test(std::ref(value));
    std::cout << value << std::endl;//output: 2
    
example in std::thread:
int value = 1;
void start_thread(int& param) {
    std::cout << param++ << std::endl;
}
    std::thread mythread(start_thread, std::ref(value));// value = 2 after this
    mythread.join();

## std::alignof ##
## std::decay_t ##
## std::declval ##
return a rvalue reference to what type passed
useful when no chance to create an object but able to access member function in context of std::decltype
template<typename T, typename U>
using sum_type = std::decltype(std::declval<T>() + std::declval<U>());

## std::common_type ##
#include <type_traits>
c++11, (typename) std::common_type<T, U>::type
c++14, std::common_type_t<T, U>

## std::tuple ##
group of multiple variable:
#include <tuple>
auto mytuple = std::make_tuple("Tom", 30, "American", 'A');
std::cout << std::get<0>(mytuple) << " " << std::get<1>(mytuple) << std::endl;
std::get<3>(mytuple) = 'C';
std::cout << std::get<3>(mytuple) << std::endl;

std::string name, nation;
int age;
std::tie(name, age, nation, std::ignore) = mytuple;
std::cout << "name = " << name << " age = " << age << " nation = " << nation << std::endl;
