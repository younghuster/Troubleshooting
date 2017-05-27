# Android 7.0 低概率定屏和Camera2 ANR问题攻关

## 问题出现
- W16.49.4/W16.50.4版本，低概率出现定屏和Camera2 ANR问题，表现如下:

   - 复现概率低
   - adb能响应，除了system_server/Camera2其它进程均正常
   - system_server/Camera2不响应kill -3信号，无法dump java和native堆栈
   - system_server/Camera2的多数线程处于futex_wait状态
   
## 分析定位
### 初步分析
  
  定屏后，system_server进程的log很少: 只有少数几个线程打印log信息。
  - GC线程异常log
  
    分析多份log，发现GC线程输出异常信息
    ```
    A001-02 11:19:31.708   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:22:49.907   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:26:08.106   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:29:26.304   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:32:44.502   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:36:02.701   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:39:20.899   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:42:39.098   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:45:57.296   731   741 E art     : Unexpected time out during suspend all.
    A001-02 11:49:15.494   731   741 E art     : Unexpected time out during suspend all.
    ```
  - Debuggerd native堆栈
    ```
    "system_server" sysTid=731
     #00 pc 000173e4  /system/lib/libc.so (syscall+28)
     #01 pc 000b69bd  /system/lib/libart.so (_ZN3art17ConditionVariable16WaitHoldingLocksEPNS_6ThreadE+92)
     #02 pc 0027aca3  /system/lib/libart.so (_ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc+214)
     #03 pc 00091ee5  /system/lib/libandroid_runtime.so
     #04 pc 73104e99  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x2830000)

    "Signal Catcher" sysTid=736
     #00 pc 000173e4  /system/lib/libc.so (syscall+28)
     #01 pc 000b69bd  /system/lib/libart.so (_ZN3art17ConditionVariable16WaitHoldingLocksEPNS_6ThreadE+92)
     #02 pc 000f9d6d  /system/lib/libart.so (_ZN3art11ClassLinker14DumpForSigQuitERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEE+232)
     #03 pc 0031f9eb  /system/lib/libart.so (_ZN3art7Runtime14DumpForSigQuitERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEE+114)
     #04 pc 0032431b  /system/lib/libart.so (_ZN3art13SignalCatcher13HandleSigQuitEv+1394)
    #05 pc 00323489  /system/lib/libart.so (_ZN3art13SignalCatcher3RunEPv+336)
     #06 pc 00046fd3  /system/lib/libc.so (_ZL15__pthread_startPv+22)
     #07 pc 00019ded  /system/lib/libc.so (__start_thread+6)
    
    "HeapTaskDaemon" sysTid=741
     #00 pc 000173e4  /system/lib/libc.so (syscall+28)
     #01 pc 0033bcc5  /system/lib/libart.so (_ZN3art10ThreadList18SuspendAllInternalEPNS_6ThreadES2_S2_b+496)
     #02 pc 0033c187  /system/lib/libart.so (_ZN3art10ThreadList10SuspendAllEPKcb+374)
     #03 pc 0016eb45  /system/lib/libart.so (_ZN3art2gc9collector16GarbageCollector11ScopedPauseC2EPS2_+32)
     #04 pc 001737dd  /system/lib/libart.so (_ZN3art2gc9collector9MarkSweep9RunPhasesEv+160)
     #05 pc 0016e555  /system/lib/libart.so (_ZN3art2gc9collector16GarbageCollector3RunENS0_7GcCauseEb+248)
     #06 pc 00191a6d  /system/lib/libart.so   (_ZN3art2gc4Heap22CollectGarbageInternalENS0_9collector6GcTypeENS0_7GcCauseEb+2364)
     #07 pc 0019734d  /system/lib/libart.so (_ZN3art2gc4Heap12ConcurrentGCEPNS_6ThreadEb+68)
     #08 pc 0019be83  /system/lib/libart.so (_ZN3art2gc4Heap16ConcurrentGCTask3RunEPNS_6ThreadE+18)
     #09 pc 001b4243  /system/lib/libart.so (_ZN3art2gc13TaskProcessor11RunAllTasksEPNS_6ThreadE+30)
     #10 pc 72ce401f  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x2830000)
     ... ...
    ```
