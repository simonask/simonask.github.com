Option Types in C++11
=====================

One of the most frequent sources of crashes in C/C++ programs is the lack of a NULL pointer check in a place where a pointer turns out to actually be NULL. On the other hand, it is quite handy to have a default way of saying "no value" in many situations.

In C and C++, the usual way of designing an API that needs to be able to return either a value or nothing is to return an `std::unique_ptr` or an `std::shared_ptr`, often containing a pointer to a heap-allocated object that will be auto-released when the smart pointer goes out of scope. This has the obvious drawback of being unable to return a value without heap-allocating it, which can be prohibitively slow in some circumstances, and it has the additional drawback of needing the API user to check the return value for NULL before dereferencing it.

In functional languages, these problems are solved by using [Option types](http://en.wikipedia.org/wiki/Option_type), which are types that represent either a value or nothing. Learning from the wisdom of functional programming, why don't we make a habit of using the exact same pattern in C++? Using the C++ type system, we can automatically guard against NULL pointer dereferences.

Introducing `Maybe<T>`
-------------------

We can create a class that wraps any type `T` by value, instead of wrapping a pointer to `T`. Using C++11 move semantics, we can eliminate the copying that would otherwise slow down such a solution.

Here is a simple declaration for such an implementation:

    template <typename T>
    class Maybe {
    public:
    	Maybe() : ptr_(nullptr) {}
    	Maybe(const Maybe<T>& other);
    	Maybe(Maybe<T>&& other);
    	Maybe(const T& other);
    	Maybe(T&& other);
    	~Maybe();
    	Maybe<T>& operator=(const Maybe<T>& other);
    	Maybe<T>& operator=(Maybe<T>&& other);
    	Maybe<T>& operator=(const T& other);
    	Maybe<T>& operator=(T&& other);

    	void clear();
    	T* get();
    	const T* get() const;
    	T* operator->();
    	const T* operator->() const;
    	T& operator*();
    	const T& operator*() const;
    	
    	explicit operator bool() const { return get() != nullptr; }
    private:
    	T* ptr_;
    	
    	struct Placeholder {
    		byte _[sizeof(T)];
    	};
    	Placeholder memory_;
    };

There are a few things that should be noted about this class:

1. It provides explicit move constructors and move assignment operators.

2. It assumes that its own const-ness applies to the inner type as well (i.e., getting the contents of a `const Maybe<T>` results in a `const T`).

3. It uses a chunk of placeholder memory allocated next to the object itself, so as to avoid heap allocation. It is important that this memory isn't just a member of type `T`, because all types may not have default constructors, and we do not want an empty `Maybe<T>` to actually call the constructor of `T`.

4. Whether or not the object has a value is determined by whether or not `ptr_ == nullptr`. When the object has a value, `ptr_` points to the memory contained in the placeholder.

5. Note the use of an explicit conversion operator. Prior to C++11, providing conversion operators for `bool` was considered unsafe, because `bool` can implicitly be converted to other integer types, and that could result in unwanted funky situations. In C++11, the `explicit` keyword can be applied to conversion operators to prevent implicit conversion to other types than `bool`.

Accessing the placeholder memory
--------------------------------

As a convenience, we can create a function to access the placeholder memory. Note that calling `memory()` on an empty object will return a pointer to uninitialized memory, and not a valid instance of `T`. Therefore, this function is private.

    private:
    T* memory() { return reinterpret_cast<T*>(&memory_); }

Replicating the semantics of the inner type
-------------------------------------------

In order to implement the different constructors and operators, we will need a series of assignment functions. The type `T` may be any combination of copy-assignable, move-assignable, copy-constructible, or move-constructible, and we want the `Maybe` type to reflect these semantics as well as it can. Using `std::enable_if` (new in C++11, identical to `boost::enable_if`) and type traits, we can create a series of assignment functions which our constructors and assignment operators can call, selecting the proper version depending on the available constructors and operators on the type `T`. Using `Maybe<T>` in a way that the contained instance of `T` cannot accomodate results in a compilation error.

See [maybe.hpp from my project *Reflect*](https://github.com/simonask/reflect/blob/master/maybe.hpp#L49) for an implementation of these assignment functions.

Securing against NULL pointer dereferencing
-------------------------------------------

So far, we have a class that may or may not contain a value. It can be used like this:

    Maybe<int> m;
    if (!m) {
      m = 123;
      std::cout << "My number: " << *m << '\n';
      m.clear();
    }

We haven't solved the real problem, though. Right now, the API user still has to check if `m` actually has a value before trying to get that value, and if she forgets, she will still get a crash.

But there is a way to get around that. Using C++11 lambda functions, we can implement a function that does the NULL pointer check for us, and then lock the user out from accessing the pointer directly. When we're done, it will look something like this:

    Maybe<int> m = 123;
    maybe_if(m, [](int number) {
      std::cout << number << '\n';
    });

The difference between this code and the above is subtle, but important. The lambda closure is only executed when `m` contains an int, but the important idea here is that the *only* way to access the int is through this guard function, which takes care of the dereferencing.

For more complex types, we may wish to implement `maybe_if` so it can pass a reference of the value instead of a copy:

    Maybe<std::vector<int>> m = some_function_that_may_or_may_not_generate_a_list_of_numbers();
    maybe_if(m, [](const std::vector<int>& list_of_numbers) {
      for (auto number: list_of_numbers) {
        std::cout << number << '\n';
      }
    });

Using it this way, no copies are made of the vector object at any point, and it is still only safely accessed.

Implementing `maybe_if`
-----------------------

C++ template syntax has never been known for its readability, so I'll try to do my best explaining what goes on.

The central function is this:

    template <typename T, typename Functor>
    typename MaybeIf<T,Functor>::ResultType
    maybe_if(Maybe<T>& maybe, Functor function) {
    	return MaybeIf<T,Functor>::maybe_if(maybe, function);
    }

    template <typename T, typename Functor>
    typename MaybeIf<T,Functor>::ResultType
    maybe_if(const Maybe<T>& maybe, Functor function) {
    	return MaybeIf<T,Functor>::maybe_if(maybe, function);
    }

Because we decided that the const-ness of `Maybe<T>` is also the const-ness of `T`, we need both versions of this function. Because lambdas are allowed to have the `void` return type, and C++ disallowed returning a value from a function with the `void` return type, we need to provide a special case for when the lambda function has return type `void`. Because C++ disallows function template specialization, this in turn means that we have to wrap the real guts of `maybe_if` in a struct type, which *can* be partially specialized.

The `MaybeIf` struct is implemented like this:

    template <typename T, typename Functor>
    struct MaybeIf {
    	typedef typename std::result_of<Functor(T&)>::type ReturnType;
    	typedef MaybeIfImpl<T, ReturnType, Functor> Impl;
    	typedef typename Impl::ResultType ResultType;
	
    	static ResultType maybe_if(Maybe<T>& maybe, Functor function) {
    		return Impl::maybe_if(maybe, function);
    	}
	
    	static ResultType maybe_if(const Maybe<T>& maybe, Functor function) {
    		return Impl::maybe_if(maybe, function);
    	}
    };

This level exists for the deduction of the return type of the provided lambda function, using `std::result_of`. The `void` return type specialization hasn't happened yet, but is invoked from here. There is one important detail here: Note the '`&`' in `std::result_of<Functor(T&)>` -- this is because we want to allow the users of `maybe_if` to modify the internal object, and `T&` can degrade to both `T` (by-value), `const T&`, and can be turned into `T&&` by using `std::move` in the provided lambda. Without the '`&`', only lambdas taking `T` or `const T&` as their arguments could be used with `maybe_if`.

The `MaybeIfImpl` struct is implemented like this:

    template <typename T, typename R, typename Functor>
    struct MaybeIfImpl;

    template <typename T, typename Functor>
    struct MaybeIfImpl<T, void, Functor> {
    	typedef bool ResultType;
	
    	static bool maybe_if(Maybe<T>& maybe, Functor function) {
    		if (maybe) { function(*maybe); return true; }
    		return false;
    	}
    	static bool maybe_if(const Maybe<T>& maybe, Functor function) {
    		if (maybe) { function(*maybe); return true; }
    		return false;
    	}
    };

    template <typename T, typename R, typename Functor>
    struct MaybeIfImpl {
    	typedef typename RemoveMaybe<R>::Type ReturnType;
    	typedef Maybe<ReturnType> ResultType;
	
    	static ResultType maybe_if(Maybe<T>& maybe, Functor function) {
    		if (maybe) return function(*maybe);
    		return ResultType();
    	}
	
    	static ResultType maybe_if(const Maybe<T>& maybe, Functor function) {
    		if (maybe) return function(*maybe);
    		return ResultType();
    	}
    };

This is where the specialization for lambdas returning `void` happens. There are some interesting things to note about this:

1. The result of `maybe_if` is normally `Maybe<R>`, where `R` is the return type of the provided lambda. If the lambda has `void` return type, the result of `maybe_if` is a boolean value indicating whether or not the lambda executed.

2. If the lambda returns a `Maybe<Foo>`, the result of `maybe_if` would naïvely be `Maybe<Maybe<Foo>>`, but that's not something we're very interested in, so we must define a `RemoveMaybe<T>` type that can tell us the inner type of a `Maybe` type, or just pass along `T`:

`RemoveMaybe<T>`:

      template <typename T>
      struct RemoveMaybe;
      template <typename T>
      struct RemoveMaybe<Maybe<T>> {
      	typedef T Type;
      };
      template <typename T>
      struct RemoveMaybe {
      	typedef T Type;
      };

The last missing piece is securing `Maybe<T>` against direct manipulation of its inner memory. This can be achieved by marking the getters private, and declaring `MaybeIfImpl` friend. Our final class declaration then looks like this:
  
    template <typename T>
    class Maybe {
    public:
    	Maybe() : ptr_(nullptr) {}
    	Maybe(const Maybe<T>& other);
    	Maybe(Maybe<T>&& other);
    	Maybe(const T& other);
    	Maybe(T&& other);
    	~Maybe() { clear(); }
    	Maybe<T>& operator=(const Maybe<T>& other);
    	Maybe<T>& operator=(Maybe<T>&& other);
    	Maybe<T>& operator=(const T& other);
    	Maybe<T>& operator=(T&& other);

    	void clear();
    	
    	explicit operator bool() const { return get() != nullptr; }
    private:
    	template <typename U, typename R, typename Functor> friend struct MaybeIfImpl;

    	const T* get() const { return ptr_; }
    	T* get() { return ptr_; }
    	T* operator->() { return get(); }
    	const T* operator->() const { return get(); }
    	T& operator*() { return *get(); }
    	const T& operator*() const { return *get(); }
    	
    	T* ptr_;
    	struct Placeholder {
    		byte _[sizeof(T)];
    	};
    	Placeholder memory_;

    	T* memory() { return reinterpret_cast<T*>(&memory_); }
      
      // assigners here...
    };


All done
--------

And that's it! I recommend taking a look at [a full implementation of Maybe<T>](https://github.com/simonask/reflect/blob/master/maybe.hpp), to get a better idea of some of the more low-level details dealing with C++'s syntactic idiosyncracies.

In my implementation, I have also provided an `otherwise` function, because "if-else" is a common thing to need. The API looks like this:

    Maybe<int> m = 123;
    maybe_if(m, [](int) {
      // there is a value
    }).otherwise([]() {
      // there is no value
    });

Reflecting on the insight provided by this implementation, it is worth noting that there is no reason that the `maybe_if` pattern couldn't be applied to all nullable types. The implementation could be easily generalized such that `maybe_if` could take a pointer of any type, check whether it's NULL, and then pass it on to a lambda function if not. The key to the option type pattern, though, is the inability to access objects that we aren't 100% sure are initialized with something valid.