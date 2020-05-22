**问题1**【Android NDK】cannot initialize a parameter of type 'jboolean *' (aka 'unsigned char *') with an rva

产生问题的原因是由于由老版本升级到新版本导致，新版的jni规范更加严格，需要使用内置的bool属性常量，改成JNI_FALSE之后这个错误消失了。

以前可以这样写
```
const char *jstringTocharArray(JNIEnv *env, jstring str) {
 return env->GetStringChars(str,false);//这里的bool类型可以直接写
    
}
```

**新版写法**

```
char16 *fixed_ptr = (char16*)(*env).GetStringChars(fixed_str, JNI_FALSE); //必须使用JNI内置的值

```

**另一种解决办法**

在对应项目的jni/Application.mk添加一句话<br>

APP_CFLAGS += -Wno-error=format-security