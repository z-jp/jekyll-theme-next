---
layout: post
title: Android 4.x 中的SwitchPreference
category: Android
tags: Android
---

终于为[TranslateIt](http://www.coolapk.com/apk/com.zjp.translateit)添加了设置页，实机测试没发现问题就提交了新版本，不料没多久就有反馈设置开关完全错乱。在4.X的系统中，点击“退出前确认”可能是前面两项被打开，也可能是后面两项被关闭，而且毫无规律。虚拟机上运行结果也是如此，看来是我代码有问题咯。然而各种代码检查，各种谷歌搜索，无果。无聊之下，把所有SwitchPreference换成CheckBoxPreFerence  ——两者在我看来只有外观上的区别 ——然后，错位的问题解决了！

问题定位在4.X上的SwichPreference 终于谷歌到一个google code上的一个[issus](https://code.google.com/p/android/issues/detail?id=26194)：

>Found in Android4.0
The bug is that state of SwitchPreference will be changed sometime when it is scrolled out of the screen.
After investigate and debug the source code, we found the reason.
This bug happens when a SwitchPreference(use SP_A for short) is scrolled out of the screen and at the same time another SwitchPreference(use SP_B for short) is scrolled in the screen. The SP_B will use the view of SP_A as the parameter of method onBindView. The onBindView method please refer to the attachment.

不过在我这，页面不处于滚动状态时，点击也时错位的。不管怎样，这SwitchPreference是没法正常使用了，解决方法在issus里也贴出来了

````
public class CustomSwitchPreference extends SwitchPreference {

    /**
     * Construct a new SwitchPreference with the given style options.
     *
     * @param context The Context that will style this preference
     * @param attrs Style attributes that differ from the default
     * @param defStyle Theme attribute defining the default style options
     */
    public CustomSwitchPreference(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
    }

    /**
     * Construct a new SwitchPreference with the given style options.
     *
     * @param context The Context that will style this preference
     * @param attrs Style attributes that differ from the default
     */
    public CustomSwitchPreference(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    /**
     * Construct a new SwitchPreference with default style options.
     *
     * @param context The Context that will style this preference
     */
    public CustomSwitchPreference(Context context) {
        super(context, null);
    }

    @Override
    protected void onBindView(View view) {
        // Clean listener before invoke SwitchPreference.onBindView
        ViewGroup viewGroup= (ViewGroup)view;
        clearListenerInViewGroup(viewGroup);
        super.onBindView(view);
    }

    /**
     * Clear listener in Switch for specify ViewGroup.
     *
     * @param viewGroup The ViewGroup that will need to clear the listener.
     */
    private void clearListenerInViewGroup(ViewGroup viewGroup) {
        if (null == viewGroup) {
            return;
        }

        int count = viewGroup.getChildCount();
        for(int n = 0; n &lt; count; ++n) {
            View childView = viewGroup.getChildAt(n);
            if(childView instanceof Switch) {
                final Switch switchView = (Switch) childView;
                switchView.setOnCheckedChangeListener(null);
                return;
            } else if (childView instanceof ViewGroup){
                ViewGroup childGroup = (ViewGroup)childView;
                clearListenerInViewGroup(childGroup);
            }
        }
    }

}
````

看了下提交时间 2012年！本人，卒。
