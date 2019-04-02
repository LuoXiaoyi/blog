---
layout:    post
title:     Windows DLL(动态链接库)的使用
category:  Windows
tags: DLL 动态链接库
---

以前一直在 Linux 下进行 C/C++ 的程序开发，但是在 Windows 下开发时就懵逼了，最近公司有个项目有需求，需要 Windows 下开发和调用别人的动态库，所以就抽时间实战了下，大致的原理是差不多的，只是有些细节不一样。关于如何使用动态库的问题，主要有两种方法
* 应用程序中通过 include dll 库的头文件，直接调用有关的函数来实现对动态库的调用，这时应用程序是通过默认动态库加载路径来查找相关动态库的
* 应用程序通过 dlopen(linux) 或 LoadLibrary(windows) 来根据动态库的路径 load 动态库，然后通过函数指针的方式来调用相关函数

*<strong>下面就 Windows 下动态库的两种调用方式做个简单的记录</strong>*

## Windows dll 库的编写
动态库的编写与在 Linux 下稍有不同，需要使用有关的宏来定义相关的类或者方法，不然，在使用的时候会找不到对应的符号。以下是样例代码，我们定义一个 dll.h 的头文件，如下

### dll.h 头文件
```
#ifndef _DLL_H_
#define _DLL_H_

#if BUILDING_DLL
#define DLLIMPORT __declspec(dllexport)
#else
#define DLLIMPORT __declspec(dllimport)
#endif

class DLLIMPORT DllClass
{
	public:
		DllClass();
		virtual ~DllClass();
		void HelloWorld();
};
#endif
```

### fun.h 头文件
```
#ifndef _FUN_H_
#define _FUN_H_

#if BUILDING_DLL
#define DLLIMPORT __declspec(dllexport)
#else
#define DLLIMPORT __declspec(dllimport)
#endif

#include<stdio.h>

// 这个非常重要，extern "C" DLLIMPORT 为了确保在 C++ 中可以找到 print_hello 函数
extern "C" DLLIMPORT void print_hello();
#endif
```

```DLLIMPORT``` 宏的主要作用是将 ```DllClass``` 导出，这样当 dll.h 被其他应用程序引用的时候，直接调用 ```DllClass``` 就像调用本地的程序一样。

### dllmain.cpp 对应的实现如下

```
#include "dll.h"
#include <stdio.h>
#include <windows.h>

DllClass::DllClass(){
}

DllClass::~DllClass(){
}

void DllClass::HelloWorld(){
	printf("Hello World from DLL!\n"); 
}

BOOL WINAPI DllMain(HINSTANCE hinstDLL,DWORD fdwReason,LPVOID lpvReserved){
	switch(fdwReason){
		case DLL_PROCESS_ATTACH: {
			printf("DLL_PROCESS_ATTACH \n");
			break;
		}
		case DLL_PROCESS_DETACH:{
			printf("DLL_PROCESS_DETACH \n");
			break;
		}
		case DLL_THREAD_ATTACH:{
			printf("DLL_THREAD_ATTACH \n");
			break;
		}
		case DLL_THREAD_DETACH:{
			printf("DLL_THREAD_DETACH \n");
			break;
		}
	}

	return TRUE;
}
```

### fun.cpp 对应的实现如下
```
#include "fun.h"

DLLIMPORT void print_hello(){
	printf("print_hello--> Hello World from DLL!\n"); 
}
</pre>

### 编译的 make file
Makefile.win 文件是由 DEV C++ 自动生成，如下

<pre>
# Project: test
# Makefile created by Dev-C++ 5.11

CPP      = g++.exe
CC       = gcc.exe
WINDRES  = windres.exe
OBJ      = com_perfma_xshark_plugins_jmeter_NativeClient.o dllmain.o fun.o
LINKOBJ  = com_perfma_xshark_plugins_jmeter_NativeClient.o dllmain.o fun.o
LIBS     = -L"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib" -L"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/lib" -static-libgcc
INCS     = -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include" -I"C:/Program Files/Java/jdk1.7.0_80/include/win32" -I"C:/Program Files/Java/jdk1.7.0_80/include"
CXXINCS  = -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include/c++" -I"C:/Program Files/Java/jdk1.7.0_80/include/win32" -I"C:/Program Files/Java/jdk1.7.0_80/include"
BIN      = test.dll
CXXFLAGS = $(CXXINCS) -DBUILDING_DLL=1
CFLAGS   = $(INCS) -DBUILDING_DLL=1
RM       = rm.exe -f
DEF      = libtest.def
STATIC   = libtest.a

.PHONY: all all-before all-after clean clean-custom

all: all-before $(BIN) all-after

clean: clean-custom
	${RM} $(OBJ) $(BIN) $(DEF) $(STATIC)

$(BIN): $(LINKOBJ)
	$(CPP) -shared $(LINKOBJ) -o $(BIN) $(LIBS) -Wl,--output-def,$(DEF),--out-implib,$(STATIC),--add-stdcall-alias

com_perfma_xshark_plugins_jmeter_NativeClient.o: com_perfma_xshark_plugins_jmeter_NativeClient.cpp
	$(CPP) -c com_perfma_xshark_plugins_jmeter_NativeClient.cpp -o com_perfma_xshark_plugins_jmeter_NativeClient.o $(CXXFLAGS)

dllmain.o: dllmain.cpp
	$(CPP) -c dllmain.cpp -o dllmain.o $(CXXFLAGS)

fun.o: fun.cpp
	$(CPP) -c fun.cpp -o fun.o $(CXXFLAGS)
```

使用 Makefile 进行编译，会在当前的源码目录下生成一个 test.dll 的动态库文件，接下来我们要做的事情，就是调用该动态库了。是不是很简单，哈哈。。。

