---
title: "Fun with flags"
description: "Playing with type safe enums and constexpr"
date: 2013-10-11
categories: [cpp, cpp11, templates]
---

Every now and then I catch myself in such situation:

``` cpp
enum Alignment {
  Left, Center, Right, Justify,
};

enum Anchor {
  Top, Right, Bottom, Left,
};

void SetAnchor(byte anchor);
void SetAlignment(byte alignment);
```

This just begs for errors. So what can we do here? Obvious answer -- lets make a class, with overloaded `operator|` and _voilia_, we just created a type safe flag implementation.
This time I wanted to do something different, in a way, that performance of these flags would match (in some extent) natural binary operations.

So the goals for today:

*   Syntax:

    ``` cpp
    auto alignment = Alignment::Justify | Alignment::Right;
    SetAnchor(Anchor::Top | Anchor::Right);
    ```

*   Speed:

    Should be resolved during compile time most of the time and with the same speed as std::bitset when performing in runtime.

*   Ease of defining:

    I liked the way [Molecular](http://molecularmusings.wordpress.com/2011/08/23/flags-on-steroids) defined macros for their flag creation, so I shall do the same.


Note:
  Code for this post is stored at [GitHub](https://github.com/xytis/mad-flag)

Since I want compile time evaluation, I must somehow store the expressions `a | b | c` as types. Naturally this can be achieved using lists: `a | (b | c)`, where `b | c` creates a compound type.

``` cpp
  template<typename T, size_t pos, bool state = true>
  struct flag {
    constexpr flag() {}
  };

  template<typename T, typename lhs_t, typename rhs_t>
  struct flag_group {
    lhs_t & lhs;
    rhs_t & rhs;
  };
```

These types can be nested using these operations:
``` cpp
template<...>
constexpr auto operator | (left_flag_t<T, ...> lhs, right_flag_t<U, ...> rhs) -> flag_group<T, decltype(lhs), decltype(rhs)> {
  static_assert(std::is_same<T,U>::value, "Flag type missmatch!");
  return {lhs, rhs};
}

template<...>
constexpr auto operator | (left_group_type<T, ...> lhs, right_flag_t<U, ...> rhs) -> flag_group<T, decltype(lhs), decltype(rhs)> {
  static_assert(std::is_same<T,U>::value, "Flag type missmatch!");
  return {lhs, rhs};
}

template<...>
constexpr auto operator | (left_group_type<T, ...> lhs, right_group_type<U, ...> rhs) -> flag_group<T, decltype(lhs), decltype(rhs)> {
  static_assert(std::is_same<T,U>::value, "Flag type missmatch!");
  return {lhs, rhs};
}
```

Resulting object from operation `a | b | c` contains all expression flags, carefully nested in a compile time defined tree. Now how do we use them in orderly fashion?
``` cpp
namespace Anchor {
    typedef struct{} _guard;
    typedef Flags::field<_guard, 4> State;
    constexpr Flags::flag<_guard, 0> Top;
    constexpr Flags::flag<_guard, 1> Right;
    constexpr Flags::flag<_guard, 2> Bottom;
    constexpr Flags::flag<_guard, 3> Left;
}
```

Here the namespace serves as a syntactic container for flags and for magical guard struct. The type of the struct is unique (as long as there are only a single definition of a namespace named Anchor with an empty struct) and can be used by compiler while resolving templates to imply type safety on the flags.
State typedef in the namespace is used to create flag containers.

After using the same macro magic as in Molecular engine, flags can be used like this:
``` cpp
FLAGS(PlayerState, active, idle, damaged, jumping);

int main ()
{
    using namespace PlayerState;
    State state = State();

    state |= active | jumping | idle;
    std::cout << ToString(state) << std::endl;

    auto argument = damaged | !jumping;
    state |= argument;
    std::cout << ToString(state) << std::endl;

    return 0;
}
```
