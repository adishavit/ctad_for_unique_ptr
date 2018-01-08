*Document number: --   
Date: 2018-??-??    
Audience: Library Working Group   
Reply-To: Adi Shavit `<my-email>`*  

---

# Allowing CTAD for `unique_ptr` with Custom Deleter

## Motivation

Consider the following example:
```
std::unique_ptr<HANDLE, decltype(CloseHandle)> fh0{ CreateFile(...), CloseHandle }; // this is okay
std::unique_ptr fh1{ CreateFile(...), CloseHandle };                                // this is ill-formed
```

The second expression is ill-formed due to the wording in [**[unique.ptr.single.ctor]/16**](http://eel.is/c++draft/unique.ptr#single.ctor-16):  

> *Remarks:* If class template argument deduction ([over.match.class.deduct]) would select a function template corresponding to either of these constructors, then the program is ill-formed.

but the first expression is fine because the `unique_ptr` the class template arguments are specified explicitly.  
The type `HANDLE` of the return value of `CreateFile()` must be explicitely specified *and* the deleter must be mentioned twice, once with `decltype()` and once when passed as the 2nd constructor argument - the custom deleter.     

When used with lambdas the code is even more verbose. Consider the following example:  

```
auto rel_img = [](IplImage* p){ ::cvReleaseImage(&p); };                                // must name lambda
std::unique_ptr<IplImage, decltype(rel_img)> img0{ ::cvCreateImage(...), rel_img };     // this is okay  
std::unique_ptr img1{ ::cvCreateImage(...), [](IplImage* p){ ::cvReleaseImage(&p); } }; // this is ill-formed  
```
Every lambda gets a unique type, so using a lambda as a deleter requires naming it to make it usable both as the class template argument (via `decltype()`) and as the constructor argument.  
It is currently not possible to use in-line unnamed lambdas as custom deleters.  

There is no class template argument deduction from a raw pointer type because it is impossible to distinguish a pointer obtained from array and non-array forms of new. This may be true in the general case, and may lead to undefined-behavior if the wrong default deleter is called (`delete` or `delete[]`) or when providing `operator[]` to a non-array pointer type. However, when the user is providing a custom deleter, this custom deleter will be called unconditionally regardless of any ambiguity in the default deleter case.    

It seems that **[unique.ptr.single.ctor]/16** is overly strict.

This use case is basically the whole point of having `unique_ptr` support deleters and the current wording makes the code cumbersome and error prone.

## Proposal

*Formal wording TBD*

Relax the requirements of [unique.ptr.single.ctor]/16 to allow class template argument deduction in the case of a raw pointer and a user provided deleter.  

Add the following two deduction guides: 
```
template<typename T, typename D>
unique_ptr(T* p, D d)->unique_ptr<T, D>;

template<class T, class D> 
unique_ptr(T, D)->unique_ptr<typename pointer_traits<typename D::pointer>::element_type, D>;
```

In this case the raw pointer type `T*` shall default to a *non-array* pointer behavior and `operator[]` shall be *not* generated.

For raw _array pointers_, the user will *still* have to explicitly specify the template arguments as before if `operator[]` support is desired.


## Discussion

In [P0433R1 §20.11 [smartptr]](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0433r1.html), the authors propose adding various deduction guides for `unique_ptr`, `shared_ptr` and `weak_ptr`. The proposed guides for `unique_ptr` also support the use case where a custom deleter is *not* specified. In that case the guide defaults to *non-array* raw pointers. As discussed above this might lead to undefined behaviour because the non-array version of `delete` will be applied to the array. This part of the propsal [was removed from newer revisions](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0433r2.html) of the proposal precisely for this reason. 
> _The downside is that it will fail to deduce an array type ... We choose not to allow this problem, so we require that such deductions be ill-formed._  

The *current* proposal is limited only to the case where a custom deleter is supplied where UB is not possible. 

P0433R1 proposes a 3rd `unique_ptr` deduction guide:  
`template<class U, class V> unique_ptr(U, V) -> unique_ptr<typename pointer_traits<V::pointer>::element_type, V>;`  
This will not affect the case of using lambdas as deleters. Further examination of this guide is TBD before submission.

Finally, note that the current proposal is not so pertinent to `shared_ptr<>` since the Deleter type is _not_ part of the `shared_ptr<>` type. 


#### Desirata

It would have been desirable if one could "cast" the raw pointer `T* p` to the *incomplete type* `T[]` such that the following would generate the array form of `unique_ptr<T[],D>` with `operator[]`:

```
unique_ptr up{ as_array(p), del };          // std::as_const() / std::move() form 
unique_ptr up{ array_cast<int[]>(p), del }; // alternative casting style form
up[0] = 42;
```
Alas, as far as I understand, this is not currenty possible in the language and is beyond the scope of this proposal.

A related "ommision" is `make_unique()` which does not currently have a form supporting custom deleters at all.    
If and when such a form will be added it would presumably support automatic type deduction. Such alternative `make_unique()` forms are also beyond the scope of this proposal.   


## Acknowledgements

I'd like to thank Agustín Bergé, Peter Dimov and Timur Doumler for very fruitful discussions and suggesting that this should be a proposal.
