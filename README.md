# Lessons learned from C++ 20 modules

## Introduction

I have dabbled with C++ modules the last couple of weeks in MSVC. It brings a lot of cool new stuff to the table, but you
can still sense that is not quite finished yet. I have tried to use it in a real project, and I have learned a lot from the
experience. These findings are based on my own experiences which I decided to write down for myself, but could also benefit
others. But because the following is based on my own experiences and observations, it might not be 100% accurate. I will not
not go into the details of how modules are 'formally' supposed to work.

As mentioned before, I have used MSVC to try out modules. I have not tried it with other compilers, so I cannot say if the 
findings in this document are applicable to all compilers or if they are specific to MSVC. I'm using version 19.36.32535 
of the MSVC compiler at the time of writing. As modules are still heavily in development, it is very likely that the findings
in this document will be outdated in the future.

I'm writing out these learnings in no particular order, so they might seem a bit random at times.

## 1. Modules are not header files

The first thing to understand is that modules behave differently than header files. Although this might seem obvious,
you need to mindful of it while you are writing your code. It is a mindset thing. The most important things to remember are:

- They are treated differently from a pre-processor perspective. With header based libraries, you could use include guards
to prevent multiple inclusion. They are no longer a possible mechanism to prevent cyclic dependencies between modules. When 
creating modules you need to carefully think ahead about the design of your module. (More on that later)
- You can't base your library on macro's anymore. Although macros are mostly considered 'evil' these days, you still see
them often used in libraries. Modules cannot export these. You can still use them inside your module, and they have their
uses, but if your library is based on them, you need to rethink your design.
- You cannot simply mix header files and modules. You can still use header files in your modules, but you need
to be careful. Due to the different ways these are treated, you can get unexpected compiler errors which might not seem 
very obvious. But in my experience these are often related to the pre-processor and compiler not playing nice together
resulting in duplicate symbol definitions, etc.

## 2. Modules are not libraries but they kinda are

Modules are not libraries in the traditional sense. Modules on itself do not produce a (static or dynamic) library file.
They can be shipped alongside a library, but they can also be a self-contained unit used within library or executable projects.
At the same time, they share some of the nice things that you have with libraries in the traditional sense: They are compiled units of code. So this means that if you split up your code base in modules it reduces the chance
greatly that a change in one module will result in a recompilation of all other modules, unless the change directly 
affects the interface of the module on which other parts of the library/application depend.

## 3. MSVC

With the latest version of MSVC, modules are enabled by default. You don't have to use the /experimental:module flag anymore
if you want to experiment with modules. This flag is only needed if you want to use the experimental version of the standard
library modules. Please note that I initially expected that the standard library modules are in fact part of the standard,
but it seems that they are not. I'm not sure at this point if they will eventually become part of the standard, or if this
is just a Microsoft-specific thing. For now, I would avoid them if you want to be 100% sure that your code will be portable.
(and ps: you can obviously still use the STL in your modules through regular headers.)

It's also important to know that to use modules, the compiler must be set to conformance mode through the /permissive- flag.
This is also the default setting in the latest version of MSVC. So not something you have to worry about, but it is good
to be aware of. This also means that if you have an (older) code base that is not yet fully conforming, you will first need to
fix that before enabling modules. I tried to simply turn modules on in an existing code base, but that resulted in a lot of
compiler errors. So be prepared to do some work before you can start using modules.

## 4. Design

As mentioned in the previous section, when I initially turned on modules in an existing code base, I got a lot of compiler
errors and the work required to fix them was simply too much. So I decided to use modules in a new project and take it
step-by-step.

When thinking about a module there are different ways to design/structure it. Some simply expose an entire library through
a single module. Others split up their library in multiple modules. For my new project I used the latter, because of the 
following reasons:

- I wanted to layer my architecture in such a way that I would have core building blocks in the form of modules
that can easily be used by other modules. This way I effectively create a library of building blocks which can relatively
easy be mixed and matched together. 
- It was also a 'natural' way of segregating responsibilities in my code and making sure the building blocks were truly loosely coupled.
- It helps in 'only using what you need': instead of pulling in an entire library, you can simply pull in the modules you need.
- On a more practical note; it also helped me in clearly creating a module hierarchy that prevents cyclic dependencies. The layered
approach basically means that higher level modules can only depend on lower level modules, but not the other way around. For example:
I've defined some basic types in the lower level modules which are likely to be used everywhere else, which than get consumed by the
higher level modules. You can kinda compare it to an inverted pyramid where the lower level modules are at the bottom and the higher level
are at the top. This is not a hard rule, but it helps in keeping things clean and simple.

Other alternatives are to use module partitions. It's a way to segregate parts of a module into logical blocks while still
exporting through a single module interface file. I have not used this approach myself yet, but it seems like a useful feature.
The other reason why I haven't used it yet, is because somehow I always encountered errors when using it. I'm not sure if this
is a bug in the compiler or if I'm simply doing something wrong. I will need to investigate this further.

## 5. Actually creating a module interface file (.ixx)

Although creating a module interface file is relatively straightforward, I've encountered some things which I needed to follow
to get the modules to compile successfully. I'm not sure if these "rules" are based on how it is supposed to formally work of
if it is the result of modules still being new. Let's first take a look at a simple module interface which exports a class which is 
contained in a separate header file. Somehow every tutorial I've seen so far always skips on this very obvious use case (they
usually leave it at what I will show in my second example). Let's just assume I have a header file which contains the 'MyClass' class
which is located in the 'MyNamespace' namespace.

```cpp

module;

#include "myclass.h"

export module MyModule;

export {
    namespace MyNamespace {
        using MyNamespace::MyClass;
    }
}

```

