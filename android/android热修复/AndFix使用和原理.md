### 概述 （去查看官网）
- AndFix的基本介绍
- AndFix执行流程及核心原理
- 使用AndFix完成线上bug修复
- AndFix源码讲解

### AndFix的使用
1. 集成使用
   gradle集成
2. 编写AndFix管理类
```java

public class AndFixPathManager {
    private static AndFixPathManager mInstance = null;
    private PatchManager mPatchManager;

    public static AndFixPathManager getInstance() {
        if (mInstance == null) {
            synchronized (AndFixPathManager.class) {
                if (mInstance == null) {
                    mInstance = new AndFixPathManager();
                }
            }
        }
        return mInstance;
    }

    private AndFixPathManager() {

    }

    //初始化AndFix方法
    public void initPatch(Context context) {
        mPatchManager = new PatchManager(context);
        mPatchManager.init(AppUtils.getVersionName(context));
        mPatchManager.loadPatch();
    }

    /**
     * 加载我们的patch文件
     * @param path
     */
    public void addPatch(String path) {
        try {
            if (mPatchManager != null) {
                mPatchManager.addPatch(path);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```  

### 准备阶段


### 将AndFix组件化

### AndFix优劣
- 原理简单，集成简单，使用简单，即时生效
- 只能修复方法级别的bug，极大的限制了使用场景



