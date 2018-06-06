---
title: (大坑) apk push进system/app/之后.so库无法加载
date: 2018-01-20 14:11:27
tags:
---

---

当我开心的以为我的程序成了系统app再也无忧的时候，果然又出现了一个bug，.so的库找不到了，于是我去/data/data/底下的app安装目录里寻找，果然唯独少了so文件，这是什么情况&#63;  
<!--more-->

如果不知道怎么将apk变成系统apk，请看《系统Lanucher app开发》

苦思冥想开始分析原因&#58;

当我们app push到system/app目录下成为系统级app时，app默认加载so库的路径变成了system/lib，而system/lib却是只读的，导致so库无法正常解压  
知道了原因，于是就开始解决方案&#58;  

#### 1.将所有的so库提取出来放到system/lib目录下  
于是开始实验  
同样用上面的mount命令获取system的读写权限，再将so库依次push进system/lib 重启手机，发现程序运行成功了，可是发现了这样做的缺点，不便于调试，因为我们项目有很多自己的.so库，所以就需要嵌入式只要给我们so
库，我们就得替换一个新的进去，如果这样放在system/lib的目录，每次都得push最新的进去，确实麻烦，我们的嵌入式大神也规劝我不要这么干，这样会污染系统本身自带的so库，后续可能会有一些不必要的麻烦  

于是又开始寻找新方案解决 
#### 2.在程序最开始运行如下Java代码  
思路就是复制apk中的.so库到/data/data/app的安装目录中，再通过反射更改app加载so库的路径，将nativeLibraryDir这个属性获取并修改成我们解压so库的目标路径，所有加载so库的方法都放在这个类执行之后开始，这个时候重新打包运行，发现在/data/data/apk的安装目录里，之前没有的
so库出现了，程序也正常运行  
最后总结一下优缺点&#58;优点是便于调试，不用更改一个so库就得push一下，可以动态加载  
缺点是 路子有点野，属于奇淫巧技之类，可能会留下坑