### 开会讨论      
  - [x] signal catcher为什么不能dump堆栈？

    - signal catcher线程已被GC线程suspend，除非被GC resume，否则无法响应kill -3信号。
  - [x] GC线程suspend机制梳理
    
    - GC线程先向各线程请求suspend，各线程会在检测到相应标志后主动suspend，GC线程等待其它线程suspend后才去做GC操作。
  - [x] w16.49.4回退提交排查
     
    - 后期宣告失败，因为早期版本也存在此问题。
  
### 添加debug log

  - 尝试分析debuggerd堆栈，排查未suspended线程。但system_server线程个数接近90，排查完后剩余的线程仍然很多。
  - 复现问题时，打印不能被suspend的线程及相关信息

### 问题复现和分析        
#### ART debug log

```
A001-01 09:30:03.427   722   733 E art     : Unexpected time out during suspend all.
A001-01 09:30:03.428   722   733 I art     : ThreadId = 18, tid = 761, name = android.display, state = Runnable

A001-01 09:49:14.140   722   733 E art     : Unexpected time out during suspend all.
A001-01 09:49:14.141   722   733 I art     : ThreadId = 18, tid = 761, name = android.display, state = Runnable

A001-01 10:08:24.853   722   733 E art     : Unexpected time out during suspend all.
A001-01 10:08:24.853   722   733 I art     : ThreadId = 18, tid = 761, name = android.display, state = Runnable
```
#### debuggerd堆栈分析
- system_server main线程
    - debuggerd native堆栈
        ```
        "system_server" sysTid=722
         #00 pc 000173e4  /system/lib/libc.so (syscall+28)
         #01 pc 000b69bd  /system/lib/libart.so (_ZN3art17ConditionVariable16WaitHoldingLocksEPNS_6ThreadE+92)
         #02 pc 0027aca3  /system/lib/libart.so (_ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc+214)
         #03 pc 00091ee5  /system/lib/libandroid_runtime.so
         #04 pc 732e1e99  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x2830000)
        ```
    - 查看对应的文件名和行号等信息
      
        #03$ **arm-linux-androideabi-addr2line -C -f -e   ~/symbols/system/lib/libandroid_runtime.so 00091ee5**
        
        ```
        android_content_AssetManager_getArrayStringResource
        frameworks/base/core/jni/android_util_AssetManager.cpp:1977
        ```
        
    - 相应的函数code
        ```c++
        1937static jobjectArray android_content_AssetManager_getArrayStringResource(JNIEnv* env, jobject clazz,
        1938                                                                        jint arrayResId)
        1939{
        1940    AssetManager* am = assetManagerForJavaObject(env, clazz);
        1941    if (am == NULL) {
        1942        return NULL;
        1943    }
        1944    const ResTable& res(am->getResources());
        1945
        1946    const ResTable::bag_entry* startOfBag;
        1947    const ssize_t N = res.lockBag(arrayResId, &startOfBag);
        1948    if (N < 0) {
        1949        return NULL;
        1950    }
        1951
        1952    jobjectArray array = env->NewObjectArray(N, g_stringClass, NULL);
        1953    if (env->ExceptionCheck()) {
        1954        res.unlockBag(startOfBag);
        1955        return NULL;
        1956    }
        1957
        1958    Res_value value;
        1959    const ResTable::bag_entry* bag = startOfBag;
        1960    size_t strLen = 0;
        1961    for (size_t i=0; ((ssize_t)i)<N; i++, bag++) {
        1962        value = bag->map.value;
        1963        jstring str = NULL;
        1964
        1965        // Take care of resolving the found resource to its final value.
        1966        ssize_t block = res.resolveReference(&value, bag->stringBlock, NULL);
        1967        if (kThrowOnBadId) {
        1968            if (block == BAD_INDEX) {
        1969                jniThrowException(env, "java/lang/IllegalStateException", "Bad resource!");
        1970                return array;
        1971            }
        1972        }
        1973        if (value.dataType == Res_value::TYPE_STRING) {
        1974            const ResStringPool* pool = res.getTableStringBlock(block);
        1975            const char* str8 = pool->string8At(value.data, &strLen);
        1976            if (str8 != NULL) {
        1977                str = env->NewStringUTF(str8);
        ```
