### 一、Android OpenSL ES 介绍
OpenSL ES (Open Sound Library for Embedded Systems)是无授权费、跨平台、针对嵌入式系统精心优化的硬件音频加速API。它为嵌入式移动多媒体设备上的本地应用程序开发者提供标准化, 高性能,低响应时间的音频功能实现方法，并实现软/硬件音频性能的直接跨平台部署，降低执行难度，促进高级音频市场的发展。简单来说OpenSL ES是一个嵌入式跨平台免费的音频处理库。 

Android的OpenSL ES库是在NDK的platforms文件夹对应android平台先相应cpu类型里面，如：ndk库/platforms/android-24/arch-arm/usr/lib

## 二、Android OpenSL ES 开发流程
OpenSL ES 的开发流程主要有如下6个步骤：

- 1、 创建接口对象
- 2、设置混音器
- 3、创建播放器（录音器）
- 4、设置缓冲队列和回调函数
- 5、设置播放状态
- 6、启动回调函数

> 注明：其中第4步和第6步是OpenSL ES 播放PCM等数据格式的音频是需要用到的。

在使用OpenSL ES的API之前，需要引入OpenSL ES的头文件，代码如下：
```
#include <SLES/OpenSLES.h>
#include <SLES/OpenSLES_Android.h>
```

由于是在Native层使用该特性，所需需要在Android.mk中增加链接选项，以便在链接阶段使用到系统系统的OpenSL ES的so库：
```
LOCAL_LDLIBS += -lOepnSLES
```

我们知道OpenSL ES提供的是基于C语言的API，但是它是基于对象和接口的方式提供的，会采用面向对象的思想开发API。因此我们先来了解一下OpenSL ES中对象和接口的概念：

- 对象：对象是对一组资源及其状态的抽象，每个对象都有一个在其创建时指定的类型，类型决定了对象可以执行的任务集，对象有点类似于C++中类的概念。
- 接口：接口是对象提供的一组特征的抽象，这些抽象会为开发者提供一组方法以及每个接口的类型功能，在代码中，接口的类型由接口ID来标识。

需要重点理解的是，一个对象在代码中其实是没有实际的表示形式的，可以通过接口来改变对象的状态以及使用对象提供的功能。对象有可以有一个或者多个接口的实例，但是接口实例肯定只属于一个对象。

如果明白了OpenSL ES 中对象和接口的概念，那么下面我们就继续看看，在代码中是如何使用它们的。

上面我们也提到过，对象是没有实际的代码表示形式的，对象的创建也是通过接口来完成的。通过获取对象的方法来获取出对象，进而可以访问对象的其他的接口方法或者改变对象的状态，下面是使用对象和接口的相关说明。

### 2.1 OpenSL ES 开发最重要的接口类 SLObjectItf
 通过SLObjectItf接口类我们可以创建所需要的各种类型的类接口，比如：

- 创建引擎接口对象：SLObjectItf engineObject
- 创建混音器接口对象：SLObjectItf outputMixObject
- 创建播放器接口对象：SLObjectItf 

以上等等都是通过SLObjectItf来创建的。

### 2.2 SLObjectItf 创建的具体的接口对象实例
OpenSL ES中也有具体的接口类，比如（引擎：SLEngineItf，播放器：SLPlayItf，声音控制器：SLVolumeItf等等）。

