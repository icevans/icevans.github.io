---
title: Ruby Pointers
layout: post
---

It is often said that in Ruby, variables are _pointers_ -- they point at, or reference, objects, but are not themselves objects. The 'Pickaxe' puts it nicely:

> Ruby variables and constants hold references to objects. Variables themselves do not have an intrinsic type. ... (When we say that a variable is not typed, we mean that any given variable can at different times hold references to objects of many different types.)

So, for example, consider the following Ruby code:

```ruby
a = "before"
a = 1
```

Ruby first creates a string object in memory with the contents `'before'`. The variable `a` is then set to hold a reference to that memory location. An invocation of `a` will now look to that memory location and pull out whatever object is there. But then, in the next line, we instruct Ruby to create an integer object at new location in memory with the value `1`, and then we _reassign_ `a` to hold a reference to this new memory location.

And this is not purely theoretical: it explains, for example, if we set `b` to `a`, and then mutate the object `a` points at, invoking `b` will reveal that mutation. So, in the sense that they merely point at memory locations, Ruby variables are pointers.

Especially if Ruby is your first programming language, I think it's easy to fall into the trap of thinking this is how a variable _has_ to work -- what else is a variable than something that can point to different objects at different times?

## Pointers in C

That's all that Rubyist needs to go about her day-to-day job of writing Ruby code. But it's to fall into the trap of thinking that this is how variables _must_ work. To me though, one of most fun parts of programming is learning different approaches to problems. What's more understanding how other languages handle common tasks can yield greater insight into how your own language handles these tasks, as well as the tradeoffs involved.

I knew that C handles variables differently than Ruby, and that pointers are their own special thing. I've been wanting a deeper understanding of Ruby variables. Since it's Sunday and I've got the day to myself, I decided to open up [K&R](http://cs.indstate.edu/~cbasavaraj/cs559/the_c_programming_language_2.pdf) and learn enough C to understand the basics of its variables and pointers.

Variables in C work more like 1 above. When you initialize a C variable, a specific location in memory is set aside (the size of the location is determined by the _type_ of the variable -- floats need more space than integers, which need more space than characters). Assigning a value to the variable writes that value to the location in memory that was created at initialization. Reassignment changes the value at this location.

C gives us a neat operator: `&`. The `&` operator returns the actual memory address of a particular variable (representable as a hexadecimal number). So we can verify the preceding paragraph with the following simple C program:

```c
#include <stdio.h>

int main(void) {
  int a; /* Initialize a new variable of type int */
  a = 1; /* Assign 1 to a

  printf("%d", a); /* => 1 */
  printf("%p", &a); /* => 0x7ffee3f7773c */

  a = 2;

  printf("%d", a); /* => 2 */
  printf("%p", &a); /* 0x7ffee3f7773c */
}
```

Ignoring the C boilerplate code, what we see is that the value of `a` has changed, but its location in memory has not. This means that if we initialize another variable `b` and assign `a` to it, we will only be writing the current _value_ of `a` to `b`'s unique memory location -- but `a` and `b` will point to distinct locations in memory and subsequently changing one would have no effect on the other. 

```c
int a;
int b;

a = 1;
b = a;


printf("%d", a); /* => 1 */
printf("%p", &a); /* 0x7ffee3f7773c */

printf("%d", b); /* => 1 */
printf("%p", &b); /*  0x7ffee3f77730 */
```

This is quite different from Ruby! In Ruby, after assigning `a` to `b`, `b` would point at the same object in memory as `a`. 

If you want that sort of thing in C, you create a _pointer_ instead of a variable. Like variables, pointers have to be initialized with a type (which limits the type of object they can point at). 

```c
int a
int *b; /* Initialize an empty int pointer */

a = 1;
b = &a;

printf("%p\n", &a); /* => 0x7ffee3f7773c */
printf("%p\n", b);  /* => 0x7ffee3f7773c */ 
printf("%p\n", &b); /* => 0x7ffee3f77730 */
```

Here we see that the value of `b` is the address of `a`. But `b` has its own address in which it holds the address of `a` as a value. Essentially, `b` is a variable that can contain addresses to `int`s. If we want to get the value at the address that `b` stores, we have to _dereference_ the pointer using the `*` operator:

