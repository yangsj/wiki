转自：https://www.jianshu.com/p/0efcf4ad5963

Android刘海屏适配
1. 前言
在我们进行APP开发时，屏幕适配是一件很头疼的事，而Android又相对iOS的较为混杂，各种各样厂商和屏幕类型，有全面屏、刘海屏、水滴屏、钻孔屏、折叠屏、挖孔屏等等，所以在适配的过程中需要考虑不少东西。以下记录一下我在适配过程中遇到的厂商和解决方案。(如有更好的意见可以一起探讨 ^ _ ^ )

在适配之前需要先了解 刘海屏的开关设置 和 以下三个类的代码内容。

刘海屏除了正常的开关之外，当开启刘海屏时还有一个自定义显示刘海区域的选项，在这里面也可以控制刘海的显示，所以有些设备并不是开着刘海获取到的刘海状态就是开着的。

BTDeviceFather抽象类，每个厂商适配类都需要继承这个父类。根据每个厂商的特色重写需要实现的方法。
```
import android.graphics.Point;

abstract class BTDeviceFather {

    /**
     * 刘海宽度
     * 
     * @return
     */
    public int getNotchWidth() {
        return 0;
    }

    /**
     * 刘海高度
     * 
     * @return
     */
    public int getNotchHeigth() {
        return 0;
    }

    /**
     * 屏幕底部危险高度
     * 
     * @return
     */
    public int getBottomDangerHeigth() {
        return 0;
    }

    /**
     * 是否隐藏刘海
     * 
     * @return
     */
    public boolean isHideNotch() {
        return false;
    }

    /**
     * 是否支持刘海
     * 
     * @return
     */
    public boolean isSupportNotch() {
        return false;
    }

    /**
     * 获取设备(Physical Size)真实分辨率
     * 
     * @return
     */
    public Point getResolution() {
        return DeviceTools.getScreenRealSize();
    }

}
```
DeviceTools工具类，含有各种获取状态栏高度、导航条高度、屏幕宽高等通用方法。
```
import java.lang.reflect.Method;

import android.app.Activity;
import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.Point;
import android.os.Build;
import android.util.DisplayMetrics;
import android.view.Display;
import android.view.View;
import android.view.WindowManager;

public class DeviceTools {

    /**
     * 是否支持全面屏系统 Android N+
     * 
     * @return
     */
    public static boolean isSupportAndroidVersion() {
        return Build.VERSION.SDK_INT >= Build.VERSION_CODES.N;
    }

    /**
     * 获取屏幕真实分辨率(除vivo系统)
     * 
     * @return
     */
    @SuppressWarnings("deprecation")
    public static Point getScreenRealSize() {
        Point screenSize = null;
        try {
            screenSize = new Point();
            WindowManager windowManager = (WindowManager) BTDeviceSDK.getInstance().getActivity().getSystemService(Context.WINDOW_SERVICE);
            Display defaultDisplay = windowManager.getDefaultDisplay();
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
                defaultDisplay.getRealSize(screenSize);
            } else {
                try {
                    Method mGetRawW = Display.class.getMethod("getRawWidth");
                    Method mGetRawH = Display.class.getMethod("getRawHeight");
                    screenSize.set((Integer) mGetRawW.invoke(defaultDisplay), (Integer) mGetRawH.invoke(defaultDisplay));
                } catch (Exception e) {
                    screenSize.set(defaultDisplay.getWidth(), defaultDisplay.getHeight());
                    e.printStackTrace();
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        return screenSize;
    }

    /**
     * 获取显示分辨率
     * 
     * @param activity
     * @return
     */
    public static Point getScreenDisplaySize(Activity activity) {
        DisplayMetrics dm = activity.getResources().getDisplayMetrics();
        Point screenSize = new Point();
        screenSize.set(dm.widthPixels, dm.heightPixels);
        return screenSize;
    }

    /**
     * 获取截图分辨率
     */
    @SuppressWarnings("deprecation")
    public static Point getScreenShotSize() {
        View view = BTDeviceSDK.getInstance().getActivity().getWindow().getDecorView();
        view.setDrawingCacheEnabled(true);
        view.buildDrawingCache();
        Bitmap bmp = view.getDrawingCache();
        Point screenSize = new Point();
        if (bmp != null) {
            screenSize.set(bmp.getWidth(), bmp.getHeight());
        }
        return screenSize;
    }

    /**
     * 获取状态栏高度
     * 
     * @return
     */
    public static int getStatusBarHeight(Activity activity) {
        int result = 0;
        int resourceId = activity.getResources().getIdentifier("status_bar_height", "dimen", "android");
        if (resourceId > 0) {
            result = activity.getResources().getDimensionPixelSize(resourceId);
        }
        return result;
    }

    /**
     * 获取导航栏高度
     * 
     * @return
     */
    public static int getNavigationBarHeight(Activity activity) {
        int resourceId = 0;
        int rid = activity.getResources().getIdentifier("config_showNavigationBar", "bool", "android");
        if (rid != 0) {
            resourceId = activity.getResources().getIdentifier("navigation_bar_height", "dimen", "android");
            return activity.getResources().getDimensionPixelSize(resourceId);
        }
        return 0;
    }

}
```
BTDisplayCutout类，获取Android9.0+屏幕安全区域大小方法。
```
import java.lang.reflect.Method;

import android.annotation.SuppressLint;
import android.util.Log;
import android.view.View;
import android.view.WindowInsets;

public class BTDisplayCutout {

    private static View mView;

    public static void init(View view) {
        BTDisplayCutout.mView = view;
    }

    @SuppressLint("NewApi")
    private static Object getDisplayCutout(View view) {
        if (view != null) {
            Log.d("DisplayCutOutTools", "Build.VERSION.SDK_INT:" + android.os.Build.VERSION.SDK_INT);
            if (android.os.Build.VERSION.SDK_INT >= android.os.Build.VERSION_CODES.O) {
                try {
                    WindowInsets windowInsets = view.getRootWindowInsets();
                    if (windowInsets != null) {
                        Method method = windowInsets.getClass().getDeclaredMethod("getDisplayCutout");
                        return method.invoke(windowInsets);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        Log.d("DisplayCutOutTools", "getDisplayCutout is null.");
        return null;
    }

    /**
     * 获取顶部安全区域高度
     *
     * @return
     */
    public static int getSafeInsetTop() {
        try {
            Object displayCutoutInstance = getDisplayCutout(BTDisplayCutout.mView);
            if (displayCutoutInstance != null) {
                Class<?> cls = displayCutoutInstance.getClass();
                return (Integer) cls.getDeclaredMethod("getSafeInsetTop").invoke(displayCutoutInstance);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return 0;
    }

    /**
     * 获取底部安全区域高度
     *
     * @return
     */
    public static int getSafeInsetBottom() {
        try {
            Object displayCutoutInstance = getDisplayCutout(BTDisplayCutout.mView);
            if (displayCutoutInstance != null) {
                Class<?> cls = displayCutoutInstance.getClass();
                return (Integer) cls.getDeclaredMethod("getSafeInsetBottom").invoke(displayCutoutInstance);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return 0;
    }

    /**
     * 获取左部安全区域高度
     *
     * @return
     */
    public static int getSafeInsetLeft() {
        try {
            Object displayCutoutInstance = getDisplayCutout(BTDisplayCutout.mView);
            if (displayCutoutInstance != null) {
                Class<?> cls = displayCutoutInstance.getClass();
                return (Integer) cls.getDeclaredMethod("getSafeInsetLeft").invoke(displayCutoutInstance);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return 0;
    }

    /**
     * 获取右部安全区域高度
     *
     * @return
     */
    public static int getSafeInsetRight() {
        try {
            Object displayCutoutInstance = getDisplayCutout(BTDisplayCutout.mView);
            if (displayCutoutInstance != null) {
                Class<?> cls = displayCutoutInstance.getClass();
                return (Integer) cls.getDeclaredMethod("getSafeInsetRight").invoke(displayCutoutInstance);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return 0;
    }

}
```
2. 华为屏幕适配
官方文档: https://developer.huawei.com/consumer/cn/devservice/doc/50114
```
import java.lang.reflect.Method;

import android.graphics.Point;
import android.provider.Settings;
import android.util.Log;

/**
 * 
 * 需要再AndroidManifest.xml中配置
 * 
 * <meta-data android:name="android.notch_support" android:value="true"/>
 * 
 * @author Cylee
 *
 */
class BTHuawei extends BTDeviceFather {

    private String TAG = "BTHuawei";

    @Override
    public boolean isSupportNotch() {
        try {
            ClassLoader cl = BTDeviceSDK.getInstance().getActivity().getClassLoader();
            Class<?> HwNotchSizeUtil = cl.loadClass("com.huawei.android.util.HwNotchSizeUtil");
            Method get = HwNotchSizeUtil.getMethod("hasNotchInScreen");
            return (Boolean) get.invoke(HwNotchSizeUtil);
        } catch (ClassNotFoundException e) {
            Log.e(TAG, "isFeatureSupport ClassNotFoundException");
        } catch (NoSuchMethodException e) {
            Log.e(TAG, "isFeatureSupport NoSuchMethodException");
        } catch (Exception e) {
            Log.e(TAG, "isFeatureSupport Exception");
        }
        return false;
    }

    @Override
    public boolean isHideNotch() {
        boolean isHide = Settings.Secure.getInt(BTDeviceSDK.getInstance().getActivity().getContentResolver(), "display_notch_status", 0) == 1;
        if (!isHide) {
            Point pReal = BTDeviceSDK.getInstance().getResolution();
            Point pShot = DeviceTools.getScreenShotSize();
            int realSize = pReal.x > pReal.y ? pReal.x : pReal.y;
            int shotSize = pShot.x > pShot.y ? pShot.x : pShot.y;
            int statusBarHeigth = BTDeviceSDK.getInstance().getStatusBarHeight();
            // 设备分辨率-截图分辨率=状态栏，则代表隐藏刘海
            int off = realSize - shotSize;
            if (off == statusBarHeigth || off == statusBarHeigth + 1 || off == statusBarHeigth - 1) {
                return true;
            }
            return false;
        }
        return true;
    }

    @Override
    public int getNotchWidth() {
        // 如果不具备特性或者隐藏了刘海，则返回0
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        return getNotchSize()[0];
    }

    @Override
    public int getNotchHeigth() {
        // 如果不具备特性或者隐藏了刘海，则返回0
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        return getNotchSize()[1];
    }

    /**
     * 获取刘海屏凹槽尺寸
     * 
     * @param context
     * @return
     */
    private int[] getNotchSize() {
        int[] ret = new int[] { 0, 0 };
        try {
            ClassLoader cl = BTDeviceSDK.getInstance().getActivity().getClassLoader();
            Class<?> HwNotchSizeUtil = cl.loadClass("com.huawei.android.util.HwNotchSizeUtil");
            Method get = HwNotchSizeUtil.getMethod("getNotchSize");
            ret = (int[]) get.invoke(HwNotchSizeUtil);
        } catch (ClassNotFoundException e) {
            Log.e(TAG, "getNotcSize ClassNotFoundException");
        } catch (NoSuchMethodException e) {
            Log.e(TAG, "getNotcSize NoSuchMethodException");
        } catch (Exception e) {
            Log.e(TAG, "getNotcSize Exception");
        }
        return ret;
    }
}
```
3. 小米屏幕适配
官方文档: https://dev.mi.com/console/doc/detail?pId=1293
```
import java.lang.reflect.Method;

import android.annotation.SuppressLint;
import android.os.Build;
import android.os.Handler;
import android.os.Message;
import android.provider.Settings;
import android.view.Window;

/**
 * 在AndroidManifest.xml中配置
 * 
 * <meta-data android:name="notch.config" android:value="portrait|landscape"/>
 * 横竖屏都绘制到耳朵区
 * 
 * <meta-data android:name="notch.config" android:value="none"/> 不绘制到耳朵区
 * 
 * @author Cylee
 *
 */
class BTXiaomi extends BTDeviceFather {

    @Override
    public boolean isSupportNotch() {
        try {
            Class<?> mClassType = Class.forName("android.os.SystemProperties");
            Method mGetIntMethod = mClassType.getDeclaredMethod("getInt", String.class, int.class);
            Integer v = (Integer) mGetIntMethod.invoke(mClassType, "ro.miui.notch", 0);
            return v.intValue() == 1;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    @Override
    public boolean isHideNotch() {
        return Settings.Global.getInt(BTDeviceSDK.getInstance().getActivity().getContentResolver(), "force_black", 0) == 1;
    }

    @Override
    public int getNotchWidth() {
        // 如果不具备特性，则返回0
        if (!isSupportNotch()) {
            return 0;
        }

        // 如果隐藏了刘海，则设置不使用耳朵区
        if (isHideNotch()) {
            Message msg = new Message();
            msg.obj = "clearExtraFlags";
            mHandler.sendMessage(msg);
            return 0;
        }

        // 如果显示刘海，则设置使用耳朵区
        Message msg = new Message();
        msg.obj = "addExtraFlags";
        mHandler.sendMessage(msg);

        String model = Build.MODEL;
        if (model.contains("MI8Lite")) {
            return 296;
        }
        if (model.contains("Redmi 6 Pro")) {
            return 352;
        }
        if (model.contains("MI 8 SE")) {
            return 540;
        }
        if (model.contains("MI 8") || model.contains("MI 8 Explorer Edition") || model.contains("MI 8 UD")) {
            return 560;
        }
        if (model.contains("POCO F1")) {
            return 588;
        }

        int result = 0;
        int resourceId = BTDeviceSDK.getInstance().getActivity().getResources().getIdentifier("notch_width", "dimen", "android");
        if (resourceId > 0) {
            result = BTDeviceSDK.getInstance().getActivity().getResources().getDimensionPixelSize(resourceId);
        }
        return result;
    }

    @Override
    public int getNotchHeigth() {
        // 如果不具备特性，则返回0
        if (!isSupportNotch()) {
            return 0;
        }

        // 如果隐藏了刘海，则设置不使用耳朵区
        if (isHideNotch()) {
            Message msg = new Message();
            msg.obj = "clearExtraFlags";
            mHandler.sendMessage(msg);
            return 0;
        }

        // 如果显示刘海，则设置使用耳朵区
        Message msg = new Message();
        msg.obj = "addExtraFlags";
        mHandler.sendMessage(msg);

        String model = Build.MODEL;
        if (model.contains("MI8Lite")) {
            return 82;
        }
        if (model.contains("MI 8 SE")) {
            return 85;
        }
        if (model.contains("POCO F1")) {
            return 86;
        }
        if (model.contains("MI 8") || model.contains("MI 8 Explorer Edition") || model.contains("MI 8 UD") || model.contains("Redmi 6 Pro")) {
            return 89;
        }
        if (model.contains("MI 9")) {
            return 89;
        }

        int result = 0;
        int resourceId = BTDeviceSDK.getInstance().getActivity().getResources().getIdentifier("notch_height", "dimen", "android");
        if (resourceId > 0) {
            result = BTDeviceSDK.getInstance().getActivity().getResources().getDimensionPixelSize(resourceId);
        }
        return result;
    }

    @SuppressLint("HandlerLeak")
    private Handler mHandler = new Handler() {

        @Override
        public void dispatchMessage(Message msg) {
            String cmd = (String) msg.obj;
            try {
                // 此方法需要在主线程上调用，否则会崩溃。
                Method method = Window.class.getMethod(cmd, int.class);
                method.invoke(BTDeviceSDK.getInstance().getActivity().getWindow(), 0x00000100 | 0x00000200 | 0x00000400);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

};
```
4. VIVO屏幕适配
官方文档: https://dev.vivo.com.cn/documentCenter/doc/103
```
import java.lang.reflect.Method;

import android.graphics.Point;
import android.util.Log;

/**
 * 在AndroidManifest.xml中配置
 * 
 * <meta-data android:name="android.vendor.home_indicator"android:value="hide"/>
 * 隐藏导航栏
 * 
 * <meta-data android:name="android.vendor.full_screen" android:value="true" />
 * 全屏显示
 * 
 * 在对应的Activity中添加下面的meta用于隐藏导航条 <meta-data
 * android:name="android.vendor.home_indicator" android:value="hide" />
 * 
 * @author Cylee
 *
 */
class BTVivo extends BTDeviceFather {

    private String TAG = "BTVivo";

    @Override
    public Point getResolution() {
        int statusBarHeight = BTDeviceSDK.getInstance().getStatusBarHeight();
        int navigationBarHeight = BTDeviceSDK.getInstance().getNavigationBarHeight();
        Point pReal = DeviceTools.getScreenRealSize();
        Point pShot = DeviceTools.getScreenShotSize();
        int realSize = pReal.x > pReal.y ? pReal.x : pReal.y;
        int shotSize = pShot.x > pShot.y ? pShot.x : pShot.y;
        // 由于vivo系统的问题，如果获取的真实分辨率-截图分辨率=状态栏+导航条的高度，则使用显示分辨率
        if (realSize - shotSize == statusBarHeight + navigationBarHeight) {
            return DeviceTools.getScreenDisplaySize(BTDeviceSDK.getInstance().getActivity());
        }
        return pReal;
    }

    @Override
    public boolean isSupportNotch() {
        try {
            Class<?> mClass = Class.forName("android.util.FtFeature");
            Method[] methods = mClass.getDeclaredMethods();
            Method method = null;
            for (Method m : methods) {
                if (m.getName().equalsIgnoreCase("isFeatureSupport")) {
                    method = m;
                    break;
                }
            }
            // 0x00000020表示是否有凹槽
            // 0x00000008表示是否有圆角
            return (Boolean) method.invoke(null, 0x00000020);
        } catch (ClassNotFoundException e) {
            Log.e(TAG, "getFeature ClassNotFoundException");
        } catch (Exception e) {
            Log.e(TAG, "getFeature Exception");
        }
        return false;
    }

    @Override
    public boolean isHideNotch() {
        Point pReal = BTDeviceSDK.getInstance().getResolution();
        Point pShot = DeviceTools.getScreenShotSize();
        int realSize = pReal.x > pReal.y ? pReal.x : pReal.y;
        int shotSize = pShot.x > pShot.y ? pShot.x : pShot.y;
        int statusBarHeigth = BTDeviceSDK.getInstance().getStatusBarHeight();
        // 设备分辨率-截图分辨率=状态栏，则代表隐藏刘海
        int off = realSize - shotSize;
        if (off == statusBarHeigth || off == statusBarHeigth + 1 || off == statusBarHeigth - 1) {
            return true;
        }
        // 如果是横屏游戏
        if (BTDeviceSDK.getInstance().getOrientation() == Configuration.ORIENTATION_LANDSCAPE) {
            // 如果设备是IQOO，设备分辨率-截图分辨率=73，则代表隐藏刘海
            String model = Build.MODEL;
            if (model.contains("V1824") && realSize - shotSize == 73) {
                return true;
            }
        }
        return false;
    }

    @Override
    public int getNotchWidth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        int statusBarHeight = BTDeviceSDK.getInstance().getStatusBarHeight();
        return statusBarHeight * 100 / 32;
    }

    @Override
    public int getNotchHeigth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        int statusBarHeight = BTDeviceSDK.getInstance().getStatusBarHeight();
        return statusBarHeight * 27 / 32;
    }

    @Override
    public int getBottomDangerHeigth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        int statusBarHeight = BTDeviceSDK.getInstance().getStatusBarHeight();
        return statusBarHeight * 24 / 32;
    };

}
```
5. OPPO屏幕适配
官方文档: https://open.oppomobile.com/wiki/doc#id=10159
```
import android.content.pm.PackageManager;
import android.graphics.Point;

class BTOppo extends BTDeviceFather {

    @Override
    public boolean isSupportNotch() {
        PackageManager pm = BTDeviceSDK.getInstance().getActivity().getPackageManager();
        return pm.hasSystemFeature("com.oppo.feature.screen.heteromorphism");
    }

    @Override
    public boolean isHideNotch() {
        Point pReal = BTDeviceSDK.getInstance().getResolution();
        Point pShot = DeviceTools.getScreenShotSize();
        int realSize = pReal.x > pReal.y ? pReal.x : pReal.y;
        int shotSize = pShot.x > pShot.y ? pShot.x : pShot.y;
        int statusBarHeigth = BTDeviceSDK.getInstance().getStatusBarHeight();
        // 设备分辨率-截图分辨率=状态栏，则代表隐藏刘海
        int off = realSize - shotSize;
        if (off == statusBarHeigth || off == statusBarHeigth + 1 || off == statusBarHeigth - 1) {
            return true;
        }
        return false;
    }

    @Override
    public int getNotchWidth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        return 324;
    }

    @Override
    public int getNotchHeigth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        return 80;
    }

}
```
6. OnePlus屏幕适配
官方文档: 貌似没有官方API文档
```
import android.graphics.Point;
import android.os.Build;

class BTOnePlus extends BTDeviceFather {

    @Override
    public boolean isSupportNotch() {
        String model = Build.MODEL;
        if (model.contains("ONEPLUS A6000") || model.contains("ONEPLUS A6010") || model.contains("GM1900")) {
            return true;
        }
        return false;
    }

    @Override
    public boolean isHideNotch() {
        Point pReal = BTDeviceSDK.getInstance().getResolution();
        Point pShot = DeviceTools.getScreenShotSize();
        int realSize = pReal.x > pReal.y ? pReal.x : pReal.y;
        int shotSize = pShot.x > pShot.y ? pShot.x : pShot.y;
        int statusBarHeigth = BTDeviceSDK.getInstance().getStatusBarHeight();
        // 设备分辨率-截图分辨率=状态栏，则代表隐藏刘海
        int off = realSize - shotSize;
        if (off == statusBarHeigth || off == statusBarHeigth + 1 || off == statusBarHeigth - 1) {
            return true;
        }
        return false;
    }

    @Override
    public int getNotchHeigth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        return BTDeviceSDK.getInstance().getStatusBarHeight();
    }

}
```
7. 联想 (摩托罗拉) 屏幕适配
官方文档: http://open.lenovo.com/sdk/%E5%85%A8%E9%9D%A2%E5%B1%8F%E9%80%82%E9%85%8D%E6%8C%87%E5%8D%97/
```
import android.graphics.Point;

class BTLenovo extends BTDeviceFather {

    @Override
    public boolean isSupportNotch() {
        boolean result = false;
        int resourceId = BTDeviceSDK.getInstance().getActivity().getResources().getIdentifier("config_screen_has_notch", "bool", "android");
        if (resourceId > 0) {
            result = BTDeviceSDK.getInstance().getActivity().getResources().getBoolean(resourceId);
        }
        return result;
    }

    @Override
    public boolean isHideNotch() {
        Point pReal = BTDeviceSDK.getInstance().getResolution();
        Point pShot = DeviceTools.getScreenShotSize();
        int realSize = pReal.x > pReal.y ? pReal.x : pReal.y;
        int shotSize = pShot.x > pShot.y ? pShot.x : pShot.y;
        int statusBarHeigth = BTDeviceSDK.getInstance().getStatusBarHeight();
        // 设备分辨率-截图分辨率=状态栏，则代表隐藏刘海
        int off = realSize - shotSize;
        if (off == statusBarHeigth || off == statusBarHeigth + 1 || off == statusBarHeigth - 1) {
            return true;
        }
        return false;
    }

    @Override
    public int getNotchHeigth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }

        int result = 0;
        int resourceId = BTDeviceSDK.getInstance().getActivity().getResources().getIdentifier("notch_h", "integer", "android");
        if (resourceId > 0) {
            result = BTDeviceSDK.getInstance().getActivity().getResources().getInteger(resourceId);
        }
        return result;
    }

    @Override
    public int getNotchWidth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }

        int result = 0;
        int resourceId = BTDeviceSDK.getInstance().getActivity().getResources().getIdentifier("notch_w", "integer", "android");
        if (resourceId > 0) {
            result = BTDeviceSDK.getInstance().getActivity().getResources().getInteger(resourceId);
        }
        return result;
    }

}
```
8.魅族屏幕适配
```
import java.lang.reflect.Field;

import android.provider.Settings;
import android.util.Log;

import com.platform.device.feature.BTDeviceFather;

public class BTMeizu extends BTDeviceFather {

    private String TAG = "BTMeizu";

    @Override
    public boolean isSupportNotch() {
        boolean fringeDevice = false;
        try {
            Class<?> clazz = Class.forName("flyme.config.FlymeFeature");
            Field field = clazz.getDeclaredField("IS_FRINGE_DEVICE");
            fringeDevice = (Boolean) field.get(null);
        } catch (Exception e) {
            Log.e(TAG, "isSupportNotch:\n" + e.toString());
        }
        return fringeDevice;
    }

    @Override
    public boolean isHideNotch() {
        // 判断隐藏刘海开关(默认关)
        return Settings.Global.getInt(BTDeviceSDK.getInstance().getActivity().getContentResolver(), "mz_fringe_hide", 0) == 1;
    }

    @Override
    public int getNotchHeigth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }

        // 获取刘海高度（51px）
        int fringeHeight = 0;
        int fhid = BTDeviceSDK.getInstance().getActivity().getResources().getIdentifier("fringe_height", "dimen", "android");
        if (fhid > 0) {
            fringeHeight = BTDeviceSDK.getInstance().getActivity().getResources().getDimensionPixelSize(fhid);
        }
        return fringeHeight;
    }

}
```
9.努比亚屏幕适配
```
import android.graphics.Point;
import android.os.Build;

class BTNubia extends BTDeviceFather {

    @Override
    public boolean isSupportNotch() {
        String model = Build.MODEL;
        if (model.contains("NX606J")) {
            return true;
        }
        return false;
    }

    @Override
    public boolean isHideNotch() {
        Point pReal = BTDeviceSDK.getInstance().getResolution();
        Point pShot = DeviceTools.getScreenShotSize();
        int realSize = pReal.x > pReal.y ? pReal.x : pReal.y;
        int shotSize = pShot.x > pShot.y ? pShot.x : pShot.y;
        int statusBarHeigth = BTDeviceSDK.getInstance().getStatusBarHeight();
        // 设备分辨率-截图分辨率=状态栏，则代表隐藏刘海
        int off = realSize - shotSize;
        if (off == statusBarHeigth || off == statusBarHeigth + 1 || off == statusBarHeigth - 1) {
            return true;
        }
        return false;
    }

    @Override
    public int getNotchHeigth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }
        return BTDeviceSDK.getInstance().getStatusBarHeight();
    }

}
```
10.三星屏幕适配
官方文档: http://support-cn.samsung.com/App/DeveloperChina/home/list?parentid=11&newsid=72
```
package com.platform.device.feature;

import android.content.res.Resources;
import android.graphics.Point;
import android.text.TextUtils;
import android.util.Log;

class BTSamsung extends BTDeviceFather {

    private String TAG = "BTSamsung";

    @Override
    public boolean isSupportNotch() {
        try {
            final Resources res = BTDeviceSDK.getInstance().getActivity().getResources();
            final int resId = res.getIdentifier("config_mainBuiltInDisplayCutout", "string", "android");
            final String spec = resId > 0 ? res.getString(resId) : null;
            return spec != null && !TextUtils.isEmpty(spec);
        } catch (Exception e) {
            Log.e(TAG, "getFeature Exception");
        }
        return false;
    }

    @Override
    public boolean isHideNotch() {
        int orientation = BTDeviceSDK.getInstance().getOrientation();
        if (orientation == Configuration.ORIENTATION_PORTRAIT) {
            int top = BTDisplayCutout.getSafeInsetTop();
            int bottom = BTDisplayCutout.getSafeInsetBottom();
            return top > 0 || bottom > 0 ? false : true;
        }

        int left = BTDisplayCutout.getSafeInsetLeft();
        int right = BTDisplayCutout.getSafeInsetRight();
        return left > 0 || right > 0 ? false : true;
    }

    @Override
    public int getNotchHeigth() {
        if (!isSupportNotch() || isHideNotch()) {
            return 0;
        }

        int orientation = BTDeviceSDK.getInstance().getOrientation();
        if (orientation == Configuration.ORIENTATION_PORTRAIT) {
            int top = BTDisplayCutout.getSafeInsetTop();
            int bottom = BTDisplayCutout.getSafeInsetBottom();
            return top > bottom ? top : bottom;
        }

        int left = BTDisplayCutout.getSafeInsetLeft();
        int right = BTDisplayCutout.getSafeInsetRight();
        return left > right ? left : right;
    }

}
```
11. 设置沉浸式布局
设置沉浸式布局，可在一定程度上避免获取的屏幕宽高错误的问题。
比如修改了刘海开关、导航条显示方式(三键模式、手势模式等)，都会影响获取的屏幕宽高大小。
```
@SuppressLint("InlinedApi")
private void setSystemUiVisibility() {
    int uiOptions = View.SYSTEM_UI_FLAG_LAYOUT_STABLE | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
            | View.SYSTEM_UI_FLAG_FULLSCREEN;
    if (Build.VERSION.SDK_INT >= 19) {
        uiOptions |= View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;
    }
    mActivity.getWindow().getDecorView().setSystemUiVisibility(uiOptions);
}
```
12. 解决Android9.0+不全屏问题
在安卓9.0系统的小米刘海屏手机中，当用户打开刘海开关时，但耳朵区并不进行绘制。
解决此问题在生命周期onCreate中调用以下方法即可。
```
@SuppressLint("InlinedApi")
private void initDisplayCutoutMode() {
    if ((Build.VERSION.SDK_INT >= 28)) {
        WindowManager.LayoutParams lp = mActivity.getWindow().getAttributes();
        lp.layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
        mActivity.getWindow().setAttributes(lp);
        mActivity.getWindow().setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN, WindowManager.LayoutParams.FLAG_FULLSCREEN);
        mActivity.getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
    }
}
```
13. 动态屏幕适配
当用户在使用APP的过程中，动态修改了刘海屏的显示或隐藏，那么我们就要在runtime下进行及时调整避让区域等，以解决UI显示异常等现象。
```
private void initObserver() {
    ViewTreeObserver vto = this.mActivity.getWindow().getDecorView().getViewTreeObserver();
    vto.addOnGlobalLayoutListener(new OnGlobalLayoutListener() {

    @Override
    public void onGlobalLayout() {
        // 这里处理新视图的逻辑
    });
}
```
