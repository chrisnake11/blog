---
title: vs环境下CMake编译不加载资源文件
published: 2025-08-23T16:52:48Z
description: '将代码从vscode搬到了vs2022中使用CMake进行编译。并且将代码分类到不同的文件夹中，出现了不加载资源文件的情况。'
image: ''
tags: [ChatRoom, C++, QT, CMake]
category: 'ChatRoom'
draft: false
---

# vs环境下CMake编译不加载资源文件

我将代码从vscode搬到了vs2022中使用CMake进行编译。并且将代码分类到不同的文件夹中。

我的代码目录结构为:  
```
+ ChatRoom
    + resources/
        + images/
        + styles/
    + source/
        + *.cpp
    + header/
        + *.h
    + ui/
        + *.ui
    + resources.qrc
    + CMakeLists.txt
```


对应的CMakeList.txt如下:
```cmake
cmake_minimum_required(VERSION 3.16)
project(ChatRoom LANGUAGES CXX)

include(qt.cmake)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(MSVC)
    add_compile_options(/utf-8)
endif()

find_package(QT NAMES Qt6 Qt5 REQUIRED COMPONENTS Core Gui Widgets Network)
find_package(Qt${QT_VERSION_MAJOR}
    COMPONENTS
        Core
        Gui
        Widgets
        Network
)
qt_standard_project_setup()

# 包含代码文件夹，方便预编译检索
include_directories(${CMAKE_SOURCE_DIR}/header)
include_directories(${CMAKE_SOURCE_DIR}/source)
include_directories(${CMAKE_SOURCE_DIR}/ui)


# 自动收集所有源文件、头文件、UI文件
file(GLOB SOURCES "${CMAKE_SOURCE_DIR}/source/*.cpp")
file(GLOB HEADERS "${CMAKE_SOURCE_DIR}/header/*.h")
file(GLOB UIS "${CMAKE_SOURCE_DIR}/ui/*.ui")

set(PROJECT_SOURCES ${SOURCES} ${HEADERS} ${UIS})
set(CMAKE_AUTOUIC_SEARCH_PATHS "${CMAKE_SOURCE_DIR}/ui")

# QT6 手动添加qrc文件
qt_add_resources(APP_RESOURCES resources.qrc)

# 生成可执行文件
qt_add_executable(${PROJECT_NAME} ${PROJECT_SOURCES} ${APP_RESOURCES})

set_target_properties(${PROJECT_NAME}
    PROPERTIES
        WIN32_EXECUTABLE TRUE
)

target_link_libraries(${PROJECT_NAME}
    PUBLIC
        Qt::Core
        Qt::Gui
        Qt::Widgets
        Qt::Network
)


# 拷贝 resources 目录
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_directory
    ${CMAKE_SOURCE_DIR}/resources
    $<TARGET_FILE_DIR:${PROJECT_NAME}>/resources
)

# 复制配置文件 config.ini 到可执行文件目录
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
    ${CMAKE_SOURCE_DIR}/config.ini
    $<TARGET_FILE_DIR:${PROJECT_NAME}>
)
```

编译成功后，出现了不加载资源文件的情况。原因有2点。

1. qrc文件没有正确地被编译。

错误的代码，QT6下直接将qrc添加到add_executable()中，容易导致资源文件没有被正确编译地问题，需要调用add_qt_resources()来处理qrc文件。

错误代码：

```cmake
# 创建可执行文件，包含源文件、头文件、UI文件和资源文件
add_executable(${PROJECT_NAME}
    ${SOURCES}
    ${HEADERS}
    ${UIS}
    resources.qrc # 添加资源文件
)
```

正确代码：

```cmake
set(PROJECT_SOURCES ${SOURCES} ${HEADERS} ${UIS})

# QT6 手动添加qrc文件
qt_add_resources(APP_RESOURCES resources.qrc)

# 生成可执行文件
qt_add_executable(${PROJECT_NAME} ${PROJECT_SOURCES} ${APP_RESOURCES})
```

2. qrc文件中地注释存在问题。

在我之前的qrc代码中，开头有一大段注释，导致XML解析失败，资源文件无法被正确加载。

```xml
<!--
    代码访问路径： ": + prefix + alias"
    例如：":/images/wechat.png"

    实际文件索引路径：<file> real_path </file>
    例如：<file alias="wechat.png">resources/images/wechat.png</file>
    QT Designer 会根据<file>标签内的路径，去项目文件夹下寻找对应的资源文件

    优点：这样如果文件位置变动，只需修改<file>标签内的路径，代码中访问路径不变
-->
<RCC>
    <qresource prefix="/">
        ...
    </qresource>
</RCC>
```