## 通过动态库提供的头文件调用动态库
使用头文件来调用动态库 test.dll 文件，这个比较简单，唯一需要注意的包括
* 在编译调用程序的时候需要把 dll.h 所在目录通过 -I 的参数添加到调用程序的 Makefile 文件中
* 在编译调用程序的时候，需要把 test.dll 的动态库通过 -L 的参数添加到调用程序的 Makefile 中，让目标程序正确完成链接
* 在调用程序运行的时候，要保证调用程序能够在 ldpath 的路径下能够找到，当然，默认情况下，把 test.dll 放到调用程序的可执行程序同路径下是没有问题，否则就需要进行相关的设置了，具体设置，可以自行百度或者 google Linux 或者 Windows 下的动态库路径设置；

### 调用程序的源码 main.c
```
#include "dll.h"
#include "fun.h" 

using namespace std;
int main(int argc, char** argv) {
    
	DllClass dll;
	dll.HelloWorld();

	return 0;
}
```
<br>
哈哈，代码是不是很简单呀，没错，就是这么简单，include 头文件 dll.h，然后调用 DllClass 类来生成一个对象，最后调用 HelloWorld() 方法，非常简单方便了。

### main.c 的 Makefile 文件
```
# Project: dll_caller_test
# Makefile created by Dev-C++ 5.11

CPP      = g++.exe -D__DEBUG__
CC       = gcc.exe -D__DEBUG__
WINDRES  = windres.exe
OBJ      = main.o
LINKOBJ  = main.o
LIBS     = -L"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib" -L"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/lib" -static-libgcc ../test/test.dll -g3
INCS     = -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include" -I"C:/Users/xiluo/test"
CXXINCS  = -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include/c++" -I"C:/Users/xiluo/test"
BIN      = dll_caller_test.exe
CXXFLAGS = $(CXXINCS) -g3
CFLAGS   = $(INCS) -g3
RM       = rm.exe -f

.PHONY: all all-before all-after clean clean-custom

all: all-before $(BIN) all-after

clean: clean-custom
	${RM} $(OBJ) $(BIN)

$(BIN): $(OBJ)
	$(CPP) $(LINKOBJ) -o $(BIN) $(LIBS)

main.o: main.cpp
	$(CPP) -c main.cpp -o main.o $(CXXFLAGS)
```

```../test/test.dll``` 这个非常重要，不然在链接的阶段会报错。make 完之后，会产生 dll_caller_test.exe 可执行程序，直接运行即可，当然正如上面提到的第三注意点，要保证 test.dll 动态库和 dll_caller_test.exe 在同一个目录，或者在 Windows 搜索的 dll 库的路径下。

## 通过 LoadLibrary 来打开指定路径的动态库来使用
第二种就是通过使用 LoadLibrary 来加载动态库的形式实现，这样可以将 test.dll 放在任意的路径下，而不需要关注其是否在 dll 搜索的路径下，因为很多时候，我们可能并没有相关的配置权限，这在 Linux 下相当有用，毕竟不是所有人都有 root 权限的。

### 调用程序的源码 main.c
```
#include <iostream>
#include <windows.h>
#include <wtypes.h>   
#include <winbase.h> 
// 定义函数指针，用于调用 动态库中的 void print_hello() 函数
typedef void (*ph) ();

using namespace std;
int main(int argc, char** argv) {
	cout <<"begin to call dll." << endl;
	
    // 1. 指定 dll 文件的路径，并进行加载
    HMODULE hDLL = LoadLibrary("../test/test.dll");
    if(hDLL == NULL) std::cout<<"Error!!!" << endl;

    // 2. 从 dll 的文件对象中，找到 print_hello 的函数
	ph p_h =(ph)GetProcAddress(hDLL,"print_hello");
    if(p_h == NULL)  std::cout<<"GetProcAddress Error!!!" << endl;

    // 3. 调用 print_hello 的函数
  	p_h();
      
    // 4. 释放动态库
    FreeLibrary(hDLL);
	
	return 0;
}
```

### Makefile 文件的内容如下
```
# Project: dll_caller_test
# Makefile created by Dev-C++ 5.11

CPP      = g++.exe -D__DEBUG__
CC       = gcc.exe -D__DEBUG__
WINDRES  = windres.exe
OBJ      = main.o
LINKOBJ  = main.o
LIBS     = -L"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib" -L"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/lib" -static-libgcc -g3
INCS     = -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include"
CXXINCS  = -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/x86_64-w64-mingw32/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include" -I"C:/Program Files (x86)/Dev-Cpp/MinGW64/lib/gcc/x86_64-w64-mingw32/4.9.2/include/c++"
BIN      = dll_caller_test.exe
CXXFLAGS = $(CXXINCS) -g3
CFLAGS   = $(INCS) -g3
RM       = rm.exe -f

.PHONY: all all-before all-after clean clean-custom

all: all-before $(BIN) all-after

clean: clean-custom
	${RM} $(OBJ) $(BIN)

$(BIN): $(OBJ)
	$(CPP) $(LINKOBJ) -o $(BIN) $(LIBS)

main.o: main.cpp
	$(CPP) -c main.cpp -o main.o $(CXXFLAGS)
```

从上述的代码看起来，调用的过程相比直接 include 头文件的情况还是稍微复杂一点的，不过过程还是很清晰的，简单明了。虽然代码看起来稍微复杂一点，但是，我们在编译的时候，完全不需要关注 ```动态库的头文件``` 以及 ```dll 文件```在什么地方了，是不是也带来了一些什么方便呢？哈哈。。。

运行可执行文件的效果和前面是一样的。