- android.display线程
    - debuggerd native堆栈
        ```
        "android.display" sysTid=761
         #00 pc 000173e4  /system/lib/libc.so (syscall+28)
         #01 pc 000476a5  /system/lib/libc.so (_ZL33__pthread_mutex_lock_with_timeoutP24pthread_mutex_internal_tbPK8timespec+176)
         #02 pc 00093975  /system/lib/libandroid_runtime.so
         #03 pc 732e17e5  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x2830000)
        ```
    - 查看对应的文件名和行号等信息
    
        #02$ **arm-linux-androideabi-addr2line -C -f -e ~/symbols/system/lib/libandroid_runtime.so 00093975**
        
        ```
        android_content_AssetManager_applyStyle
        frameworks/base/core/jni/android_util_AssetManager.cpp:1430
        ```
    - 相应的函数code
        ```c++
        1340static jboolean android_content_AssetManager_applyStyle(JNIEnv* env, jobject clazz,
        1341                                                        jlong themeToken,
        1342                                                        jint defStyleAttr,
        1343                                                        jint defStyleRes,
        1344                                                        jlong xmlParserToken,
        1345                                                        jintArray attrs,
        1346                                                        jintArray outValues,
        1347                                                        jintArray outIndices)
        1348{
        1349    if (themeToken == 0) {
        1350        jniThrowNullPointerException(env, "theme token");
        1351        return JNI_FALSE;
        1352    }
        1353    if (attrs == NULL) {
        1354        jniThrowNullPointerException(env, "attrs");
        1355        return JNI_FALSE;
        1356    }
        1357    if (outValues == NULL) {
        1358        jniThrowNullPointerException(env, "out values");
        1359        return JNI_FALSE;
        1360    }
        1361
        1362    if (kDebugStyles) {
        1363    ALOGI("APPLY STYLE: theme=0x%" PRIx64 " defStyleAttr=0x%x defStyleRes=0x%x "
        1364          "xml=0x%" PRIx64, themeToken, defStyleAttr, defStyleRes,
        1365          xmlParserToken);
        1366    }
        1367
        1368    ResTable::Theme* theme = reinterpret_cast<ResTable::Theme*>(themeToken);
        1369    const ResTable& res = theme->getResTable();
        1370    ResXMLParser* xmlParser = reinterpret_cast<ResXMLParser*>(xmlParserToken);
        1371    ResTable_config config;
        1372    Res_value value;
        1373
        1374    const jsize NI = env->GetArrayLength(attrs);
        1375    const jsize NV = env->GetArrayLength(outValues);
        1376    if (NV < (NI*STYLE_NUM_ENTRIES)) {
        1377        jniThrowException(env, "java/lang/IndexOutOfBoundsException", "out values too small");
        1378        return JNI_FALSE;
        1379    }
        1380
        1381    jint* src = (jint*)env->GetPrimitiveArrayCritical(attrs, 0);
        1382    if (src == NULL) {
        1383        return JNI_FALSE;
        1384    }
        1385
        1386    jint* baseDest = (jint*)env->GetPrimitiveArrayCritical(outValues, 0);
        1387    jint* dest = baseDest;
        1388    if (dest == NULL) {
        1389        env->ReleasePrimitiveArrayCritical(attrs, src, 0);
        1390        return JNI_FALSE;
        1391    }
        1392
        1393    jint* indices = NULL;
        1394    int indicesIdx = 0;
        1395    if (outIndices != NULL) {
        1396        if (env->GetArrayLength(outIndices) > NI) {
        1397            indices = (jint*)env->GetPrimitiveArrayCritical(outIndices, 0);
        1398        }
        1399    }
        1400
        1401    // Load default style from attribute, if specified...
        1402    uint32_t defStyleBagTypeSetFlags = 0;
        1403    if (defStyleAttr != 0) {
        1404        Res_value value;
        1405        if (theme->getAttribute(defStyleAttr, &value, &defStyleBagTypeSetFlags) >= 0) {
        1406            if (value.dataType == Res_value::TYPE_REFERENCE) {
        1407                defStyleRes = value.data;
        1408            }
        1409        }
        1410    }
        1411
        1412    // Retrieve the style class associated with the current XML tag.
        1413    int style = 0;
        1414    uint32_t styleBagTypeSetFlags = 0;
        1415    if (xmlParser != NULL) {
        1416        ssize_t idx = xmlParser->indexOfStyle();
        1417        if (idx >= 0 && xmlParser->getAttributeValue(idx, &value) >= 0) {
        1418            if (value.dataType == value.TYPE_ATTRIBUTE) {
        1419                if (theme->getAttribute(value.data, &value, &styleBagTypeSetFlags) < 0) {
        1420                    value.dataType = Res_value::TYPE_NULL;
        1421                }
        1422            }
        1423            if (value.dataType == value.TYPE_REFERENCE) {
        1424                style = value.data;
        1425            }
        1426        }
        1427    }
        1428
        1429    // Now lock down the resource object and start pulling stuff from it.
        1430    res.lock();
        ```
