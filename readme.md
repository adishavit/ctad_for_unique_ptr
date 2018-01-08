*Document number: -- 
Date: 2018-01-08  
Audience: Library Working Group
Reply-To: Adi Shavit adishavit@gmail.com*

---
# Allowing CTAD for `unique_ptr` with Custom Deleter

## Motivation

Consider the following example:
```
std::unique_ptr<HANDLE, decltype(CloseHandle)> fh0{ CreateFile(), CloseHandle }; // this is okay
std::unique_ptr fh1{ CreateFile(), CloseHandle };                                // this is ill-formed
```

The second expression is ill-formed due to the wording in [unique.ptr.single.ctor]/16:  

> *Remarks:* If class template argument deduction ([over.match.class.deduct]) would select a function template corresponding to either of these constructors, then the program is ill-formed.

but the first expression is fine because `unique_ptr` the class template arguments are specified explicitly.  
The type `HANDLE` of the return value of `CreateFile()` must be explicitely specified *and* the deleter must be mentioned twice, once with `decltype()` and once when passed as the 2nd constructor argument - the custom deleter.     

When used with lambdas the code is even more verbose. Consider the following example:  

```
auto rel_img = [](IplImage* p){ ::cvReleaseImage(&p); };                                // must name lambda
std::unique_ptr<IplImage, decltype(rel_img)> img0{ ::cvCreateImage(...), rel_img };     // this is okay  
std::unique_ptr img1{ ::cvCreateImage(...), [](IplImage* p){ ::cvReleaseImage(&p); } }; // this is ill-formed  
```
Every lambda gets a unique type, so using a lambda as a deleter requires naming it so it can be used both as the class template argument (via `decltype()`) and also as the constructor argument.  
It is currently not possible to use in-line unnamed lambdas as custom deleters.  

According to [cppreference](http://en.cppreference.com/w/cpp/memory/unique_ptr/unique_ptr):

> _There is no class template argument deduction from pointer type because it is impossible to distinguish a pointer obtained from array and non-array forms of new._

In the general case, this may be true and lead to undefined-behavior if the wrong default deleter is called (`delete` or `delete[]`) or when providing `operator[]` to a non-array pointer type. However, when the user is providing a custom deleter, this deleter is called regardless of the ambiguity in the default deleter case.    

[unique.ptr.single.ctor]/16 seems to be overly strict.

Essentially, this use case is basically the whole point of having `unique_ptr` support deleters and the current wording makes this cumbersome and error prone.

## Proposal

*Formal wording TBD*

Relax the requirements of [unique.ptr.single.ctor]/16 to allow class template argument deduction in the case of a raw pointer and a user provided deleter.  

Add a deduction guideline: `template<typename T, typename D> unique_ptr(T* p, D d) -> unique_ptr<T,D>;`

In this case the raw pointer type `T*` shall default to a *non-array* pointer and `operator[]` shall be *not* generated.

For raw array pointers, the user will still have to explicitly specify the template arguments as before.

## Discussion

It would have been desirable if one could "cast" the raw pointer `T* p` to the *incomplete type* `T[]` such that the following would generate the array form of `unique_ptr<>` with `operator[]`:

```
unique_ptr up{ as_array(p), del };          // std::as_const() / std::move() form 
unique_ptr up{ array_cast<int[]>(p), del }; // alternative casting style form
up[0] = 42;
```
Alas, as far as I understand, this is not currenty possible in the language and is beyond the scope of this proposal.

A related "ommision" is `make_unique()` which does not currently have a form supporting custom deleters at all.    
If and when such a form will be added it would presumable support automatic type deduction. Such alternative `make_unique()` forms are also beyond the scope of this proposal.   


## Acknowledgements

Thanks to Agustín Bergé, Peter Dimov and Timur Doumler for suggesting that this should be a proposal.