Some comments:
- When including a 'traditional' header file, you need to #include it before the module declaration. If you don't do this, you
will get a compiler error. From what I could figure it had something to do with how the pre-processor works.
- When you are exposing a class from a included header file, you need to use a using statement. The export statement will not work
- You might also have noticed that I have to include the namespace of the class in the using statement, even though I'm inside a namespace block.
I still have to experiment a bit with this, but I believe you can use a class from a different namespace (and thus need to specify it), inside
a module export in a different namespace. In any case: it's required to mention the namespace in your using statement.
- The above is a way to structure your module interface file. You can also swap out the hierarchy and put the export block in the namespace block.
- The export block is optional, it saves you from having to use the export keyword in front of every declaration. I personally like it.
- You can have multiple export blocks in a single module interface file and you can switch 'styles' halfway through. Although I prefer
to keep it consistent throughout the file, it can help to deal with special cases etc.

I also found out that the following does not work well (or I haven't figured out yet how it should be done):

Let's assume the following is part of a header file:

```cpp

namespace MyNamespace {

   enum class MyEnum {
      Value1,
      Value2
   };
   
   typedef std::uint64_t SomeHandleType;

}

```

And the following is part of a module interface file (warning: this will not actually work and I will comment later):

```cpp

module;

#include "MyHeader.h"

export module MyModule;

export {
    namespace MyNamespace {
        using MyNamespace::MyEnum;
        using MyNamespace::SomeHandleType;
    }
}

```

Visual Studio wouldn't complain at all, but when I tried to compile I got multiple errors about how the compiler couldn't
resolve the enum class and typedef statement. The only way I found to fix it, was by including it in the interface file itself:

```cpp

module;

#include "MyHeader.h"

export module MyModule;

export {
    namespace MyNamespace {
        enum class MyEnum {
            Value1,
            Value2
        };
        
        typedef std::uint64_t SomeHandleType;
    }
}

```

But as mentioned before you can mix and match styles so the following also works:

```cpp

module;

#include "MyHeader.h"
#include "MyClass.h"

export module MyModule;

export {
    namespace MyNamespace {
        using MyNamespace::MyClass;
        
        enum class MyEnum {
            Value1,
            Value2
        };
        
        typedef std::uint64_t SomeHandleType;
    }
}

```

But what if your class that you want to include in your module is located in the global namespace (for some reason)? You can omit the namespace in your using 
statement in that case, but you do need to put a '::' prefix in front of it:

```cpp

module;

#include "myclass.h"

export module MyModule;

export {
    namespace MyNamespace {
        using ::MyClass;
    }
}

```

## 6. Some tricks for dealing with conditional compilation

In my code base I had some debugging code that I wanted to expose through a module. When using a tradition approach I could
simply define a macro which would call a function when building in DEBUG mode, but would resolve into nothing when building
in RELEASE mode. However, as mentioned before, modules don't support macros. So how do you deal with this? Well, used an empty
consteval function to act as an empty placeholder for the function call in RELEASE mode. As these are evaluated at compile time,
it results in the function call being removed from the code.

It is good to know that you can still use macros in the module interface files themselves to deal with conditional compilation. So the
following worked very well for me (not sure if it is the best way to do it, but it works):

```cpp

``cpp

module;

#include "myclass.h"

export module MyModule;

export {
    namespace MyNamespace {
        #ifdef _DEBUG
            export void DebugFunction() {
                // Do something
            }
        #else
            export consteval void DebugFunction() { } //Do nothing
        #endif
    }
}

```

## 6. Using modules

Using modules is relatively straightforward. You simply import them and use them. The only thing you would want to keep in mind is to include
any traditional header files before the module import statement. This is because the pre-processor is still used to process the traditional header
files and the compiler needs to know about them before it can process the module import statement. So the following would work


```cpp

//I usually tend to include STL headers first for consistency and it seems to matter sometimes
#include <vector>

//then I include my own headers or other third party headers
#include "myclass.h"

import MyModule;

namespace MyNamespace {
    class OtherClass {
        // Use your stuff here
    };
}

```

## 6.5 Using modules - the quirks

Ok, so all great stuff. But I've encountered some quirks which I'm not sure if they are bugs or if I'm doing something wrong. But you
are probably likely to encounter them as well, so I'll mention them here.

- The order in which you are importing/including things matter. If you are not careful, you get weird errors about not being able to resolve
symbols. Or you run into a situation where the compiler stars complaining about re-definition of symbols from the STL. It can be
maddening at times to figure out.
- As an extension to my previous point; it sometimes appears as if your regular include guards don't work anymore. I'm not sure
if that is actually the case or if it is a side effect of something else. My compiler will freak out about re-definition of symbols
from time to time, which I'm able to fix by re-ordering my includes. It makes me feel uncomfortable though, as I'm not sure what
is actually the cause of it - and I don't like that.

for example, this won't work:

```cpp

#include "class1.h"
#include "class2.h"
#include "class3.h"
#include "class4.h" //ARGH! Panic about re-defining stuff, although there are include guards everywhere!

import MyModule;

namespace MyNamespace {
    // Stuff
}

```

But this does work:

```cpp

#include "class4.h" //This is fine now - BUT WHY?!
#include "class1.h"
#include "class2.h"
#include "class3.h"

import MyModule;

namespace MyNamespace {
    // Stuff
}

```

I also ran into issues when including files beyond the module import statements. (This could be due to how the pre-processor processes these things) For example:

```cpp

#include "class1.h"
#include "class2.h"
#include "class3.h"

import MyModule;

#include "class4.h" //ARGH! Panic about re-defining stuff, although there are include guards everywhere!

#include <vector> //Even more issues!

namespace MyNamespace {
// Stuff
}

```

