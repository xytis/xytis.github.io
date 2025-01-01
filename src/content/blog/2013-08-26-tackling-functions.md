---
title: "Tackling Functions"
description: "Trying to understand function optimisations in c++"
date: 2013-08-26
categories: [cpp, cpp11, templates]
---

Recently I've found [this post](http://probablydance.com/2013/01/13/a-faster-implementation-of-stdfunction/), which describes a better implementation of std::functions. Since I had severe problems with understanding how this code works, I decided to write down my adventures in it step by step.

#### Syntax

The code in question provides an alternate implementation of std::function, so at least it should make such code work:
``` cpp
int foo(int a) {};
func::function<int(int)> f(foo);
f(5);
```

I started analyzing this code by searching for a definition of `function`. After skipping compiler specific workarounds and macro definitions we arrive at first meaningful statement:
``` cpp
template<typename>
class function;
```

Great, we have a forward declaration of template class `function` which takes one parameter. For the time being lets skip the `namespace detail` and arrive at a partial instantiation of this template:
``` cpp
template<typename Result, typename... Arguments>
class function<Result (Arguments...)>
```

Now this is something I did not know before: You may define a partial template instantiation, by using a template. Here `function<template T>` type `T` is specialized as `Result (Arguments...)`, where `Result` and `Arguments...` are both template types. Why? Well You get this nice syntax support (`function<void(int)>`) where compiler fills the `Result` and `Arguments...` with appropriate types.

Since we skipped the `detail`, lets ignore the parent class too, we will come back to it later.
Next thing I jumped to is the initialization:
``` cpp
function();                             //[1]
function(std::nullptr_t);               //[2]
function(function && other);            //[3]
function(const function & other);       //[4]

template<typename T>
function(T functor,
  typename std::enable_if<detail::is_valid_function_argument<T, Result (Arguments...)>::value, detail::empty_struct>::type = detail::empty_struct());   //[5]
```

Later constructors are the same as these, only they take user defined allocator as a parameter.
First constructor is obvious, it creates an empty function. Second overload is a new feature of cpp11, this constructor is called if user passes nullptr as the argument. Note: standard define `NULL 0` will not trigger it. Third constructor is also a new feature -- move constructor, which works as copy constructor, but avoids allocating any resources by 'stealing' them from given argument (which always should be a temporary rvalue). Fourth -- simple copy constructor.

Now the most interesting is the fifth constructor. It is triggered by `func::function<int(int)> f(foo)` initialization, note that the second parameter has a default value `detail::empty_struct()` and as far as I can tell You will never explicitly specify it with any other value. But why should it have another parameter in the first place? The construct `typename std::enable_if<...>::type` should deny template instantiation by raising a compile time error (the specialization `std::enable_if<false, T>` has no field `type`) if given type `T` does not match some sort of requirements. This works partially because of function precedence, since compiler when resolving overloaded functions will prefer non-template functions to template ones, but if an implicit cast must be made to the argument, template functions are considered before doing the casts. So the template constructor is considered only if all above constructors fail to match, so in case `std::enable_if<>::type` does not exist, raised error means that function can not be binded (template argument mismatch or given argument can not be converted to a functor).

Lets look at requirements imposed on template constructor: `detail::is_valid_function_argument<T, Result (Arguments...)>::value`. As the name implies -- this code should check if `T` is a function-like type, which can take `Arguments...` and return `Result`. This check is quite complicated since type `T` may be a `struct` or a member function, so that compiler is unable to match it to `Result (Arguments...)` pattern implicitly.
Check is declared in `namespace detail` as follows:

``` cpp
template<typename, typename>
struct is_valid_function_argument
{
  static const bool value = false;
};

template<typename Result, typename... Arguments>
struct is_valid_function_argument<function<Result (Arguments...)>, Result (Arguments...)>
{
  static const bool value = false;
};

template<typename T, typename Result, typename... Arguments>
struct is_valid_function_argument<T, Result (Arguments...)>
{
#ifdef _MSC_VER
  // as of january 2013 visual studio doesn't support the SFINAE below
  static const bool value = true;
#else
  template<typename U>
  static decltype(to_functor(std::declval<U>())(std::declval<Arguments>()...)) check(U *);
  template<typename>
  static empty_struct check(...);

  static const bool value = std::is_convertible<decltype(check<T>(nullptr)), Result>::value;
#endif
};
```
Compiler (as far as I know) always matches the strictest definition first, so the first definition works as a trap for everything that is not a valid function argument. Second specialization is aimed at ruling out template instantiation for function object. I am not sure why this template should ever be picked, since copy constructor should pick out functions that have the same type signature.

