## Android采用fmod库实现变声效果

### 一、下载Fmod库

[Fmod库官网](https://www.fmod.com/)下载fmod库，上方有个download，点击进去，选择安卓版本

### 二、创建c/c++工程
1. 倒入Fmod库到工程中


2. 编写Cmake文件
```cmake
cmake_minimum_required(VERSION 3.4.1)
#设置库路径
set(my_lib_path ${CMAKE_SOURCE_DIR}/../../../libs)

add_library(changeVoice
        SHARED
        VoiceChangeImpl.cpp VoiceChange.cpp )

#添加第三方库
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -L${my_lib_path}/${ANDROID_ABI}")


#添加头文件
include_directories( inc)

find_library( log-lib
              log )

target_link_libraries(changeVoice
                        fmod        #注意库的名字
                        fmodL
                       ${log-lib} )

```
3. 配置gradle
```gradle
android {
    compileSdkVersion 29
    buildToolsVersion "29.0.3"

    defaultConfig {
        minSdkVersion 19
        targetSdkVersion 29
        versionCode 1
        versionName "1.0"

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        consumerProguardFiles 'consumer-rules.pro'
        externalNativeBuild {
            cmake {
                cppFlags "-frtti -fexceptions"
            }
        }
        ndk {
           // abiFilters "arm64-v8a","armeabi-v7a","x86"
        }
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }
    externalNativeBuild {
        cmake {
            path file('src/main/cpp/CMakeLists.txt')
        }
    }
    sourceSets.main {
        jniLibs.srcDirs = ['libs']
    }
}

```
4. 编写代码
```


```

**播放声音基本上有五个步骤**

1. System_Create创建一个system
2. init初始化
3. createSound创建一个声音
4. playSound播放声音
5. system->update();执行后声音才能播放出去。

第四步->第五步之间可以添加一些声音的特殊处理。


