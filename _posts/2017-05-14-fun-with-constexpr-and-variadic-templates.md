---
layout: post
title: ["Fun with C++ constexpr, variadic templates, and fold expressions"]
date: 2017-05-14
---

I ended up going on a little adventure through a myriad of new features provided by C++11, C++14, and C++17 this past week. I wanted to see how much of a simple bit banging program could be done at compile time instead of runtime. I had to piece together a bunch of different stack overflow posts and pages of [the documentation](http://en.cppreference.com/w/) to grasp some of these things so, really more for myself, I figured I would summarize everything I learned in this post. If anyone else is able to learn something from this, then that's great too.

## Starting in C
I have been playing around a lot with embedded Linux boards like the beaglebone, raspberry pi, etc. I find myself doing a lot of bit manipulation in specific registers or hardware addresses, often in the context of flashing LEDs or just toggling pins with specific duty cycles. This is pretty straightforward using the `/dev/mem` interface provided by Linux. A quick call to [mmap](http://man7.org/linux/man-pages/man2/mmap.2.html) and you're good to go! My (admittedly simple) bitbanging routines usually look somewhat like the following snippet:

```c
// Contrived C routines to toggle some hardware pins
// Not a complete program.
#define BASE_ADDRESS  0x123
#define GPIO_REG      0x456

// my LEDs labeled 0,1,2,etc start at bit 3 of the register
#define OFFSET        3
#define BITMASK(num)  1 << (num + OFFSET)

// some data for 'doing stuff'
// It is set somewhere else in our program
#define MAXBITS 8
int some_data[MAXBITS];

volatile uint32_t* mem_region;

void toggle_on(uint32_t reg_addr, uint32_t mask) {
	mem_region[reg_addr] |= mask;
}

void toggle_off(uint32_t reg_addr, uint32_t mask) {
	mem_region[reg_addr] &= ~mask;
}

// Combined bitmask because I find I am setting these pins together *a lot*
const uint32_t COMBO_MASK = BITMASK(1) | BITMASK(3) | BITMASK(5);

void do_stuff() {
	// Convenience bitmask for toggling everything at once
	// I could set this up as a constant like COMBO_MASK,
	// but that would look ugly and be annoying.
	uint32_t allbits_mask = 0;
	for(int x = 0; x < MAXBITS; x++) {
		allbits_mask |= BITMASK(x);
	}

	// somewhere else we have already opened /dev/mem
	// and mmapped desired segment to mem_region.

	// set up our pins
	toggle_off(GPIO_REG, allbits_mask);
	toggle_on(GPIO_REG, COMBO_MASK);

	// do some stuff with our data and toggle pins accordingly
	for(int x = 0; x < MAXBITS; x++) {
		if(some_data[x] < 50) {
			toggle_off(GPIO_REG, BITMASK(x));
		}
	}
}
```

## Basic C++11 constexpr
I decided to C++-ify things and make use of some of the new features introduced in C++11 like [constexpr](http://en.cppreference.com/w/cpp/language/constexpr). Constexpr provides some pretty neat functionality by telling the compiler "this thing marked constexpr is simple enough to be computed at compile time. If it is used in a compile-time constant, evaluate it then." In C++11 the things that could be done with constexpr were very limited due to some strict rules about what is allowed in a constexpr expression. For example, a constexpr function can consist of only a single return statement. For-loops are out, and likewise for if-else statements that aren't a ternary conditional in that single return statement. Despite these restrictions, the compile-time evaluation of constexpr has the nice side effect of letting me turn all those `#define` macros into fully type-checked constant variables. This wouldn't be too amazing on its own since we always had the `const` keyword for constant variables, however we can start to do fun things like declare arrays using our constexpr variables like so:

```c
constexpr int MAXBITS = 8;
int some_data[MAXBITS];
```

Additionally, constexpr functions are _allowed_ to be to be evaluated in compile time expressions, but they aren't _required_ to be compile-time only. And since they are still functions, you are more than capable of using them just like you would any other function in non-constexpr situations. Thus our convenient `MASK` macro above can be turned into a constexpr function and we no longer have to worry about all those interesting [side effects](https://www.securecoding.cert.org/confluence/display/c/PRE31-C.+Avoid+side+effects+in+arguments+to+unsafe+macros) that can happen when one isn't very careful with preprocessor macros.

So now the top portion of that C code ends up looking something like this in C++11:

```c++
// C++11 version of that simple C snippet
constexpr uint32_t BASE_ADDRESS = 0x123;
constexpr uint32_t GPIO_REG     = 0x456;

// my LEDs labeled 0,1,2,etc start at bit 3 of the register
constexpr uint32_t OFFSET = 3;
constexpr uint32_t bitmask(int num) {
	return 1 << (num + OFFSET);
}

// some data for 'doing stuff'
// It is set somewhere else in our program
constexpr int MAXBITS = 8;
int some_data[MAXBITS];
```

## Variadic Templates and Recursion
Let's take a look at that variable `COMBO_MASK`. It's kind of annoying chaining those macros together by hand, and I am only doing it for 3 bits! Another tool introduced in C++11 that can help us address this is variadic templates. Just like how [printf](http://www.cplusplus.com/reference/cstdio/printf/) and its ilk can take a variable number of arguments, so to can templates in C++11. Additionally, we can use some recursion with our variadic template function to unpack our arguments through successive calls to itself. However, we also will need to provide a base case overload that will end the recursion. Whew! Now that we have that out of the way, lets make a function for those multiple-bit masks using variadic templates:

```c++
// our original bit masking function
constexpr uint32_t singlebit_mask(int num) {
	return 1 << (num + OFFSET);
}

// base case
constexpr uint32_t multibit_mask(int bit) {
	return singlebit_mask(bit);
}

// recursion!
template<typename... Args>
constexpr uint32_t multibit_mask(int bit, Args... args) {
	return singlebit_mask(bit) | multibit_mask(args...);
}

// our combination bitmask
constexpr uint32_t COMBO_MASK = multibit_mask(1, 3, 5);
```

If you haven't already, I highly recommend you read Eli Bendersky's [excellent blog post](http://eli.thegreenplace.net/2014/variadic-templates-in-c/) for a more in depth explanation of variadic templates and what is going on there. In a nutshell when we call `multibit_mask(1,3,5)`, the function recurses, calling `singlebit_mask()` on the first argument and expanding the remaining arguments as the parameters to `multibit_mask()`. When we get to the case where there is only one argument lef in the parameter pack, we hit the base case function overload. I purposely left `singlebit_mask` as its own function because I didn't want to inline the bit shifting by hand in two separate functions and also convolute things further in the variadic template function return statement.

Frankly, I think template syntax can be a little... painful or esoteric at times. Coupled with the recursion, the variadic arguments, and the need for the base case overload, this solution is just a little too messy for me. It's fairly straightforward in our contrived example, but things can get out of hand quickly. So let's explore some other solutions. To do that though, we are going to need to leave C++11 land.

## C++14 and a More Versatile Constexpr {#cpp14-constexpr}
C++14 relaxed a lot of the rules around constexpr. Now constexpr functions are defined by what _isn't_ allowed in them rather than what very specific things are allowed in them. For example, constexpr functions are no longer limited to a just a single return statement and we can now have loops in constexpr functions. Additionally, it is easy for us to unpack variadic template parameter packs using the [range-based for loops](http://en.cppreference.com/w/cpp/language/range-for) and [brace-enclosed initializers](http://en.cppreference.com/w/cpp/language/parameter_pack#Brace-enclosed_initailizers) introduced in C++11. This is great for our bit masking function:

```c++
// our original bit shifting function
constexpr uint32_t singlebit_mask(int num) {
	return 1 << (num + OFFSET);
}

// build up a multi-bit mask
template<typename... Args>
constexpr uint32_t multibit_mask(Args... args) {
	uint32_t mask = 0;
	for(int x : {args...}) {
		mask |= singlebit_mask(x);
	}
	return mask;
}
```

Now that we have managed to simplify the multi-bit masking function down to a single function definition, and since it can also handle masking a single bit, I think we can combine everything nicely like so:

```c++
// build up a multi-bit mask
template<typename... Args>
constexpr uint32_t multibit_mask(Args... args) {
	uint32_t mask = 0;
	for(int x : {args...}) {
		mask |= 1 << (x + OFFSET);
	}
	return mask;
}
```

This is by far my favorite solution. Once you have a basic undertanding of the parameter packing and unpacking syntax, I think it is actually pretty straighforward to see what is going on there. However, unfortunately for me this solution was the last thing I discovered while I was trying to learn constexpr functions and variadic templates at the same time. For completeness sake I am going to show some other ways we can go about this problem.


## C++17 and Fold Expressions
You knew this one was coming from the title of the post. C++17 introduced [fold expressions](http://en.cppreference.com/w/cpp/language/fold). [Folds](https://en.wikipedia.org/wiki/Fold_(higher-order_function)) are a concept common in function programming and involve recursive operations on a data structure (There's more to them than that, but that's all I am going to mention here).

_Side Note_: I am going to front load this section by stating that although this is a pretty neat concept, I am not a fan of the solution for this particular instance. Between C++17 not being a completed standard[\[1\]](#footnotes), compiler support being limited[\[2\]](#footnotes), and the more complicated syntax we end up with for our simple example, I think that our [final program](#final-program) is best served with the range-based for constexpr solution [above](#cpp14-constexpr). However, with that said, fold expressions are pretty neat and I am definitely looking for more interesting situations to use them in.

Let's break down what exactly we want to do to build our bitmask and how we might do that with a fold expression. We want to:

1. Apply some offset to every bit number
2. Create an individual bitmask by left shifting 1 by that resulting (offset + number)
3. Bitwise-OR all of the individual bitmasks together

As I delved into the fold expressions I found that step 3 is pretty straightforward. To do that with a unary left fold the syntax will be:

```c++
template<typename... Args>
constexpr uint32_t multibit_mask(Args... args) {
	return ( ... | args );
}
```

However when we throw in the bit shifting, it isn't so straightforward. For example, we can't do this:

```c++
template<typename... Args>
constexpr uint32_t multibit_mask(Args... args) {
	return ( ... | 1<<(args+OFFSET) );
}
```

That won't even compile. Yet if we keep our separate single-bit masking function, everything works out fine:

```c++
constexpr uint32_t singlebit_mask(int num) {
	return 1 << (num + OFFSET);
}

template<typename... Args>
constexpr uint32_t multibit_mask(Args... args) {
	return ( ... | singlebit_mask(args) );
}
```

I really wanted to find a way to reduce everything to a single function call, since the `multibit_mask` function works fine when passed a single argument. At this point `singlebit_mask` is just an implementation detail. One way to acheive this is by turning `singlebit_mask` into a lambda function. Starting with C++14, lambda functions are allowed to be constexpr. It ends up with this (ugly) syntax:

```c++
template<typename... Args>
constexpr uint32_t multibit_mask(Args... args) {
	return ( ... | [](auto x){ return 1 << (x + OFFSET); }(args) );
}
```

Even if we try to separate things onto different lines, I think this syntax makes the entire function painful to parse and way more difficult to read than it needs to be. But it works! Once again although things aren't _too_ horrible in this simple example, I think having the inline lambda vs three lines of implementation detail are a no-brainer from a readability standpoint.[[3]](#footnotes)

There are cases where lambdas are really wonderful tools and can even help make things more readable, but I think that this is definitely not one of them. Even if we try to separate the structure a little, I am still not a fan. I think it looks messy when you immediately call lambdas with parameters right after the lambda definition. Anyway, here's what we end up with:

```c++
template<typename... Args>
constexpr uint32_t multibit_mask(Args... args) {
	return ( ... | [](auto x) {
		return 1 << (x + OFFSET);
	}(args) );
}
```

I do however think that a constexpr lambda could be useful in turning that runtime loop used to setup our `allbits_mask` convenience variable into a compile-time constant. Check out the final program below for that one.

_Final note_: If we made some nifty use of `auto` for our variables and function return type, `decltype` for deduced return types based on the function arguments, some `static_assert`s, and [`std::is_integral`](http://en.cppreference.com/w/cpp/types/is_integral) we probably could make the `multibit_mask` function applicable to all integral types, not just `uint32_t`. However, I think that's enough for one day.

Thanks for reading.



#### Final C++14 program/snippet {#final-program}

```c++
// C++14 version of that simple C snippet
constexpr uint32_t BASE_ADDRESS = 0x123;
constexpr uint32_t GPIO_REG     = 0x456;

// my LEDs labeled 0,1,2,etc start at bit 3 of the register
constexpr uint32_t OFFSET = 3;
constexpr uint32_t bitmask(int num) {
	return 1 << (num + OFFSET);
}

// build a bit mask
template<typename... Args>
constexpr uint32_t bitmask(Args... args) {
	uint32_t mask = 0;
	for(int x : {args...}) {
		mask |= 1 << (x + OFFSET);
	}
	return mask;
}

// some data for 'doing stuff'
// It is set somewhere else in our program
constexpr int MAXBITS = 8;
int some_data[MAXBITS];

volatile uint32_t* mem_region;

void toggle_on(uint32_t reg_addr, uint32_t mask) {
	mem_region[reg_addr] |= mask;
}

void toggle_off(uint32_t reg_addr, uint32_t mask) {
	mem_region[reg_addr] &= ~mask;
}

// some masks
constexpr uint32_t COMBO_MASK = bitmask(1, 3, 5);
uint32_t ALLBITS_MASK = []{
	uint32_t mask = 0;
	for(int x = 0; x < MAXBITS; x++) {
		mask |= bitmask(x);
	}
	return mask;
}

void do_stuff() {
	// open /dev/mem and mmap desired region to mem_region here

	// set up our pins
	toggle_off(GPIO_REG, ALLBITS_MASK);
	toggle_on(GPIO_REG, COMBO_MASK);

	// do some stuff with our data and toggle pins
	for(int x = 0; x < MAXBITS; x++) {
		if(some_data[x] < 50) {
			toggle_off(GPIO_REG, bitmask(x));
		}
	}
}
```


#### Footnotes ####
1. C++17 has been [feature complete](https://herbsutter.com/2016/06/30/trip-report-summer-iso-c-standards-meeting-oulu/) regarding core language changes for a while now, but Standard Library changes were still being worked on until recently. C++17 as an ISO standard should be finalize [in the next few months.](https://web.archive.org/web/20170514184511/https://isocpp.org/std/status) 
1. In order to even compile the C++17 stuff I needed to download gcc-7 and build it from source. The [GCC 7.1 package](https://launchpad.net/~ubuntu-toolchain-r/+archive/ubuntu/test) isn't available for Ubuntu distributions prior to 17.04.
1. Much like the [spaces vs tabs](https://www.youtube.com/watch?v=SsoOG6ZeyUI) or [indent style](https://en.wikipedia.org/wiki/Indent_style) debates, I am sure many people would disagree.
