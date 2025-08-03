---
title: CMake编译项目
published: 2025-08-03T14:11:29Z
description: ''
image: ''
tags: [ChatRoom, CMake, C++]
category: 'ChatRoom'
draft: false
---

# CMake编译项目

使用CMake来编译ChatRoom项目，避免在直接使用QT Creator的qmake时，出现文件丢失无法找到等奇奇怪怪的问题。并且项目结构更加清晰，易于维护。

## 项目目录

```bash
$ ls
build/            config.ini       logindialog.h   registerdialog.cpp  resources/      timerbtn.h           UserManager.h
ChatDialog.cpp    desktop.ini      logindialog.ui  registerdialog.h    resources.qrc   ui_ChatDialog.h
ChatDialog.h      global.cpp       main.cpp        registerdialog.ui   singleton.h     ui_logindialog.h
ChatDialog.ui     global.h         mainwindow.cpp  resetdialog.cpp     styles/         ui_mainwindow.h
clickedlabel.cpp  httpmanager.cpp  mainwindow.h    resetdialog.h       TcpManager.cpp  ui_registerdialog.h
clickedlabel.h    httpmanager.h    mainwindow.ui   resetdialog.ui      TcpManager.h    ui_resetdialog.h
CMakeLists.txt    logindialog.cpp  RCa31408        resource.h          timerbtn.cpp    UserManager.cpp

$ ls resources
avatar.png  eye.png  eye-close.png  eye-hover.png  eye-hover-close.png  wechat.png

$ ls styles/
stylesheet.qss
```



## CMakeLists.txt

```cmake
# 指定CMake最低版本要求
cmake_minimum_required(VERSION 3.16)

# 设置项目名称和使用的语言
project(ChatRoom2 LANGUAGES CXX)

# 设置C++标准为C++17
set(CMAKE_CXX_STANDARD 17)

# 启用Qt的自动MOC、RCC和UIC功能，自动处理元对象、资源和UI文件
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_AUTOUIC ON)

# 查找Qt6所需模块（Core、Gui、Widgets、Network）
find_package(Qt6 REQUIRED COMPONENTS Core Gui Widgets Network)

# 列出所有源文件
set(SOURCES
    main.cpp
    mainwindow.cpp
    logindialog.cpp
    registerdialog.cpp
    resetdialog.cpp
    timerbtn.cpp
    httpmanager.cpp
    TcpManager.cpp
    ChatDialog.cpp
    global.cpp
    clickedlabel.cpp
    UserManager.cpp
)

# 列出所有头文件
set(HEADERS
    mainwindow.h
    logindialog.h
    registerdialog.h
    resetdialog.h
    timerbtn.h
    httpmanager.h
    TcpManager.h
    ChatDialog.h
    global.h
    singleton.h
    UserManager.h
    clickedlabel.h
)

# 列出所有UI文件
set(UIS
    mainwindow.ui
    logindialog.ui
    registerdialog.ui
    resetdialog.ui
    ChatDialog.ui
)

# 创建可执行文件，包含源文件、头文件、UI文件和资源文件
add_executable(${PROJECT_NAME}
    ${SOURCES}
    ${HEADERS}
    ${UIS}
    resources.qrc # 添加资源文件
)   

# 链接Qt6库
target_link_libraries(${PROJECT_NAME}
    Qt6::Core
    Qt6::Gui
    Qt6::Widgets
    Qt6::Network
)

# 收集resources下的所有png文件
file(GLOB RESOURCE_IMAGES "${CMAKE_SOURCE_DIR}/resources/*.png")

set(RESOURCE_FILES
    ${RESOURCE_IMAGES}
)

# 将资源文件复制到可执行文件目录
foreach(RES_FILE ${RESOURCE_FILES})
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E make_directory
        $<TARGET_FILE_DIR:${PROJECT_NAME}>/resources
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
        ${RES_FILE}
        $<TARGET_FILE_DIR:${PROJECT_NAME}>/resources/
    )
endforeach()

# 单独复制配置文件 config.ini 到可执行文件目录
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
    ${CMAKE_SOURCE_DIR}/config.ini
    $<TARGET_FILE_DIR:${PROJECT_NAME}>
)

# 复制样式表文件到可执行文件目录
add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
    ${CMAKE_SOURCE_DIR}/styles/stylesheet.qss
    $<TARGET_FILE_DIR:${PROJECT_NAME}>
)

# 如果有其他dll文件需要复制，可以继续添加
```

## 编译命令

```bash
# 在源代码目录下
mkdir build

## 进入build目录, 使用MSVC 2022编译器
# 注意：需要根据实际的Qt安装路径修改CMAKE_PREFIX_PATH
cd build && cmake .. -DCMAKE_PREFIX_PATH="D:/dev/Qt6.9/6.9.0/msvc2022_64"

# 生成Makefile或项目文件
# 如果使用Visual Studio，可以直接打开生成的.sln文件
cmake --build . --config Debug

# 使用windeployqt将Qt依赖库复制到可执行文件目录
"/d/dev/Qt6.9/6.9.0/msvc2022_64/bin/windeployqt.exe" ./Debug/ChatRoom2.exe

# 运行程序
./Debug/ChatRoom2.exe
```

## 资源文件问题

在`add_executable`中需要添加`resource.qrc`文件，这样CMake才能正确处理资源文件。

随后，在CMake中使用`file(GLOB ...)`来收集资源文件，避免手动拷贝每个资源文件。

并使用`add_custom_command`在构建后将资源文件复制到可执行文件目录，确保资源文件在运行时可用。

可执行文件目录的资源文件目录，应当与源代码目录的资源文件目录结构一致。