Now the final specialization (again, note the templates in specialization) actually does some checking:
``` cpp
static const bool value = std::is_convertible<decltype(check<T>(nullptr)), Result>::value;
```
First, a quick detour to wikipedia for meaning of [SFINAE](http://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error): If compiler encounters a substitution error while resolving a template instantiation, it does not stop with an error and continues to search for a valid candidate. In this case it means that in case of error in first instantiation of `check` function, compiler will choose second overload. Also note the `...` in the second template definition. As wiki says:
    An ellipsis is used not only because it will accept any argument, but also because its conversion rank is lowest, so a call to the first function will be preferred if it is possible; this removes ambiguity.
So it simply reassures that compiler will apply any tricks, implicit conversions, or things I have no idea about in order to match the first definition.
The checking is done in two parts -- one that checks if given object is a functor of some sorts, and another, which checks if return type is valid and can be treated as `Result`.
Lets start with the latter (even if the compiler goes the other way around). Call to `decltype(check<T>(nullptr))` evaluates the type of the expression without actually evaluating the expression. So when the function `check<T>(nullptr)` is matched with first definition -- this evaluates to whatever the type `T` may return on when evaluating code `t(arguments...)`, where `t` is an instance of `T` and `arguments` is the list of instances of types `Arguments`. The constrain for return value is set as `std::is_convertible<>` which can be really useful when dealing with polymorphic overloads (class members, which return different polymorphic objects) like such:
``` cpp
class A {
  virtual A * me();
}

class B : public A {
  virtual B * me();
}
```

Obviously if the second definition of `check` is chosen -- `decltype(check<T>(nullptr)` will be resolved as `func::detail::empty_struct` which is pretty much not convertible to anything else than itself and it is meaningless to have a function which returns this specific struct, unless You explicitly want to break something.

One thing remains -- why the first definition uses `to_functor(std::declval<U>())(std::declval<Arguments>()...)` instead of plain `std::declval<U>()(std::declval<Arguments>()...)`?
Unfortunately I have no idea. Here is the definition of `to_functor`, perhaps later I or someone else will explain this.

``` cpp
template<typename T>
T to_functor(T && func)
{
  return FUNC_FORWARD(T, func);
}
template<typename Result, typename Class, typename... Arguments>
auto to_functor(Result (Class::*func)(Arguments...)) -> decltype(std::mem_fn(func))
{
  return std::mem_fn(func);
}
template<typename Result, typename Class, typename... Arguments>
auto to_functor(Result (Class::*func)(Arguments...) const) -> decltype(std::mem_fn(func))
{
  return std::mem_fn(func);
}
```

Now if we try such code:
``` cpp
int goo() {}
func::function<void(void)> g(goo);
```
It fails with abundance of errors, one of which actually is induced by our explored snippet:
``` bash
function.h:420:5: error: no type named ‘type’ in ‘struct std::enable_if<false, func::detail::empty_struct>’
```

So far so good, basically all the code I have analyzed up to this point is used to enforce type safety of this final constructor, but we haven't even touched the real code behind this implementation.


#### Storage

Body of the constructor brings us to another topic: How function object is saved within func::function?


``` cpp
  detail::manager_storage_type manager_storage;
  Result (*call)(const detail::functor_padding &, Arguments...);

  template<typename T, typename Allocator>
  void initialize(T functor, Allocator && allocator)
  {
    call = &detail::function_manager_inplace_specialization<T, Allocator>::template call<Result, Arguments...>;
    detail::create_manager<T, Allocator>(manager_storage, FUNC_FORWARD(Allocator, allocator));
    detail::function_manager_inplace_specialization<T, Allocator>::store_functor(manager_storage, FUNC_FORWARD(T, functor));
  }

  typedef Result(*Empty_Function_Type)(Arguments...);
  void initialize_empty() FUNC_NOEXCEPT
  {
    typedef std::allocator<Empty_Function_Type> Allocator;
    static_assert(detail::is_inplace_allocated<Empty_Function_Type, Allocator>::value, "The empty function should benefit from small functor optimization");

    detail::create_manager<Empty_Function_Type, Allocator>(manager_storage, Allocator());
    detail::function_manager_inplace_specialization<Empty_Function_Type, Allocator>::store_functor(manager_storage, nullptr);
    #               ifdef FUNC_NO_EXCEPTIONS
    call = nullptr;
    #               else
    call = &detail::empty_call<Result, Arguments...>;
    #               endif
  }
```
This class has only two private members: `manager_storage` -- some sort of object, which is defined in `detail` namespace and `call` -- function pointer which takes additional parameter `function_padding`.

If we look at both initializations we clearly see that `call` always points to a separate template instantiation of a function, dependent on `T` -- the parameter obtained on initialization and `Result` and `Arguments...` -- signature of this function. Unless it points to dummy function `empty_call`, which is self explanatory. The actual code executed by `call` is a static member function, which casts given argument of `functor_padding` to type `T &`, tries to use `()` operator on it with forwarded arguments and returns `Result` type object. This little trick with specific template instantiation binds the types during compile time at a cost of code bloat.

 The `call` function is triggered by `operator ()` with an argument of `manager_storage.functor`. Presumably a pointer could be stored in this field. But it gets more interesting here, because of 'small functor optimization'.

*** To be continued... ***

