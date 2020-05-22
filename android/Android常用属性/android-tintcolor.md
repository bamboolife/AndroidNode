```xml
android:tint="@color/main_color"
android:tintMode="multiply"
```

```
<!--src_in 内部填充-->
<!--src_atop 内部填充-->
<!--src_over  方形-->
<!--screen  外部填充-->
<!--add 外部填充-->
<!--multiply 正常填充-->
```

1. PorterDuff.Mode.CLEAR 所绘制不会提交到画布上。
2. PorterDuff.Mode.SRC 显示上层绘制图片
3. PorterDuff.Mode.DST 显示下层绘制图片
4. PorterDuff.Mode.SRC_OVER 正常绘制显示，上下层绘制叠盖。
5. PorterDuff.Mode.DST_OVER 上下层都显示。下层居上显示。
6. PorterDuff.Mode.SRC_IN 取两层绘制交集。显示上层。
7. PorterDuff.Mode.DST_IN 取两层绘制交集。显示下层。
8. PorterDuff.Mode.SRC_OUT 取上层绘制非交集部分。
9. PorterDuff.Mode.DST_OUT 取下层绘制非交集部分。
10. PorterDuff.Mode.SRC_ATOP 取下层非交集部分与上层交集部分
11. PorterDuff.Mode.DST_ATOP 取上层非交集部分与下层交集部分
12. PorterDuff.Mode.XOR 取两层绘制非交集。两层绘制非交集。
13. PorterDuff.Mode.DARKEN 上下层都显示。变暗
14. PorterDuff.Mode.LIGHTEN 上下层都显示。变亮
15. PorterDuff.Mode.MULTIPLY 取两层绘制交集
16. PorterDuff.Mode.SCREEN 上下层都显示。