```c
printf("%d\n", *b) /* => 1 */
```

Further, assignment to a dereferenced pointer changes that value at the address in question. This means any regular variable whose address that is will be affected. So, continuing with our example:

```c
*b = 2;

printf("%d\n", *b); /* => 2 */
printf("%d\n", a);  /* => 2 */
```

There isn't really a way to do that in Ruby. Neat.

## Back to Ruby

Earlier, I said it was helpful to think of a Ruby variable as containing the address of the object it is currently pointing at, but that this is obscured by the fact that the variable will always evaluate to the value at that address. We can now see that Ruby variables are very similar to dereferenced pointers in C (except during assignment, where they work pretty much like regular pointers). And they are not at all like regular variables in C.

For me, at least, this is a much clearer conceptual picture of how Ruby is interacting with the underlying machine. And with this clearer conceptual picture, we can get much clearer on what people mean when they say that Ruby is "pass by reference value."

## Pass By Reference Value

C is pass by value: if your function is defined to accept an integer as a parameter, and you call it passing it an `int` variable, a new variable, local to the function, will be created and the value at the address of the argument will be copied to this new location. However, pointers allow you to pass by reference (this example comes pretty much directly from K&R):

```c
int swap(int *x, int *y) {
  int temp;
  temp = *x;
  *x = *y;
  *y = temp;
}

int main(void) {
  int a, b;

  a = 1;
  b = 2;

  swap(&a, &b);
  printf("%d\n", a); /* => 2 */
  printf("%d\n", b); /* => 1 */
}
```

Look at that! The method was able to reassign the values of the arguments it received. What happened? Rather than passing the `swap` function the values of `a` and `b`, we passed it their addresses. Because we defined the function with pointers as its paramters, this gave us pointers to `a` and `b` to use within the function. As per the previous discussion, the value of these pointers are the addresses of `a` and `b`. So when we dereference the pointers and do reassignment, _we're changing what's stored at `a`'s and `b`'s addresses_.

We cannot do this in Ruby, and this is what leads some people to not want to call Ruby pass by reference. If a method really hads its hands on the pointer it was passed, then it should be able to change what's stored at that memory location. Of course, Ruby isn't pass by value in the standard sense either: the method does have access to the very object stored in the arguments in receives (and can mutate them if they're mutable).

The important key difference is that Ruby variables are more like C's pointers than its variables. When we pass variables to a method as arguments, we are passing pointers. But we aren't passing the pointers themselves, we're passing their value: the address at which they point. Then, within the method, a new local pointer is created to point at this same address. If, within the method, we change what this pointer points at (change what address it stores) this will of course have no effect on what the pointer outside the method points at. After all, if two C pointers point at the same address, updating one to point at a new address won't change the other.

It's almost as if Ruby methods work like this:

```c
int try_change(int *a) {
  /* Note that the value of &a here is not the same as the value of &p in 
     the main function below! */
  int b = 43;
  a = &b;
}

int main() {
  int a = 42;

  int *p;
  p = &a;

  printf("%d\n", *p); /* => 42 */
  try_change(p);
  printf("%d\n", *p); /* => 42 */
}
```

Here we pass a (non-dereferenced) pointer to our `try_change` function, which was set up to accept a pointer as a parameter. Because C is pass by value, and the value of a pointer is the memory address it points at, the pointer `a` inside the function will point at the same memory address as whatever pointer argument we pass at method invocation. However, `a` is a _new_ pointer scoped to the function: _its_ address will be different than the address of the argument we passed in. In this case, `&a` in the function is different than `&p` outside the function. That is why when we reassign `a` in the function, `p` outside the function is unchanged.

So here we didn't pass the pointer -- or reference -- itself, but the _value_ of the pointer/reference. And this is strikingly similar to how things work in Ruby. This, I think, is the sense in which people mean that Ruby is "pass by reference value". The value of a pointer is not the object at which it points, but the location of that object. In Ruby, variables are pointers in this sense (again, modulo the fact Ruby variables always evaluate to the value of the object they point at, like dereferenced pointers in C). And what gets passed to methods are the values of these pointers. Inside the method, a new locally scoped pointer is created to point at this same location. This allows us to access and/or mutate any objects referenced at invocation, but not to reassign any pointers outside the method's scope.