### 2.3 创建引擎并实现
OpenSL ES中开始的第一步都是声明SLObjectItf接口类型的引擎接口对象engineObject，然后用方法slCreateEngine创建一个引擎接口对象；创建好引擎接口对象后，需要用SLObjectItf的Realize方法来实现engineObject；最后用SLObjectItf的GetInterface方法来初始化SLEngnineItf对象实例。如：
```C
SLObjectItf engineObject = NULL;//用SLObjectItf声明引擎接口对象
SLEngineItf engineEngine = NULL;//声明具体的引擎对象实例
 
void createEngine()
{
    SLresult result;//返回结果
    result = slCreateEngine(&engineObject, 0, NULL, 0, NULL, NULL);//第一步创建引擎
    result = (*engineObject)->Realize(engineObject, SL_BOOLEAN_FALSE);//实现（Realize）engineObject接口对象
    result = (*engineObject)->GetInterface(engineObject, SL_IID_ENGINE, &engineEngine);//通过engineObject的GetInterface方法初始化engineEngine
}
```
### 2.4 利用引擎对象创建其他接口对象
其他接口对象（SLObjectItf outputMixObject，SLObjectItf playerObject）等都是用引擎接口对象创建的（具体的接口对象需要的参数这里就说了，可参照ndk例子里面的），如：
```
/混音器
SLObjectItf outputMixObject = NULL;//用SLObjectItf创建混音器接口对象
SLEnvironmentalReverbItf outputMixEnvironmentalReverb = NULL;////创建具体的混音器对象实例
 
result = (*engineEngine)->CreateOutputMix(engineEngine, &outputMixObject, 1, mids, mreq);//利用引擎接口对象创建混音器接口对象
result = (*outputMixObject)->Realize(outputMixObject, SL_BOOLEAN_FALSE);//实现（Realize）混音器接口对象
result = (*outputMixObject)->GetInterface(outputMixObject, SL_IID_ENVIRONMENTALREVERB, &outputMixEnvironmentalReverb);//利用混音器接口对象初始化具体混音器实例
 
//播放器
SLObjectItf playerObject = NULL;//用SLObjectItf创建播放器接口对象
SLPlayItf playerPlay = NULL;//创建具体的播放器对象实例
 
result = (*engineEngine)->CreateAudioPlayer(engineEngine, &playerObject, &audioSrc, &audioSnk, 3, ids, req);//利用引擎接口对象创建播放器接口对象
result = (*playerObject)->Realize(playerObject, SL_BOOLEAN_FALSE);//实现（Realize）播放器接口对象
result = (*playerObject)->GetInterface(playerObject, SL_IID_PLAY, &playerPlay);//初始化具体的播放器对象实例
```
最后就是使用创建好的具体对象实例来实现具体的功能。

## 三、OpenSL ES 使用示例

首先导入OpenSL ES和其他必须的库：
```
-lOpenSLES -landroid 

```

### 3.1 播放assets文件
创建引擎——>创建混音器——>创建播放器——>设置播放状态

