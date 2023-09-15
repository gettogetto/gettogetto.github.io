```
#include <tuple>
#include <concepts> 

template<auto x> 
consteval auto _int_tuple(auto ...p) 
{ 
    if constexpr (sizeof...(p) == x) 
        return std::tuple{ p... }; 
    else 
        return _int_tuple<x>(0, p...); 
} 

template<auto x> using int_tuple = decltype(_int_tuple<x>()); 

auto main()->int 
{ 
    static_assert(std::same_as<int_tuple<1>, std::tuple<int>>);
    static_assert(std::same_as<int_tuple<3>, std::tuple<int, int, int>>); 
}


```

[[C++]]