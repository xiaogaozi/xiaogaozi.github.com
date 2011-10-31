---
layout: post
title: "GYP 简介"
date: 2011-10-29 22:55
comments: true
categories: [Autoconf, CMake, GYP, SCons]
---

说起项目构建工具，Linux 用户最熟悉的恐怕就是 [Autoconf](http://www.gnu.org/s/autoconf/)，它将编译安装这个步骤大大简化。但对于项目作者来说，想要使用 Autoconf 生成有效的配置文件着实需要下一番功夫，用现在流行的话来说就是用户体验不够友好。对 Unix shell 的依赖，也使得 Autoconf 天生对于跨平台支持不佳。

后来我从[大猫](http://twitter.com/zhzhxtrrk)同学那里听说了 [CMake](http://www.cmake.org/)，CMake 使用 C++ 编写，原生支持跨平台，不需要像 Autoconf 那样写一堆的配置文件，只需一个 CMakeLists.txt 文件即可。简洁的使用方式，强大的功能使得我立马对 CMake 情有独钟。在后来的使用过程中，虽然会遇到一些因为使用习惯带来的小困扰，但我对于 CMake 还是基本满意的。直到我发现了 GYP。

[GYP](http://code.google.com/p/gyp/)（Generate Your Projects）是由 Chromium 团队开发的跨平台自动化项目构建工具，Chromium 便是通过 GYP 进行项目构建管理。为什么我要选择 GYP，而放弃 CMake 呢？功能上 GYP 和 CMake 很是相似，在我看来，它们的最大区别在于配置文件的编写方式和其中蕴含的思想。

编写 CMake 配置文件相比 Autoconf 来说已经简化很多，一个最简单的配置文件只需要写上源文件及生成类型（可执行文件、静态库、动态库等）即可。对分支语句和循环语句的支持也使得 CMake 更加灵活。但是，CMake 最大的问题也是在这个配置文件，请看下面这个示例文件：

{% codeblock lang:cmake %}
cmake_minimum_required(VERSION 2.8)
project(VP8 CXX)

add_definitions(-Wall)
cmake_policy(SET CMP0015 NEW)
include_directories("include")
link_directories("lib")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "../lib")
set(VP8SRC VP8Encoder.cpp VP8Decoder.cpp)

if(X86)
    set(CMAKE_SYSTEM_NAME Darwin)
    set(CMAKE_SYSTEM_PROCESSOR i386)
    set(CMAKE_OSX_ARCHITECTURES "i386")

    add_library(vp8 STATIC ${VP8SRC})
elseif(IPHONE)
    if(SIMULATOR)
        set(PLATFORM "iPhoneSimulator")
        set(PROCESSOR i386)
        set(ARCH "i386")
    else()
        set(PLATFORM "iPhoneOS")
        set(PROCESSOR arm)
        set(ARCH "armv7")
    endif()

    set(SDKVER "4.0")
    set(DEVROOT "/Developer/Platforms/${PLATFORM}.platform/Developer")
    set(SDKROOT "${DEVROOT}/SDKs/${PLATFORM}${SDKVER}.sdk")
    set(CMAKE_OSX_SYSROOT "${SDKROOT}")
    set(CMAKE_SYSTEM_NAME Generic)
    set(CMAKE_SYSTEM_PROCESSOR ${PROCESSOR})
    set(CMAKE_CXX_COMPILER "${DEVROOT}/usr/bin/g++")
    set(CMAKE_OSX_ARCHITECTURES ${ARCH})

    include_directories(SYSTEM "${SDKROOT}/usr/include")
    link_directories(SYSTEM "${SDKROOT}/usr/lib")

    add_definitions(-D_PHONE)
    add_library(vp8-armv7-darwin STATIC ${VP8SRC})
endif()
{% endcodeblock %}

你能一眼看出这个配置文件干了什么吗？其实这个配置文件想要产生的目标（target）只有一个，就是通过 `${VP8SRC}` 编译生成的静态库，但因为加上了条件判断，及各种平台相关配置，使得这个配置文件看起来很是复杂。在我看来，编写 CMake 配置文件是一种线性思维，对于同一个目标的配置可能会零散分布在各个地方。而 GYP 则相当不同，GYP 的配置文件更多地强调模块化、结构化。看看下面这个示例文件：

{% codeblock lang:python %}
{
  'targets': [
    {
      'target_name': 'foo',
      'type': '<(library)',
      'dependencies': [
        'bar',
      ],
      'defines': [
        'DEFINE_FOO',
        'DEFINE_A_VALUE=value',
      ],
      'include_dirs': [
        '..',
      ],
      'sources': [
        'file1.cc',
        'file2.cc',
      ],
      'conditions': [
        ['OS=="linux"', {
          'defines': [
            'LINUX_DEFINE',
          ],
          'include_dirs': [
            'include/linux',
          ],
        }],
        ['OS=="win"', {
          'defines': [
            'WINDOWS_SPECIFIC_DEFINE',
          ],
        }, { # OS != "win",
          'defines': [
            'NON_WINDOWS_DEFINE',
          ],
        }]
      ],
    } 
  ],
}
{% endcodeblock %}

我们可以立马看出上面这个配置文件的输出目标只有一个，也就是 `foo`，它是一个库文件（至于是静态的还是动态的这需要在生成项目时指定），它依赖的目标、宏定义、包含的头文件路径、源文件是什么，以及根据不同平台设定的不同配置等。这种定义配置文件的方式相比 CMake 来说，让我觉得更加舒服，也更加清晰，特别是当一个输出目标的配置越来越多时，使用 CMake 来管理可能会愈加混乱。

配置文件的编写方式是我区分 GYP 和 CMake 之间最大的不同点，当然 GYP 也有一些小细节值得注意，比如支持跨平台项目工程文件输出，Windows 平台默认是 Visual Studio，Linux 平台默认是 Makefile，Mac 平台默认是 Xcode，这个功能 CMake 也同样支持，只是缺少了 Xcode。Chromium 团队成员也撰文详细[比较](http://code.google.com/p/gyp/wiki/GypVsCMake)了 GYP 和 CMake 之间的优缺点，在开发 GYP 之前，他们也曾试图转到 [SCons](http://www.scons.org/)（这个我没用过，有经验的同学可以比较一下），但是失败了，于是 GYP 就诞生了。

当然 GYP 也不是没有缺点，相反，我觉得它的“缺点”一大堆：

- 文档不够完整，项目不够正式，某些地方还保留着 Chromium 的影子，看起来像是还没有完全独立出来。
- 大量的括号嵌套，很容易让人看晕，有过 Lisp 使用经验的同学可以对号入座。对于有括号恐惧症，或者不使用现代编辑器的同学基本可以绕行。
- 为了支持跨平台，有时不得不加入某些特定平台的配置信息，比如只适用于 Visual Studio 的 `RuntimeLibrary` 配置，这不利于跨平台配置文件的编写，也无形中增加了编写复杂度。
- 不支持 `make clean`，唯一的方法就是将输出目录整个删除或者手动删除其中的某些文件。

如果你已经打算尝试 GYP，那一定记得在生成项目工程文件时加上 `--depth` 参数，譬如：

    $ gyp --depth=. foo.gyp

这也是一个从 Chromium 项目遗留下来的历史问题。

也许你根本用不上跨平台特性，但是 GYP 依然值得尝试。我编写了一份 GYP 配置文件的[模板](http://code.google.com/p/princess-alist/source/browse/trunk/home/xiaogaozi/.templates/TEMPLATE.gyp.tpl)，有兴趣的同学可以参考。GYP 和 CMake 分别代表了两种迥异的“风格”，至于孰优孰劣，还得仁者见仁，智者见智。