- HeapTaskDaemon线程
  ```
      "HeapTaskDaemon" sysTid=733
         #00 pc 000173e4  /system/lib/libc.so (syscall+28)
         #01 pc 0033bcf1  /system/lib/libart.so (_ZN3art10ThreadList18SuspendAllInternalEPNS_6ThreadES2_S2_b+540)
         #02 pc 0033c2f7  /system/lib/libart.so (_ZN3art10ThreadList10SuspendAllEPKcb+374)
         #03 pc 0016eb45  /system/lib/libart.so (_ZN3art2gc9collector16GarbageCollector11ScopedPauseC2EPS2_+32)
         #04 pc 001737dd  /system/lib/libart.so (_ZN3art2gc9collector9MarkSweep9RunPhasesEv+160)
         #05 pc 0016e555  /system/lib/libart.so (_ZN3art2gc9collector16GarbageCollector3RunENS0_7GcCauseEb+248)
         #06 pc 00191a6d  /system/lib/libart.so (_ZN3art2gc4Heap22CollectGarbageInternalENS0_9collector6GcTypeENS0_7GcCauseEb+2364)
         #07 pc 0019734d  /system/lib/libart.so (_ZN3art2gc4Heap12ConcurrentGCEPNS_6ThreadEb+68)
         #08 pc 0019be83  /system/lib/libart.so (_ZN3art2gc4Heap16ConcurrentGCTask3RunEPNS_6ThreadE+18)
         #09 pc 001b4243  /system/lib/libart.so (_ZN3art2gc13TaskProcessor11RunAllTasksEPNS_6ThreadE+30)
         #10 pc 72ec101f  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x2830000)
    ```
    
