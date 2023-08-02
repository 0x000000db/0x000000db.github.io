---
title: "Crash when using `shared_from_this` when not declared public inheritance"
date: 2023-07-28T16:08:17+01:00
draft: false
slug: why-does-shared-from-this-crash-with-private-inheritance
---
# Introduction
I made a simple mistake of inheriting `std::enable_shared_from_this<T>` but not specifying the inheritance type. But using shared_from_this results in a crash, why is that? I'd like to point out that since it does not work at all it's most likely not really going to pop up during production and more likely confusion during development, however it's still interesting, let's delve. I've noticed this both on Ubuntu and Mac so far. Calling the method test on the following class is enough to reproduce:
```
struct Cls : std::enable_shared_from_this<Cls> {
public:
    void test() {
        auto self = shared_from_this();
    }
};
```
The error is `bad_weak_ptr` which is typically something you see when mishandling shared_ptr and the object had actually been destroyed. 

# Code dive
We can look to the gcc libc++ source code for further information, which can be found (here)[https://github.com/gcc-mirror/gcc/blob/b2cfe5233e682fc04a9b6fc91f3d30685515630b/libstdc%2B%2B-v3/include/bits/shared_ptr.h#L919]. We can see that internally, this class keeps a weak_ptr to itself (`_M_weak_this`), but why would changing inheritance type break this? We can theorize that since it is broken using private inheritance that it relys on a public method (on your class) being called and it seems plausible given the constructors do not appear to set anything. So what does `shared_from_this` do? It appears that the functionality is enabled by an external partner, we can see a suspect in `shared_ptr_base`
```
      template<typename _Yp, typename _Yp2 = typename remove_cv<_Yp>::type>
	typename enable_if<__has_esft_base<_Yp2>::value>::type
	_M_enable_shared_from_this_with(_Yp* __p) noexcept
	{
	  if (auto __base = __enable_shared_from_this_base(_M_refcount, __p))
	    __base->_M_weak_assign(const_cast<_Yp2*>(__p), _M_refcount);
	}
```
It appears that the `enable_shared_from_this::_M_weak_this` is assigned here, perhaps this code was never run due to SFINAE because `__enable_shared_from_this_base` is not accessible. There is a comment above this function `// Found by ADL when this is an associated class.` - I don't know the reason for sure but it seems like a plausible explanation. It does elude in the docs that public inheritance is a requirement but it's a shame there is a way to use it incorrectly! 