```C
JNIEXPORT void JNICALL
Java_com_renhui_openslaudio_MainActivity_playAudioByOpenSL_1assets(JNIEnv *env, jobject instance, jobject assetManager, jstring filename) {
 
    release();
    const char *utf8 = (*env)->GetStringUTFChars(env, filename, NULL);
 
    // use asset manager to open asset by filename
    AAssetManager* mgr = AAssetManager_fromJava(env, assetManager);
    AAsset* asset = AAssetManager_open(mgr, utf8, AASSET_MODE_UNKNOWN);
    (*env)->ReleaseStringUTFChars(env, filename, utf8);
 
    // open asset as file descriptor
    off_t start, length;
    int fd = AAsset_openFileDescriptor(asset, &start, &length);
    AAsset_close(asset);
 
    SLresult result;
 
 
    //第一步，创建引擎
    createEngine();
 
    //第二步，创建混音器
    const SLInterfaceID mids[1] = {SL_IID_ENVIRONMENTALREVERB};
    const SLboolean mreq[1] = {SL_BOOLEAN_FALSE};
    result = (*engineEngine)->CreateOutputMix(engineEngine, &outputMixObject, 1, mids, mreq);
    (void)result;
    result = (*outputMixObject)->Realize(outputMixObject, SL_BOOLEAN_FALSE);
    (void)result;
    result = (*outputMixObject)->GetInterface(outputMixObject, SL_IID_ENVIRONMENTALREVERB, &outputMixEnvironmentalReverb);
    if (SL_RESULT_SUCCESS == result) {
        result = (*outputMixEnvironmentalReverb)->SetEnvironmentalReverbProperties(outputMixEnvironmentalReverb, &reverbSettings);
        (void)result;
    }
    //第三步，设置播放器参数和创建播放器
    // 1、 配置 audio source
    SLDataLocator_AndroidFD loc_fd = {SL_DATALOCATOR_ANDROIDFD, fd, start, length};
    SLDataFormat_MIME format_mime = {SL_DATAFORMAT_MIME, NULL, SL_CONTAINERTYPE_UNSPECIFIED};
    SLDataSource audioSrc = {&loc_fd, &format_mime};
 
    // 2、 配置 audio sink
    SLDataLocator_OutputMix loc_outmix = {SL_DATALOCATOR_OUTPUTMIX, outputMixObject};
    SLDataSink audioSnk = {&loc_outmix, NULL};
 
    // 创建播放器
    const SLInterfaceID ids[3] = {SL_IID_SEEK, SL_IID_MUTESOLO, SL_IID_VOLUME};
    const SLboolean req[3] = {SL_BOOLEAN_TRUE, SL_BOOLEAN_TRUE, SL_BOOLEAN_TRUE};
    result = (*engineEngine)->CreateAudioPlayer(engineEngine, &fdPlayerObject, &audioSrc, &audioSnk, 3, ids, req);
    (void)result;
 
    // 实现播放器
    result = (*fdPlayerObject)->Realize(fdPlayerObject, SL_BOOLEAN_FALSE);
    (void)result;
 
    // 得到播放器接口
    result = (*fdPlayerObject)->GetInterface(fdPlayerObject, SL_IID_PLAY, &fdPlayerPlay);
    (void)result;
 
    // 得到声音控制接口
    result = (*fdPlayerObject)->GetInterface(fdPlayerObject, SL_IID_VOLUME, &fdPlayerVolume);
    (void)result;
 
    //第四步，设置播放状态
    if (NULL != fdPlayerPlay) {
 
        result = (*fdPlayerPlay)->SetPlayState(fdPlayerPlay, SL_PLAYSTATE_PLAYING);
        (void)result;
    }
 
    //设置播放音量 （100 * -50：静音 ）
    (*fdPlayerVolume)->SetVolumeLevel(fdPlayerVolume, 20 * -50);
 
}

````

### 3.2 播放pcm文件
(集成到ffmpeg时，也是播放ffmpeg转换成的pcm格式的数据)，这里为了模拟是直接读取的pcm格式的音频文件。

1. 创建播放器和混音器

```c
 createEngine();
 
    //第二步，创建混音器
    const SLInterfaceID mids[1] = {SL_IID_ENVIRONMENTALREVERB};
    const SLboolean mreq[1] = {SL_BOOLEAN_FALSE};
    result = (*engineEngine)->CreateOutputMix(engineEngine, &outputMixObject, 1, mids, mreq);
    (void)result;
    result = (*outputMixObject)->Realize(outputMixObject, SL_BOOLEAN_FALSE);
    (void)result;
    result = (*outputMixObject)->GetInterface(outputMixObject, SL_IID_ENVIRONMENTALREVERB, &outputMixEnvironmentalReverb);
    if (SL_RESULT_SUCCESS == result) {
        result = (*outputMixEnvironmentalReverb)->SetEnvironmentalReverbProperties(
                outputMixEnvironmentalReverb, &reverbSettings);
        (void)result;
    }
    SLDataLocator_OutputMix outputMix = {SL_DATALOCATOR_OUTPUTMIX, outputMixObject};
    SLDataSink audioSnk = {&outputMix, NULL};

````

