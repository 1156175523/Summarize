# **测试环境 Linux x86_64(192.168.133.41)**
# ***MinGW工具链(x64为例) 对外C语言接口(C++接口不同编译器命名规则不一致)***
## 1. 安装MinGW编译工具链
```
yum install 如下工具包
mingw64-gcc.x86_64
mingw64-gcc-c++.x86_64
```
### 验证
```
x86_64-w64-mingw32-gcc --version
x86_64-w64-mingw32-g++ --version
```
## 2. 创建编译工具链配置文件
```
└── toolchains
		└── mingw_x64_toolchain.cmake  -- 自定义编译工具链文件名
```
建议创建toolchains目录存放,方便后期扩展不同编译工具链
文件mingw\_x64\_toolchain.cmake内容如下:
```
##############################
## winx86-x64 mingw编译工具链
set(CMAKE_SYSTEM_NAME Windows)				#目标系统名
set(CMAKE_SYSTEM_PROCESSOR x86_64)			#系统架构

set(CMAKE_C_COMPILER /usr/bin/x86_64-w64-mingw32-gcc)		#指定C编译器
set(CMAKE_CXX_COMPILER /usr/bin/x86_64-w64-mingw32-g++)		#指定C++编译器
set(CMAKE_RC_COMPILER /usr/bin/x86_64-w64-mingw32-windres)	#指定资源文件编译器

## 交叉编译(cross-compiling)时用于设置查找路径优先级的重要配置
set(CMAKE_FIND_ROOT_PATH /usr/x86_64-w64-mingw32)				#当查找库(library)和头文件(include)时,优先从这个目录下找

#不要在 CMAKE_FIND_ROOT_PATH 下找可执行程序(编译器、工具)
#意思是 find_program() 查找 gcc, windres, 等程序时,仍然从系统路径(如 /usr/bin)查找
set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)

#查找库（find_library()）只从 CMAKE_FIND_ROOT_PATH 指定的路径中找
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)

#查找头文件（find_path()）只从 CMAKE_FIND_ROOT_PATH 中找
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
##############################
```
## 3、交叉编译时
```
mkdir build-win64 \&\& cd build-win64
cmake .. -DCMAKE_TOOLCHAIN_FILE=../toolchains/mingw_toolchain.cmake
(本机编译直接运行: cmake ..)
```

### ###############针对MinGW交叉编译生成库问题###############

### *1. 针对生成静态库*
&nbsp;&nbsp;&nbsp;&nbsp; 生成的文件为yourlib.a文件,Windows中只能通过MinGW使用  
### *2. 针对生成动态库情况,动态库不能直接被Visual Studio使用(MinGW可以直接使用)*
&nbsp;&nbsp;&nbsp;&nbsp; 生成的文件为 yourlib.dll 和 yourlib.dll.a  
  
&nbsp;&nbsp;&nbsp;&nbsp; ***解决Visual Studio使用问题***:  
&nbsp;&nbsp;&nbsp;&nbsp; **思路**: 生成.def文件(目前选择CMake自动生成),通过dlltool生成.lib文件,向VS提供.dll和.lib  
&nbsp;&nbsp;&nbsp;&nbsp; **步骤**:
```
1. CMakeLists.txt 文件添加如下set配置,自动生成.def文件  
    add_library(${META_PROJECT_NAME} ${META_LIB_TYPE} ${META_SRC_LISTS})
    #@注意后边两个配置必须在add_library之后
    set_target_properties(${META_PROJECT_NAME} PROPERTIES WINDOWS_EXPORT_ALL_SYMBOLS ON) #只针对MinGW编译器,生成定义描述符(其他编译器会忽略这行)
    set_target_properties(${META_PROJECT_NAME} PROPERTIES LINK_FLAGS "-Wl,--output-def,${CMAKE_BINARY_DIR}/mylib.def") #这里可以自定义路径和文件名

2. 使用dlltool生成.lib
    x86_64-w64-mingw32-dlltool -d yourlib.def -l yourlib.lib -D yourlib.dll
```


### 对外接口定义格式
```C
#if defined(_WIN32)         // Windows
    #define MY_API extern "C" __declspec(dllimport)
#elif defined(__linux__)    // Linux
    #define __stdcall
    #ifdef __cplusplus
        #define MY_API extern "C"
    #else
        #define MY_API
    #endif
#endif

MY_API int __stdcall MY_FUNC1(int *tag);
...
```
