# 编写CMakeLists.txt最常用的功能就是调用其他的.h头文件和.so/.a库文件，
# 将.cpp/.c/.cc文件编译成可执行文件或者新的库文件。
# CMakeLists for libfacedetectcnn

# 本CMakeLists.txt的project名称
# 会自动创建两个变量，PROJECT_SOURCE_DIR和PROJECT_NAME
# ${PROJECT_SOURCE_DIR}：本CMakeLists.txt所在的文件夹路径
# ${PROJECT_NAME}：本CMakeLists.txt的project名称  这里的工程名称为 libfacedetection
project(libfacedetection)

# cmake的版本要求
cmake_minimum_required(VERSION 2.8)

# option语句使用场景 编译脚本传递参数 -> CMake脚本接收option -> 源代码宏
# cmake选项设置 参数名 说明语句 设置默认值
# 这里主要是设置三个指令集开关 demo开关 以及OpenMP开关
option(ENABLE_NEON "whether use neon, if use arm please set it on" OFF)
option(ENABLE_AVX512 "use avx512" OFF)
option(ENABLE_AVX2 "use avx2" ON)
option(DEMO "build the demo" OFF)
option(USE_OPENMP "Use OpenMP" ON)

# BUILD_SHARED_LIBS 指定用于生成动态库的参数
if (BUILD_SHARED_LIBS)
    # add_definitions添加编译参数,这里添加BUILD_SHARED_LIBS参数
	add_definitions(-DBUILD_SHARED_LIBS)
    if (CMAKE_COMPILER_IS_GNUCXX AND NOT MINGW)
       # Just setting CMAKE_POSITION_INDEPENDENT_CODE should be enough to set
       # -fPIC for GCC but sometimes it still doesn't get set, so make sure it
       # does.
       # 添加-fPIC参数 用于生成位置无关的代码
       add_definitions("-fPIC")
    endif()
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()