2. 设置pcm格式的频率位数等信息并创建播放器
```c
// 第三步，配置PCM格式信息
    SLDataLocator_AndroidSimpleBufferQueue android_queue={SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE,2};
    SLDataFormat_PCM pcm={
            SL_DATAFORMAT_PCM,//播放pcm格式的数据
            2,//2个声道（立体声）
            SL_SAMPLINGRATE_44_1,//44100hz的频率
            SL_PCMSAMPLEFORMAT_FIXED_16,//位数 16位
            SL_PCMSAMPLEFORMAT_FIXED_16,//和位数一致就行
            SL_SPEAKER_FRONT_LEFT | SL_SPEAKER_FRONT_RIGHT,//立体声（前左前右）
            SL_BYTEORDER_LITTLEENDIAN//结束标志
    };
    SLDataSource slDataSource = {&android_queue, &pcm};
 
 
    const SLInterfaceID ids[3] = {SL_IID_BUFFERQUEUE, SL_IID_EFFECTSEND, SL_IID_VOLUME};
    const SLboolean req[3] = {SL_BOOLEAN_TRUE, SL_BOOLEAN_TRUE, SL_BOOLEAN_TRUE};
    
    result = (*engineEngine)->CreateAudioPlayer(engineEngine, &pcmPlayerObject, &slDataSource, &audioSnk, 3, ids, req);
    //初始化播放器
    (*pcmPlayerObject)->Realize(pcmPlayerObject, SL_BOOLEAN_FALSE);
 
//    得到接口后调用  获取Player接口
    (*pcmPlayerObject)->GetInterface(pcmPlayerObject, SL_IID_PLAY, &pcmPlayerPlay);

```
3. 设置缓冲队列和回调函数
```c
//    注册回调缓冲区 获取缓冲队列接口
    (*pcmPlayerObject)->GetInterface(pcmPlayerObject, SL_IID_BUFFERQUEUE, &pcmBufferQueue);
    //缓冲接口回调
    (*pcmBufferQueue)->RegisterCallback(pcmBufferQueue, pcmBufferCallBack, NULL);
```

回调函数：

```c
void * pcmBufferCallBack(SLAndroidBufferQueueItf bf, void * context)
{
    //assert(NULL == context);
    getPcmData(&buffer);
    // for streaming playback, replace this test by logic to find and fill the next buffer
    if (NULL != buffer) {
        SLresult result;
        // enqueue another buffer
        result = (*pcmBufferQueue)->Enqueue(pcmBufferQueue, buffer, 44100 * 2 * 2);
        // the most likely other result is SL_RESULT_BUFFER_INSUFFICIENT,
        // which for this code example would indicate a programming error
    }
}

```

读取pcm格式的文件：

```c
void getPcmData(void **pcm)
{
    while(!feof(pcmFile))
    {
        fread(out_buffer, 44100 * 2 * 2, 1, pcmFile);
        if(out_buffer == NULL)
        {
            LOGI("%s", "read end");
            break;
        } else{
            LOGI("%s", "reading");
        }
        *pcm = out_buffer;
        break;
    }
}
```

4. 设置播放状态并手动开始调用回调函数

```c
//    获取播放状态接口
    (*pcmPlayerPlay)->SetPlayState(pcmPlayerPlay, SL_PLAYSTATE_PLAYING);
 
//    主动调用回调函数开始工作
    pcmBufferCallBack(pcmBufferQueue, NULL);
```
> 注意：

在回调函数中result = (*pcmBufferQueue)->Enqueue(pcmBufferQueue, buffer, 44100 * 2 * 2)，最后的“44100*2*    2”是buffer的大小，因为我这里是指定了没读取一次就从pcm文件中读取了“44100*2* 2”个字节，所以可以正常播放，如果是利用ffmpeg来获取pcm数据源，那么实际大小要根据每个AVframe的具体大小来定，这样才能正常播放出声音！（44100 * 2 * 2 表示：44100是频率HZ，2是立体声双通道，2是采用的16位采样即2个字节，所以总的字节数就是：44100 * 2 * 2）





