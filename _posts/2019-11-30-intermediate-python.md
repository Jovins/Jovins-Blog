---
layout: post
title: Intermediate Python
date: 2019-11-30 17:19:36
tags: [python, django]
categories: [programming-language, python]
---

![python-logo.png](https://i.postimg.cc/vBzKnzhB/python-logo.png)

> As all known, there are two versions of Python, Python 2 and Python 3. One suggestion: learn python 3. 

> From [Python 2 sunset announcement](https://www.python.org/doc/sunset-python-2/), it states that 2020/01/01 will be the day to sunset Python 2. During that time, Python 2 will stop maintenance. Time to transfer to Python 3 lah. 

> Another suggestion is that pls keep critical thinking. This blog was initially intended to be a self reflection. Hence, bear with me if there are some mistakes through this blog. 


Python is a versatile language that can support for multiple purposes. 
1. scripting 
2. web development 
3. ML / AI track 

There are I have seen too many tutorials about Python basics, most of them introduces ML track. However, I will introduce Python in web development track since I am quite proficient in ML/AI track. For those tracks, there are much more tutorials that do better work than me. 

```Python
# This blog will include a lot of code blocks

import sys

def main():
    print("Hello, ", sys.argv[1])

if __name__ == '__main__':
    main()
```
This is a simple Python code script that print a sentence with a parameter that you input. 
The reason that you set `__name__ == '__main__'` is because you wanna different behaviors between this Python file being executed directly and being imported by other modules. 

For introduction Python tutorial, please refer to reference [[1]](https://developers.google.com/edu/python/introduction). This blog will assume readers to have such prerequisites. 
1. Python basic 
2. CS fundamentals

## general knowledge:
Python does minimum compile-time check, almost all the check is done in runtime check.  

python string is immutable, just like string in Java and C#. The reason that string type in Python is immutable is: 
1. it is safe to put as a dict key without deepcopy the content of string. If assign string to another variable then they do not point to the same object. 
2. it is threadsafe
3. easy for programming, you can add mutable wrapper on immutable type but the reverse is difficult 

Every time we manipulate string, we actually construct brand new string with changes that you made. 

P.S. There is one hidden logic in Python interactive interpreter worth noticing, the single underscore (_) is bounded to the last expression evaluated. For instance: 

```Python 
>>> lambda x, y: x * x + y * y
<function <lambda> at 0x102922e18>
>>> _(2,3)
13
```
Here the single underscore refers to the lambda function object that I initialised before. There are actually a lot of usages of underscore in python code. Some of them are just conventions and designed as a hint for programmers, while others are enforced by python interpreter. 

Some of the most common usages are as following: 
1. single underscore: `_`
    - used as a variable name, by convention the value of this variable is not cared
    - referred as last object in the interactive python shell, like last function or last variable 
2. single leading underscore: `_var`
    - by convention indicating this variable `_var` is intended for internal use 
    - When importing using wildcard (*), functions with leading single underscore will not be imported 
3. single trailing underscore: `var_`
    - used by convention to avoid conflicts with python keywords 
3. double leading underscores: `__var`
    - enforced by python interpreter, trigger name mingling when using in a class. 
        + for example, varaible name will append class name in front `__var` ==> `_Class__var` while class name does not leading underscores 
4. double leading and trailing underscores: `__var__`
    - reserved for python special methods like `__init__` and `__call__` 

For the full reference of underscore (_), please refer to [this blog](https://dbader.org/blog/meaning-of-underscores-in-python).  

One famous feature of Python is its concise syntax. There are some syntactic sugars in python. According to [Wikipedia](https://en.wikipedia.org/wiki/Syntactic_sugar), `Syntactic sugar` is programming syntax that is designed to make code easy to read and understand. One sugar is called `decorator`. 

```Python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        # do something before actual function
        result = func(*args, **kwargs)
        # do something after actual function 
        return result
    return wrapper

@my_decorator
def actual_func():
    # do something 

# similar with 
my_decorator(actual_func)
```

decorator is just normal function in python and it can be used as `@` + `function name` to simplify the usage. If the function to be wrapped has some arguments, it can be passed in through `*args` and `**args`


### python encoding & decoding 
The first question, of course, `What is encoding/decoding?`

According to statement of [this post](https://searchnetworking.techtarget.com/definition/encoding-and-decoding), 

> In computers, encoding is the process of putting a sequence of characters (letters, numbers, punctuation, and certain symbols) into a specialized format for efficient transmission or storage. Decoding is the opposite process -- the conversion of an encoded format back into the original sequence of characters. 

Simply speaking, encoding and decoding is special converting method that converting between bytes and string and bytes to string ==> decode, string to byte ==> encode. After encoding, data format changes to a format (byte) that machine can understand efficiently. After decoding, data fomrat changes to format (string) that human kinds can understand efficiently. 

To construct an unicode string in python 3, add a prefix `u` in front of the string, like `u'\xf1'`. 

If encode, then string becomes bytes with specific encoding method like `utf-8`. `UTF` stands for `Unicode Transformation Format`. Since `utf-8` is more universal and can represent more characters in computer world, most of the applications will use `utf-8` as default encoding method (especially those application that has Chinese characters)

For example, this is a small script that I ran in `python interactive interpreter` to illustrate the basic encoding and decoding process in Python. 
```Python
>>> unicode_str = u'\xf1'
>>> unicode_str
'ñ'
>>> unicode_str.encode(encoding='UTF-8',errors='strict')
b'\xc3\xb1'
>>> utf8_str = unicode_str.encode(encoding='UTF-8',errors='strict')
>>> utf8_str.decode()
'ñ'
```

Then there is a question, will `utf-16` use more bytes to encode a char compared to `utf-8`? what about `utf-32`?

The answer is `absolutely no`. For different characters, `utf8` and `utf16` and `utf-32` will use different number of bits to encode. For instance, *U+0000 to U+007F* are encoded as 8 bits in `utf-8`, 16 bits in `utf-16` and 32 bits in `utf-32` while *U+0800 to U+FFFF* requires 24 bits for `utf-8` but only 16 bits for `utf-16` and 32 bits for `utf-32`. 

From the above comparison, `utf-8` is more efficient than `utf-16` in smaller unicode indexes, which are used more frequently. Hence, most applications use `utf-8`. 

Full reference can refer here: [Unicode HOWTO](https://docs.python.org/3/howto/unicode.html) and [Unicode Compare](https://en.wikipedia.org/wiki/Comparison_of_Unicode_encodings)

### python list, tuple, dict, set 

#### list
list is passed by reference, use `=` does not copy the content of list, only copy the pointer to the object. `+` works as concatenation between lists. 

##### pythonic
list comprehension is the most common `pythonic` implementation of `for` loop. According to [this post](https://stackoverflow.com/a/25011492/5891473) and [Zen of Python](https://legacy.python.org/dev/peps/pep-0020/), 

> Exploiting the features of the Python language to produce code that is clear, concise and maintainable. 

You may understand more about `pythonic` now. From my own point of view, `pythonic` is a kind of coding style that the creator of `python` want programmer to write in. It will make python code concise and clear. 

##### Sorting 
`sorted` is the built-in function to sort a list of objects according to a key. By default, the list is sorted by integer value. The key to sort the list can be customized. 

```Python
list = [2, 3, 4, 1, 6]

sorted_list = sorted(list) # default sorting, [1, 2, 3, 4, 6]

descending_sorted_list = sorted(list, reverse=True) # [6, 4, 3, 2, 1]

def some_func(element):
    # do some operation of this element
    return element * 0.1 if element % 2 == 0 else element * 2

customized_sorted_list = sorted(list, key=some_func) # [2, 4, 6, 1, 3]

# The some_func can also be implemented by lambda function 
lambda_sorted_list = sorted(list, key=lambda element: element * 0.1 if element % 2 == 0 else element * 2) # [2, 4, 6, 1, 3]
```

##### lambda
In the last line of previous code snippet, there is a function called `lambda`. For example, 

```Python
lambda x: x * x + 1 

# same as the normal function
def some_func(x):
    return x * x + 1
```

lambda function is an anonymous function which do not need to state the name of the function. A lambda function contains three components, the keyword (`lambda`), the function input variable (`x`) and the statement (`x * x + 1`). Python lambda function only allows one statement and the result of the statement will be returned as the result of the lambda function. You don't need to specify the `return` keyword because hidden logic of lambda statement is to return the value. 

According to [Real Python post](https://realpython.com/python-lambda/#anonymous-functions),

> Lambda functions are frequently used with higher-order functions, which take one or more functions as arguments or return one or more functions. 

More specifically speaking, when a function needs to take one function as its argument, we don't need to write the function body separately, we can use lambda function to take place. The `higher-order functions` are mentioned in the next section. 

##### Advanced usage - map/filter/reduce 
There are some advanced usages of `list` in python. These are `higher-order functions` mentioned in the previous section. These higher order functions are used to conveniently manipulate elements in the list and output some response.

###### map
For map, it is used to execute a function to every element in the list (Actually may not be a list, just an iterable). For example, we would like to achieve the effect of make every element to attach a string `happy` in the end, we can do it using `map` function. `map` is so-called `higher-order function` since it takes functions as its arguments. The first argument of `map` is the function (we call it `exe`) that we would like to execute and the following arguments are iterables of arguments of `exe`. Each time of `map(exe, list_of_a, list_of_b)`, `exe(a, b)` will execute be executed and return the result. 

```Python
>>> input = [1,2,3,4,5,6]
>>> input
[1, 2, 3, 4, 5, 6]
>>> res = list(map(lambda x: str(x) + 'happy', input))
>>> res
['1happy', '2happy', '3happy', '4happy', '5happy', '6happy']
>>> input
[1, 2, 3, 4, 5, 6]
>>> res = list(map(lambda x, y: str(x) + str(y), input, range(1, 7)))
>>> res
['11', '22', '33', '44', '55', '66']
```

Note that this function is not in place, which means that the function will not change the value of original list. Rather, it will assign value to another list. 

##### filter
`filter` is a function used quite frequently in list manipulation. Nomally it will setup some filtering function and return a sublist of the list according to the filter. For example, you wanna find all the elements meeting the lower bound, you would do something like this: 
```Python
>>> res = list(filter(lambda x: x > 3, input))
>>> res
[4, 5, 6]
```
It will filter out elements less than 3. The `filter` function has the same mechanism with `map`. One thing special for `filter` is that it requires the return value of its first argument function to be boolean to judge whether to put it into returned sublist. 

##### reduce 
`reduce` is to apply function to adjacent elements cumulatively in the list and return the final result. The returned value of `reduce` is a single value. Notice that `reduce` is not import by default, you need to explicitly import it from `functools`
```Python
from functools import reduce

input = [1,2,3,4,5,6]
initial_val = 10
res = reduce(lambda x, y: x^y, input, initial_val) # res = 13
```
This reduce function computes the XOR result of all elements of `input` plus the initial value, so the reduce function can be explained as `10 ^ 1 ^ 2 ^ 3 ^ 4 ^ 5 ^ 6`, which is 13. 

#### tuple

> A tuple is a fixed size grouping of elements, such as an (x, y) co-ordinate. 

According to Google python tutorial, `tuple` is immutable. `Immutable` means once the variable is assigned some value, it cannot be modified anymore. 
The tuple can be created by using `tuple((tuple_itself))` or just simply `()`

```Python
tuple_sample = (1, 'string', 2.3)
tuple_sample = tuple((1, 'string', 2.3)) # effect same as above
tuple_sample[1] = 'another_str'
```
Once someone tries to modify tuple value like above, exception will pop up. 

```
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: 'tuple' object does not support item assignment
```

According to [tutorial of Python tuple](https://appdividend.com/2019/01/05/python-tuple-example-tutorial-complete-introduction-on-tuples/), there are some operations allowed to operate on Python tuple: 

| Allowed operation | Python expression              |
|-------------------|--------------------------------|
| find the length   | len(tuple_name)                |
| concatenate       | tuple + tuple                  |
| repetition        | tuple * times                  |
| check if inside   | if ele in tuple                |
| Iteration         | for ele in tuple:              |
| nesting           | tuple_total = (tuple1, tuple2) |

Another property that can be derived from the immutability of `tuple` is that it is hashable and as a result it can be used as key of `dict`. The reason is that if it is used as key of `dict`, the value of the key itself is passed by reference, which means if the `dict` key is confirmed, it is not supposed to be changed anymore. If it is a `list`, it will have the risk to change its value, the hashing algorithm of `dict` will gonna change its source value. Hence, `list` is not hashable. 

#### dict 
`dict` is a kind of data structure that contains key-value pairs. Full name is `Dictionary` in python. It has similar mechaism with `HashMap` in java. This kind of framework is often used when efficient searching is required and ordering is not strictly required. It does not need to loop through the whole data to find the data with specific key, since it can find the data according to its hash key. 

`dict` in python is thread safe, which is a quite important feature when it supports multi-thread application in python. 

`OrderedDict()` preserves the order of entry insertion. When loop through the dict, the order is the same as the insertion order. 

```Python
>>> my_dict = {}
>>> my_dict[(1,2)] = [1,2]
>>> my_dict
{(1, 2): [1, 2]}
>>> my_dict[[1,2]] = [1,2]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
```

For simple example above, `my_dict = {}` initialises a dictionary with empty key-value pair. When you wanna add key-value pairs into the dictionary, simply use `my_dict[key] = value`. The key's data type has to be hashable since the dictionary need to compute the hash code of input key to efficient store it. As introduced in last section, `tuple` can be used as dict key and `list` cannot be used as dict key since `tuple` is hashable and `list` is not hashable. This also reflects when serialization and deserialization of JSON structure is used. One direct method to check whether it is hashable is to check whether it has the `__hash__` method. This method is used to give a hash value to an object. 

However, we cannot simply say `tuple` is hashable because it is quite flexible and its elements may be unhashable data types. 

##### hash function
According to [wikipedia hash function](https://en.wikipedia.org/wiki/Hash_function), 
> hash function is a kind of algorithm that can be used to map data of arbitrary size to fixed-size values. The values are used to index a fixed-size table called hash table. 

To interpret this, we have to know that hash table is used to efficiently retrieve data from a chunk of data. The reason is that it makes unordered list become in-order in the underneath implementation. The hash function will convert different data types into valid array index and use constant time to find the corresponding values. 

The complete list of data types **not** available to be dictionary key: 
* list 
* dict
* set 
* tuple with above elements 

Basically if a data type is not list / dict / set or tuple with those elements, it is hashable. 

`consistent hashing` is another topic, I will mention it in blog of distributed system if I have that blog. 

#### set
set in python is implemented in hash also. 


### multithread 
`Multithread` is to execute a program using different threads, access one common resource from different places and push or pull its content into the resource. 

A thread contains:
* thread ID 
* program counter 
* register set 
* stack 

The program counter of thread is used to record the next instruction to execute in the program. Register set is used to record value of variable in memory. Register set has the fastest access speed among the methods of saving variables (like cache and hard drive). Stack is used to store the local primitive variables and reference to objects in heap. Every thread has its own stack. 

Multiple threads shares code section, data section and any operating resource that need to be accessed by different threads. 

#### GIL 
There is a mechanism in python called `GIL(Global Interpreter Lock)` if we assume that underline implementation of Python is `CPython` (Cpython is the default interpreter of Python, it will translate Python code to byte code). 

According to [Python Wiki](https://wiki.python.org/moin/GlobalInterpreterLock), 
> In CPython, the global interpreter lock, or GIL, is a mutex that protects access to Python objects, preventing multiple threads from executing Python bytecodes at once. 

This statement states that only one active thread is allowed during the whole context of Python program. Even if the CPU running python program has multiple cores, it will be blocked by the `mutex` if the processes are not fully separated.   

Because of this constraint, the native multithreading in python is not actually parallel. 

Then the question comes, why we still need `Threading` module with the constraint of `GIL`? It will be explained in next section. 

#### Concurrency and Parallelism 
Before introducing the tricks, I would like to introduce some fundamental concepts related to `multithread` first. Although some of the concepts may not quite related to Python, it is still good to know. 

When we talk about multi-threading and multi processing, what we really want to achieve is actually concurrency and parallelism. 

According to [this blog](https://learn-gevent-socketio.readthedocs.io/en/latest/general_concepts.html), 
* concurrency: managing access to shared state from different threads, like scenario of most web applications 
* parallelism: utilising multiple processors / cores to improve the performance 

So let's say concurrency is used to manage different clinets accessing one server. Since server has only one program running, this program need to accept accesses from different clients, they use multithreading to achieve this. Parallelism is used to improve performance by fully utilizing the processors to improve the overall performance. 

After knowing those basic concepts and the GIL in python, you may raise a question: Why we still need multi-threading in python since it cannot improve the performance? 

For different scenarios, there are different kinds of bottlenecks. Mainly there are two kinds of bottlenecks, IO/network bottleneck or computational power bottleneck. Python multithreading can solve IO bottleneck (like webserver) by allowing a lot of traffic accessing same server. For CPU-bound program, it will not help since only one active thread is working at the same time. 

However, python can still achieve parallelism by using multi-processing. 

#### gevent vs greenlet vs threading 
According to [the next section of the blog](https://learn-gevent-socketio.readthedocs.io/en/latest/gevent.html), it states the relationship between gevent and greenlet 
> Greenlets are a lightweight cooperative threads that are managed and scheduled inside the process. 

`Greenlets` is different from conventional thread in python in these aspects:

```
for POSIX thread (pthread):
* pthreads can switch between threads pre-emptively, switching control from a running thread to a non-running thread at any time	
* On multicore machines, pthreads can run more than one thread. However python’s Global Interpreter Lock (CPython Intepreter) prevents parallelism and concurrency is only effective for I/O-bound programs	
* Race conditions can occur when implementing multi-threading code. Use locks to manage mutex to avoid race conditions.	

for greenlets: 
* greenlets only switch when control is explicitly given up by a thread - when using yield() or wait() - or when a thread performs a I/O blocking operation such as read or write
* greenlets can only run on one single CPU and is useful for I/O-bound programs
* There’s no possibility of two threads of control accessing the same shared memory at the same time for greenlets so there will not be any race conditions.
```

From the above paragraph, it is obvious that greenlets can let program take full control of multithreading, it will not be interrupted but only switch to another thread when calling yield() or wait(). Moreover, both of these two approaches (greenlets and native thread) do not bypass the GIL but only achieve concurrency. 

According to [Gevent official doc](http://www.gevent.org/intro.html), 

If you regard `greenlets` as a thread, `gevent` is used to control the greenlets to execute in certain order. 

Gevent monkey patch is to change program behaviour without changing source code. It will use gevent to replace the native python threading module. 

```Python
import gevent.monkey; gevent.monkey.patch_thread()
import threading
```

## scripting track: requests + beautifulsoup + Pandas + Selenium(?)
I am not quite familiar with web crawling track. This track most focuses on getting massive information from websites and do data analysis based on that. 

## web development track: Django 
There will be another blog specifically talking about this topic. Because of limitation of length, I will not introduce details about Django. 

## ML / AI track 
I am not quite familiar with AI track since I only did some tiny tutorials that can be found on Internet. Further tutorials can be found everywhere else since machine learning is super hot these days. But as for me, I would concentrate on software development to make it more reliable. 

## Reference: 
1. [Google Python Tutorials](https://developers.google.com/edu/python/introduction)
2. https://www.python.org/dev/peps/pep-0008/
3. https://docs.python.org/3/library/
4. https://stackoverflow.com/questions/8680080/why-are-python-strings-immutable-best-practices-for-using-them
5. https://docs.python.org/3/howto/unicode.html
6. https://realpython.com/