#### 死锁现象总结
  - GC线程向各线程发送suspend request，并等待各线程suspend
  - main线程进入到JNI，拿到某模块一个mutex，并已suspend
  - android.display线程调用到JNI， 等待main线程释放mutex

    ![fast JNI deadlock 1](https://raw.githubusercontent.com/younghuster/Troubleshooting/master/assets/fastjni_deadlock1.png)

### 最新复现

#### 新的堆栈
- system_server main线程
  - debuggerd native堆栈
    ```
    "system_server" sysTid=720
        #00 pc 000173e4  /system/lib/libc.so (syscall+28)
        #01 pc 000b62bd  /system/lib/libart.so (_ZN3art17ConditionVariable16WaitHoldingLocksEPNS_6ThreadE+92)
        #02 pc 00279a03  /system/lib/libart.so (_ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc+214)
        #03 pc 00091ee5  /system/lib/libandroid_runtime.so
        #04 pc 72b51e99  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x2830000)
    ```

  - 查看对应的文件名和行号等信息
        
    #03$ **arm-linux-androideabi-addr2line -C -f -e ~/symbols/system/lib/libandroid_runtime.so 00091ee5**   

    ```
    android_content_AssetManager_getArrayStringResource
    frameworks/base/core/jni/android_util_AssetManager.cpp:1977
    ```
    
- Binder线程
    - debuggerd native堆栈
      ```
        "Binder:720_5" sysTid=944
        #00 pc 000173e4  /system/lib/libc.so (syscall+28)
        #01 pc 000476a5  /system/lib/libc.so (_ZL33__pthread_mutex_lock_with_timeoutP24pthread_mutex_internal_tbPK8timespec+176)
        #02 pc 0001cb49  /system/lib/libandroidfw.so (_ZN7android8ResTable13setParametersEPKNS_15ResTable_configE+24)
        #03 pc 00013993  /system/lib/libandroidfw.so (_ZNK7android12AssetManager26updateResourceParamsLockedEv+86)
        #04 pc 000138f9  /system/lib/libandroidfw.so (_ZN7android12AssetManager15setLocaleLockedEPKc+240)
        #05 pc 00013add  /system/lib/libandroidfw.so (_ZN7android12AssetManager16setConfigurationERKNS_15ResTable_configEPKc+60)
        #06 pc 00092b4f  /system/lib/libandroid_runtime.so
        #07 pc 72b5537d  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x2830000)
      ```
    
    - 查看对应的文件名和行号等信息
    
        #03$ **arm-linux-androideabi-addr2line -C -f -e ~/symbols/system/lib/libandroidfw.so 00013993**
        ```
        android::AssetManager::updateResourceParamsLocked() const
        /proc/self/cwd/frameworks/base/libs/androidfw/AssetManager.cpp:744
        ```
        #04$ **arm-linux-androideabi-addr2line -C -f -e ~/symbols/system/lib/libandroidfw.so 000138f9**
        ```
        android::AssetManager::setLocaleLocked(char const*)
        /proc/self/cwd/frameworks/base/libs/androidfw/AssetManager.cpp:414
        ```
        #05$ **arm-linux-androideabi-addr2line -C -f -e ~/symbols/system/lib/libandroidfw.so 00013add**
        ```  
        android::AssetManager::setConfiguration(android::ResTable_config const&, char const*)
        /proc/self/cwd/frameworks/base/libs/androidfw/AssetManager.cpp:441
        ```
        #06$ **arm-linux-androideabi-addr2line -C -f -e ~/symbols/system/lib/libandroid_runtime.so 00092b4f**
        ```  
        android_content_AssetManager_setConfiguration
        frameworks/base/core/jni/android_util_AssetManager.cpp:721
        ```
- HeapTaskDaemon线程
  ```
  "HeapTaskDaemon" sysTid=731
     #00 pc 000173e4  /system/lib/libc.so (syscall+28)
     #01 pc 0033aa59  /system/lib/libart.so (_ZN3art10ThreadList18SuspendAllInternalEPNS_6ThreadES2_S2_b+516)
     #02 pc 0033b13f  /system/lib/libart.so (_ZN3art10ThreadList10SuspendAllEPKcb+374)
     #03 pc 0016e445  /system/lib/libart.so (_ZN3art2gc9collector16GarbageCollector11ScopedPauseC2EPS2_+32)
     #04 pc 001730dd  /system/lib/libart.so (_ZN3art2gc9collector9MarkSweep9RunPhasesEv+160)
     #05 pc 0016de55  /system/lib/libart.so (_ZN3art2gc9collector16GarbageCollector3RunENS0_7GcCauseEb+248)
     #06 pc 0019136d  /system/lib/libart.so (_ZN3art2gc4Heap22CollectGarbageInternalENS0_9collector6GcTypeENS0_7GcCauseEb+2364)
     #07 pc 00196c4d  /system/lib/libart.so (_ZN3art2gc4Heap12ConcurrentGCEPNS_6ThreadEb+68)
     #08 pc 0019b783  /system/lib/libart.so (_ZN3art2gc4Heap16ConcurrentGCTask3RunEPNS_6ThreadE+18)
     #09 pc 001b39e3  /system/lib/libart.so (_ZN3art2gc13TaskProcessor11RunAllTasksEPNS_6ThreadE+30)
     #10 pc 7273101f  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x2830000)
  ```
#### 新的死锁 
  - 死锁如下   
  
    ![fast JNI deadlock 2](https://raw.githubusercontent.com/younghuster/Troubleshooting/master/assets/fastjni_deadlock2.png)
     
### 分析JNI
  - 前面多次复现的JNI都跟frameworks/base下面的asset manager里面的JNI函数有关。
  
  - 对比了Android 6.0 vs. Android 7.0关于gAssetManagerMethods[]的差异，7.0比6.0在部分JNI函数的signature上增加了"!", 表示是fast JNI, 目的是提高性能。
    ```
    commit 46cfd93aa22b5a4acab4626c140415eef922445c
    Author: Jeff Sharkey <jsharkey@android.com>
    Date:   Mon Nov 2 18:39:45 2015 -0800

    Let's sprinkle some FastJNI into Resources.
    
    Before:
    
                   benchmark   us linear runtime
                    GetColor 14.9 ===========
                  GetInteger 19.9 ===============
        GetLayoutAndTraverse 38.4 =============================
                   GetString 38.5 ==============================
    
    After:
    
                   benchmark   us linear runtime
                    GetColor 13.9 ==========
                  GetInteger 18.8 ==============
        GetLayoutAndTraverse 38.1 =============================
                   GetString 38.2 ==============================
    
    Change-Id: I8c20e14182d2645bc62a0e7fc6345e298b11933c

    diff --git a/core/jni/android_util_AssetManager.cpp b/core/jni/android_util_AssetManager.cpp
    index c94bc64..55b7e7e 100644
    --- a/core/jni/android_util_AssetManager.cpp
    +++ b/core/jni/android_util_AssetManager.cpp
    @@ -2156,9 +2156,9 @@ static const JNINativeMethod gAssetManagerMethods[] = {
    ... ....
    -    { "applyStyle","(JIIJ[I[I[I)Z",
    +    { "applyStyle","!(JIIJ[I[I[I)Z",
         (void*) android_content_AssetManager_applyStyle },
    -    { "setConfiguration", "(IILjava/lang/String;IIIIIIIIIIIIII)V",
    +    { "setConfiguration", "!(IILjava/lang/String;IIIIIIIIIIIIII)V",
         (void*) android_content_AssetManager_setConfiguration },
    ... ...
    ```
  - 但ART在处理fast JNI时，不会进行状态切换
    ```c++
    extern uint32_t JniMethodStart(Thread* self) {
      JNIEnvExt* env = self->GetJniEnv();
      DCHECK(env != nullptr);
      uint32_t saved_local_ref_cookie = env->local_ref_cookie;
      env->local_ref_cookie = env->locals.GetSegmentState();
      ArtMethod* native_method = *self->GetManagedStack()->GetTopQuickFrame();
      if (!native_method->IsFastNative()) {
        // When not fast JNI we transition out of runnable.
        self->TransitionFromRunnableToSuspended(kNative);
      }
      return saved_local_ref_cookie;
    }
    ```
    
---

# 解决方案
- Revert fast JNI optimization
   - 验证
     - 基于出问题的W16.50.4版本验证两轮monkey测试： 20台机器运行48小时没有复现
     -  合入patch后, 类似的定屏和Camera2 ANR问题不再出现．
     
- AOSP

  Remove FastJNI optimization on AssetManager to avoid dead lock.
  
  https://android-review.googlesource.com/#/c/340143/