``` java
public class StatSoFiles {

    private static final String TAG = StatSoFiles.class.getSimpleName();
    private static final String APK_REINSTALL = "apk_reinstall";
    private Context mContext;
    private List<String> mApkSoFiles = new ArrayList<>();
    private String APK_PATH;
    private final boolean mIs64Bit = false;
    private final String mApkLibPath;

    public StatSoFiles(Context context) {
        mContext = context;
        APK_PATH = mContext.getApplicationInfo().sourceDir;
//        mIs64Bit = MiscUtils.is64Bit();
        //初始化要释放so文件的路径
        mApkLibPath = "lib/armeabi-v7a";
    }

    //释放so文件
    public boolean verifyAndReleaseLibSo() {
        initSoFileName();
        for (int i = 0; i < mApkSoFiles.size(); i++) {
            String soFile = mApkSoFiles.get(i);
            boolean inAppLib = isInAppLib(soFile);
//            if (inAppLib) {
//            } else {
                //将so文件写到应用的包路径下
                writeSoToDir(soFile);
//            }
        }
        boolean testLoadLibrary = testLoadLibrary();
        return testLoadLibrary;
    }

    public boolean testLoadLibrary() {
        //检测so文件释放能成功加载
        try {
            System.loadLibrary("cvc");
            return true;
        } catch (Throwable e) {
            return false;
        }
    }

    private void initSoFileName() {
        //从apk中获得所有so文件名
        InputStream in = null;
        ZipInputStream zin = null;
        try {
            in = new BufferedInputStream(new FileInputStream(APK_PATH));
            zin = new ZipInputStream(in);
            ZipEntry ze;
            while ((ze = zin.getNextEntry()) != null) {
                String soName = ze.getName();
                if (soName != null && soName.startsWith(mApkLibPath)) {
                    mApkSoFiles.add(soName.replace(mApkLibPath, ""));
                }
            }
        } catch (Exception e) {
        } finally {
            if (zin != null) {
                try {
                    zin.close();
                } catch (Exception e) {
                }
            }
            if (in != null) {
                try {
                    in.close();
                } catch (Exception e) {
                }
            }
        }
    }

    private void writeSoToDir(String soName) {
        if (TextUtils.isEmpty(APK_PATH)) {
            return;
        }
        ZipFile zf = null;
        BufferedOutputStream bof = null;
        BufferedInputStream fb = null;
        try {
            zf = new ZipFile(APK_PATH);
            ZipEntry ze = zf.getEntry(mApkLibPath + soName);
            fb = new BufferedInputStream(zf.getInputStream(ze));
            File libs = mContext.getDir("libs", Context.MODE_PRIVATE);
            if (!libs.exists()) {
                libs.mkdirs();
            }
            File file = new File(libs, soName);
            bof = new BufferedOutputStream(new FileOutputStream(file));
            byte[] buff = new byte[4096];
            int len = 0;
            while ((len = fb.read(buff)) != -1) {
                bof.write(buff, 0, len);
            }
        } catch (IOException e) {
        } finally {
            closeStream(zf, bof, fb);
        }
    }

    private void closeStream(ZipFile zf, BufferedOutputStream bof, BufferedInputStream fb) {
        try {
            if (bof != null) {
                bof.close();
            }
        } catch (IOException e) {
        }
        try {
            if (fb != null) {
                fb.close();
            }
        } catch (IOException e) {
        }
        try {
            if (zf != null) {
                zf.close();
            }
        } catch (IOException e) {
        }
    }

    private boolean isInAppLib(String soName) {
        File dir = mContext.getDir("libs", Context.MODE_PRIVATE);
        String appLibPath = dir.getAbsolutePath() + "/" + soName;
        File appLibSoFile = new File(appLibPath);
        if (appLibSoFile.exists()) {
            long soSize = appLibSoFile.length();
            ZipFile zf = null;
            try {
                zf = new ZipFile(APK_PATH);
                ZipEntry ze = zf.getEntry(mApkLibPath + soName);
                long size = ze.getSize();
                if (size == soSize) {
                    return true;
                } else {
                    appLibSoFile.delete();
                    return false;
                }
            } catch (IOException e) {
            } finally {
                if (zf != null) {
                    try {
                        zf.close();
                    } catch (IOException e) {
                    }
                }
            }
        }
        return false;
    }

    //修改加载so的路径
    public void initNativeDirectory(Context context) {
        if (hasDexClassLoader()) { // 4.0以上系统
            try {
//                SLog.d(TAG, "AboveEqualApiLevel14");
                createNewNativeDirAboveEqualApiLevel14(context);
            } catch (Exception e) {
                try {
                    //6.0系统
                    createNewNativeDirAboveEqualApiLevel21(context);
                } catch (Exception e1) {
                    e1.printStackTrace();
                }
            }
        } else {
            try {
                createNewNativeDirBelowApiLevel14(context); // 4.0以下系统
            } catch (Exception e) {
            }
        }
    }

    private void createNewNativeDirAboveEqualApiLevel21(Context context)
            throws IllegalAccessException, NoSuchFieldException, ClassNotFoundException {
        if (Build.VERSION.SDK_INT > Build.VERSION_CODES.LOLLIPOP) {
            PathClassLoader pathClassLoader = (PathClassLoader) context.getClassLoader();
            Object pathList = getPathList(pathClassLoader);
            Class<?> elementClass = Class.forName("dalvik.system.DexPathList$Element");
            Constructor<?> element = null;
            try {
                element = elementClass.getConstructor(File.class, boolean.class, File.class, DexFile.class);
            } catch (NoSuchMethodException e) {
            }
            Object systemNativeLibraryDirectories = pathList.getClass()
                    .getDeclaredField("systemNativeLibraryDirectories");
            Object nativeLibraryDirectories = pathList.getClass().getDeclaredField("nativeLibraryDirectories");
            Object nativeLibraryPathElements = pathList.getClass().getDeclaredField("nativeLibraryPathElements");
            ((Field) systemNativeLibraryDirectories).setAccessible(true);
            ((Field) nativeLibraryDirectories).setAccessible(true);
            ((Field) nativeLibraryPathElements).setAccessible(true);
            // 获取 DEXPATHList中的属性值
            List<File> systemFiles = (List<File>) ((Field) systemNativeLibraryDirectories).get(pathList);
            List<File> nativeFiles = (List<File>) ((Field) nativeLibraryDirectories).get(pathList);
            Object[] elementFiles = (Object[]) ((Field) nativeLibraryPathElements).get(pathList);
            Object newElementFiles = Array.newInstance(elementClass, elementFiles.length + 1);
            systemFiles.add(context.getDir("libs", Context.MODE_PRIVATE));
            nativeFiles.add(context.getDir("libs", Context.MODE_PRIVATE));
            ((Field) systemNativeLibraryDirectories).set(pathList, systemFiles);
            ((Field) nativeLibraryDirectories).set(pathList, nativeFiles);
            if (element != null) {
                try {
                    Object newInstance = element.newInstance(
                            new File(context.getDir("libs", Context.MODE_PRIVATE).getAbsolutePath()), true, null, null);
                    Array.set(newElementFiles, 0, newInstance);
                    for (int i = 1; i < elementFiles.length + 1; i++) {
                        Array.set(newElementFiles, i, elementFiles[i - 1]);
                    }
                    ((Field) nativeLibraryPathElements).set(pathList, newElementFiles);
                } catch (InstantiationException | InvocationTargetException e) {
                }
            }
        }
    }

    private void createNewNativeDirBelowApiLevel14(Context context)
            throws NoSuchFieldException, IllegalAccessException {
        PathClassLoader pathClassLoader = (PathClassLoader) context.getClassLoader();
        Field mLibPaths = pathClassLoader.getClass().getDeclaredField("mLibPaths");
        mLibPaths.setAccessible(true);
        String[] libs = (String[]) (mLibPaths).get(pathClassLoader);
        Object newPaths = Array.newInstance(String.class, libs.length + 1);
        // 添加自定义.so路径
        Array.set(newPaths, 0, context.getDir("libs", Context.MODE_PRIVATE).getAbsolutePath());
        // 将系统自己的追加上
        for (int i = 1; i < libs.length + 1; i++) {
            Array.set(newPaths, i, libs[i - 1]);
        }
        mLibPaths.set(pathClassLoader, newPaths);
    }

    private void createNewNativeDirAboveEqualApiLevel14(Context context) throws Exception {
        PathClassLoader pathClassLoader = (PathClassLoader) context.getClassLoader();
        Object pathList = getPathList(pathClassLoader);
        // 获取当前类的属性
        Object nativeLibraryDirectories = pathList.getClass().getDeclaredField("nativeLibraryDirectories");
        ((Field) nativeLibraryDirectories).setAccessible(true);
        // 获取 DEXPATHList中的属性值
        File[] files = (File[]) ((Field) nativeLibraryDirectories).get(pathList);
        Object newfiles = Array.newInstance(File.class, files.length + 1);
        // 添加自定义.so路径
        Array.set(newfiles, 0, context.getDir("libs", Context.MODE_PRIVATE));
        // 将系统自己的追加上
        for (int i = 1; i < files.length + 1; i++) {
            Array.set(newfiles, i, files[i - 1]);
        }
        ((Field) nativeLibraryDirectories).set(pathList, newfiles);
    }

    private Object getPathList(Object obj) throws ClassNotFoundException, NoSuchFieldException, IllegalAccessException {
        return getField(obj, Class.forName("dalvik.system.BaseDexClassLoader"), "pathList");
    }

    private Object getField(Object obj, Class cls, String str) throws NoSuchFieldException, IllegalAccessException {
        Field declaredField = cls.getDeclaredField(str);
        declaredField.setAccessible(true);
        return declaredField.get(obj);
    }

    private boolean hasDexClassLoader() {
        try {
            Class.forName("dalvik.system.BaseDexClassLoader");
            return true;
        } catch (ClassNotFoundException var1) {
            return false;
        }
    }

}
```