# 设置版本号
SET(BUILD_VERSION "v0.0.1")
# 寻找git版本
# Find Git Version Patch 
# github上clone下来的项目会自带一个.git的隐藏文件
IF(EXISTS "${CMAKE_SOURCE_DIR}/.git")
    if(NOT GIT)
        SET(GIT $ENV{GIT})
    endif()
    if(NOT GIT)
        # 搜索一个外部的可执行程序
        FIND_PROGRAM(GIT NAMES git git.exe git.cmd)
    endif()
    IF(GIT)
        # 执行一个或多个子进程，按指定的先后顺序运行一个或多个命令
        EXECUTE_PROCESS(
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            COMMAND ${GIT} describe --tags
            OUTPUT_VARIABLE GIT_VERSION  OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        if(NOT GIT_VERSION)
            EXECUTE_PROCESS(
                WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
                COMMAND ${GIT} rev-parse --short HEAD
                OUTPUT_VARIABLE GIT_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE
            )
        endif()
        IF(NOT GIT_VERSION)
            SET(BUILD_VERSION ${GIT_VERSION})
	ENDIF()
    ENDIF()
ENDIF()
# 打印消息，在控制台或者对话框输出一行或多行调试信息
message("BUILD_VERSION:${BUILD_VERSION}")

# 设置目录关系
SET(fdt_base_dir   ${PROJECT_SOURCE_DIR})
SET(fdt_src_dir    ${fdt_base_dir}/src)
SET(fdt_inc_dir    ${fdt_base_dir}/src)

SET(fdt_lib_name   facedetection)
# GLOB_RECURSE 会遍历匹配目录的所有文件以及子目录下面的文件
FILE(GLOB_RECURSE fdt_source_files ${fdt_src_dir}/*.cpp)
LIST(SORT         fdt_source_files)

SET(INSTALLHEADER_FILES ${fdt_inc_dir}/facedetectcnn.h)

# 设置c++标准
set(CMAKE_CXX_STANDARD 11)
# 如果编译器不支持当前的c++标准，那么cmake会选择编译器支持的旧版本来编译程序。
# 设置CMAKE_CXX_STANDARD_REQUIRED为ON可以阻止这种情况的发生
set(CMAKE_CXX_STANDARD_REQUIRED ON) 
# 是否启用编译器扩展
set(CMAKE_CXX_EXTENSIONS OFF) 

IF(MSVC)
    # This option is to enable the /MP switch for Visual Studio 2005 and above compilers
    OPTION(WIN32_USE_MP "Set to ON to build with the /MP option (Visual Studio 2005 and above)." ON)
    IF(WIN32_USE_MP)
        #SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
        # add_compile_options 添加的编译选项是针对所有编译器的(包括C和C++编译器)
        # set命令设置CMAKE_C_FLAGS或CMAKE_CXX_FLAGS变量则是分别只针对C和C++编译器的
        add_compile_options(/MP)
    ENDIF(WIN32_USE_MP)
    # 设置MSVC编译编码
    add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
    add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
ENDIF(MSVC)

# 判断编译器类型
# CMAKE_SYSTEM_NAME 判断操作系统
IF(CMAKE_CXX_COMPILER_ID STREQUAL "GNU"
    OR CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    # 指定编译选项
    #use -O3 to speedup
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3")
ENDIF()

if(ENABLE_AVX512)
    # 添加编译参数 ENABLE_AVX512
	add_definitions(-D_ENABLE_AVX512)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx512bw")
endif()

if(ENABLE_AVX2)
	add_definitions(-D_ENABLE_AVX2)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx2 -mfma")
endif()

if(ENABLE_NEON)
	message("Using ENON")
	add_definitions(-D_ENABLE_NEON)
endif()

if(USE_OPENMP)
    # 在指定的模块目录中搜索一个名为Find<name>.cmake(例如，FindOSG.cmake)的
    # CMake脚本模块文件，执行其中的内容，意图搜索到指定的外部依赖库头文件和库文件位置
    FIND_PACKAGE(OpenMP)
    if(OPENMP_FOUND)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
    endif()
endif()
# 指定头文件的搜索路径，用来向工程添加多个特定的头文件搜索路径，可以多次调用以设置多个路径，相当于指定gcc的-I参数
INCLUDE_DIRECTORIES(${fdt_inc_dir})

# 写入库的版本信息
include(CMakePackageConfigHelpers)
include(GenerateExportHeader)
# CMAKE_INSTALL_*DIR定义的变量GNUInstallDirs
include(GNUInstallDirs)

# 可以设置要生成的链接库为SHARED或者STATIC，还可以设置MODULE(插件，可动态调用，但不作为其它工程的依赖)
# Create a library
ADD_LIBRARY(${fdt_lib_name} ${fdt_source_files} ${INSTALLHEADER_FILES})

# 用来生成dllexport头文件
# Generate export header
GENERATE_EXPORT_HEADER(${fdt_lib_name})
string(TOLOWER ${fdt_lib_name} LOWER_PROJECT_NAME)
set(INSTALLHEADER_FILES ${INSTALLHEADER_FILES} 
    ${CMAKE_CURRENT_BINARY_DIR}/${LOWER_PROJECT_NAME}_export.h)
# COPY表示复制文件，目录以及符号链接到一个目标文件夹中。
file(COPY ${CMAKE_CURRENT_BINARY_DIR}/${LOWER_PROJECT_NAME}_export.h
    DESTINATION ${CMAKE_BINARY_DIR})

include_directories(${fdt_lib_name} ${CMAKE_BINARY_DIR})

# 用来设置输出的名称，对于动态库，还可以用来指定动态库版本和API版本
set_target_properties(${fdt_lib_name} PROPERTIES
    PUBLIC_HEADER "${INSTALLHEADER_FILES}" # Install head files
    VERSION ${BUILD_VERSION}
   )

# Install target
INSTALL(TARGETS ${fdt_lib_name}
    EXPORT ${fdt_lib_name}Config
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/facedetection
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
export(TARGETS ${fdt_lib_name}
       APPEND FILE ${CMAKE_BINARY_DIR}/${fdt_lib_name}Config.cmake
)
# Install cmake configure files
install(EXPORT ${fdt_lib_name}Config
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${fdt_lib_name}"
        )
write_basic_package_version_file(
    "${CMAKE_BINARY_DIR}/${fdt_lib_name}ConfigVersion.cmake"
    VERSION ${BUILD_VERSION}
    COMPATIBILITY AnyNewerVersion)
install(FILES "${CMAKE_BINARY_DIR}/${fdt_lib_name}ConfigVersion.cmake" DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${fdt_lib_name}")

# demo配置
# Create demo. OpenCV is requred.
if (DEMO)
    find_package(OpenCV REQUIRED)
    include_directories(${OpenCV_INCLUDE_DIRS})

    add_executable(detect-image-demo ${fdt_base_dir}/example/detect-image.cpp)
    target_link_libraries(detect-image-demo ${fdt_lib_name} ${OpenCV_LIBS})

    add_executable(detect-camera-demo ${fdt_base_dir}/example/detect-camera.cpp)
    target_link_libraries(detect-camera-demo ${fdt_lib_name} ${OpenCV_LIBS})
    
    add_executable(benchmark ${fdt_base_dir}/example/benchmark.cpp)
    target_link_libraries(benchmark ${fdt_lib_name} ${OpenCV_LIBS})
endif()

if (GSTREAMER)
    find_package(OpenCV REQUIRED)

    include(FindPkgConfig)
    pkg_search_module(GSTREAMER REQUIRED gstreamer-1.0)
    pkg_search_module(GSTREAMER_BASE REQUIRED gstreamer-base-1.0)
    pkg_search_module(GSTREAMER_VIDEO REQUIRED gstreamer-video-1.0)

    add_library(gstfacedetect SHARED
        example/libfacedetect.cpp
    )

    include_directories(gstfacedetect PRIVATE
        ${GSTREAMER_INCLUDE_DIRS}
        ${GSTREAMER_BASE_INCLUDE_DIRS}
        ${GSTREAMER_VIDEO_INCLUDE_DIRS}
        ${OpenCV_INCLUDE_DIRS}
    )

    target_link_libraries(gstfacedetect
        ${GSTREAMER_LIBRARIES}
        ${GSTREAMER_BASE_LIBRARIES}
        ${GSTREAMER_VIDEO_LIBRARIES}
        ${OpenCV_LIBS}
        ${fdt_lib_shared}
    )

endif()

message("AVX512 = ${ENABLE_AVX512}")
message("AVX2 = ${ENABLE_AVX2}")
message("NEON = ${ENABLE_NEON}")
message("OpenMP = ${OPENMP_FOUND}")
message("DEMO = ${DEMO}")
