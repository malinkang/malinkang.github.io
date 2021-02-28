---
title: Android Bitmap使用
date: 2016-06-24 13:52:35
draft: false
---

<!--toc-->


### Bitmap占用内存计算

Android的Bitmap对象提供了方法`getByteCount`来获取Bitmap的字节数。

```java
    public final int getByteCount() {
        // int result permits bitmaps up to 46,340 x 46,340
        return getRowBytes() * getHeight();
    }
```
`getRowBytes()`方法用来获取每行的字节数，内部调用一个native方法。
<!--more-->
```java
   public final int getRowBytes() {
        if (mRecycled) {
            Log.w(TAG, "Called getRowBytes() on a recycle()'d bitmap! This is undefined behavior!");
        }
        return nativeRowBytes(mFinalizer.mNativeBitmap);
    }
```
接下来我们继续通过查看[Bitmap.cpp](https://github.com/android/platform_frameworks_base/blob/master/core/jni/android/graphics/Bitmap.cpp)的源码可以找到`nativeRowBytes`方法的实现。

```java
static jint Bitmap_rowBytes(JNIEnv* env, jobject, jlong bitmapHandle) {
     SkBitmap* bitmap = reinterpret_cast<SkBitmap*>(bitmapHandle)
     return static_cast<jint>(bitmap->rowBytes());
}
```
这样我们就会发现Bitmap本质上是一个SkBitmap。


```java
size_t SkBitmap::ComputeRowBytes(Config c, int width) {
    return SkColorTypeMinRowBytes(SkBitmapConfigToColorType(c), width);
 }
```
[SkImageInfo.h](https://github.com/android/platform_external_skia/blob/ba7d93fad9b248bedd11999f509044575f7781f3/include/core/SkImageInfo.h)中`SkColorTypeBytesPerPixel `方法用来获取每像素的字节数。我们发现ARGB_8888（也就是我们最常用的 Bitmap 的格式）的一个像素占用 4byte，那么 rowBytes 实际上就是 4*width bytes。
```java
static int SkColorTypeBytesPerPixel(SkColorType ct) {
   static const uint8_t gSize[] = {
    0,  // Unknown
    1,  // Alpha_8
    2,  // RGB_565
    2,  // ARGB_4444
    4,  // RGBA_8888
    4,  // BGRA_8888
    1,  // kIndex_8
  };
  SK_COMPILE_ASSERT(SK_ARRAY_COUNT(gSize) == (size_t)(kLastEnum_SkColorType + 1),
                size_mismatch_with_SkColorType_enum);

   SkASSERT((size_t)ct < SK_ARRAY_COUNT(gSize));
   return gSize[ct];
}
//获取每行的byte数
static inline size_t SkColorTypeMinRowBytes(SkColorType ct, int width) {
    return width * SkColorTypeBytesPerPixel(ct);
}
```
那么一张图片640x824的图片在内存中占用的大小就是：
```
640x824x4=2109440
```
然而实际上我们把一张640x824图片放在`drawable-xhdpi`目录下运行在Nexus5手机上获得的大小为4746240差不多是我们上面计算出来的两倍。这是因为图片占用的尺寸会受存放位置和手机屏幕密度影响。

我们读取的是 drawable 目录下面的图片，用的是 decodeResource 方法。
```java
    public static Bitmap decodeResource(Resources res, int id, Options opts) {
        Bitmap bm = null;
        InputStream is = null; 
        
        try {
            final TypedValue value = new TypedValue();
            is = res.openRawResource(id, value);

            bm = decodeResourceStream(res, value, is, null, opts);
        } catch (Exception e) {
            /*  do nothing.
                If the exception happened on open, bm will be null.
                If it happened on close, bm is still valid.
            */
        } finally {
            try {
                if (is != null) is.close();
            } catch (IOException e) {
                // Ignore
            }
        }

        if (bm == null && opts != null && opts.inBitmap != null) {
            throw new IllegalArgumentException("Problem decoding into existing bitmap");
        }

        return bm;
    }
```
decodeResource内部调用 Resource.openRawResource 方法来读取图片资源，这个方法调用完成之后会对 TypedValue 进行赋值，其中包含了原始资源的 density 等信息。
```java
    public static Bitmap decodeResourceStream(Resources res, TypedValue value,
            InputStream is, Rect pad, Options opts) {

        if (opts == null) {
            opts = new Options();
        }

        if (opts.inDensity == 0 && value != null) {
            final int density = value.density;
            if (density == TypedValue.DENSITY_DEFAULT) {
                opts.inDensity = DisplayMetrics.DENSITY_DEFAULT;
            } else if (density != TypedValue.DENSITY_NONE) {
            //这里density的值如果对应资源目录为xhdpi的话，就是320
                opts.inDensity = density;
            }
        }
        
        if (opts.inTargetDensity == 0 && res != null) {
        //获取手机屏幕密度 Nexus5的屏幕密度为480
            opts.inTargetDensity = res.getDisplayMetrics().densityDpi;
        }
        return decodeStream(is, pad, opts);
    }
```

原始资源的 density 其实取决于资源存放的目录（比如 xhdpi 对应的是320）。这里还有一个目标密度即是屏幕密度。


紧接着，用到了一个native方法`nativeDecodeStream`。

```java
static jobject doDecode(JNIEnv* env, SkStreamRewindable* stream, jobject padding, jobject options) {

......
    if (env->GetBooleanField(options, gOptions_scaledFieldID)) {
        const int density = env->GetIntField(options, gOptions_densityFieldID);
        const int targetDensity = env->GetIntField(options, gOptions_targetDensityFieldID);
        const int screenDensity = env->GetIntField(options, gOptions_screenDensityFieldID);
        if (density != 0 && targetDensity != 0 && density != screenDensity) {
        //目标密度/原始资源密度 得到一个缩放比
            scale = (float) targetDensity / density;
        }
    }
}

const bool willScale = scale != 1.0f;
......
SkBitmap decodingBitmap;
if (!decoder->decode(stream, &decodingBitmap, prefColorType,decodeMode)) {
   return nullObjectReturn("decoder->decode returned false");
}
//这里这个deodingBitmap就是解码出来的bitmap，大小是图片原始的大小
int scaledWidth = decodingBitmap.width();
int scaledHeight = decodingBitmap.height();
if (willScale && decodeMode != SkImageDecoder::kDecodeBounds_Mode) {
//根据缩放比得到缩放后的宽和高
    scaledWidth = int(scaledWidth * scale + 0.5f);
    scaledHeight = int(scaledHeight * scale + 0.5f);
}
if (willScale) {
//与原始宽高比 计算一个缩放比
    const float sx = scaledWidth / float(decodingBitmap.width());
    const float sy = scaledHeight / float(decodingBitmap.height());

    // TODO: avoid copying when scaled size equals decodingBitmap size
    SkColorType colorType = colorTypeForScaledOutput(decodingBitmap.colorType());
    // FIXME: If the alphaType is kUnpremul and the image has alpha, the
    // colors may not be correct, since Skia does not yet support drawing
    // to/from unpremultiplied bitmaps.
    outputBitmap->setInfo(SkImageInfo::Make(scaledWidth, scaledHeight,
            colorType, decodingBitmap.alphaType()));
    if (!outputBitmap->allocPixels(outputAllocator, NULL)) {
        return nullObjectReturn("allocation failed for scaled bitmap");
    }

    // If outputBitmap's pixels are newly allocated by Java, there is no need
    // to erase to 0, since the pixels were initialized to 0.
    if (outputAllocator != &javaAllocator) {
        outputBitmap->eraseColor(0);
    }

    SkPaint paint;
    paint.setFilterLevel(SkPaint::kLow_FilterLevel);

    SkCanvas canvas(*outputBitmap);
    // Canvas 按照比例缩放
    canvas.scale(sx, sy);
    //绘制bitmap
    canvas.drawBitmap(decodingBitmap, 0.0f, 0.0f, &paint);
}
......
}
```
相应的注释已经写在代码里了，所以我们存放在xhdpi目录下的640x824的图片在Nexus5手机上占用的内存计算如下。
```java
scaledWidth = (int)(640x480/320+0.5)= 960;
scaleHeight = (int)(824x480/320+0.5)= 1236;
960x1236x4 = 4,746,240 
```
得到的值和我们通过代码获取到的值一致。

### 参考
* [高效显示Bitmap](http://hukai.me/android-training-course-in-chinese/graphics/displaying-bitmaps/index.html)
* [Android 开发绕不过的坑：你的 Bitmap 究竟占多大内存？](http://mp.weixin.qq.com/s?__biz=MzA3NTYzODYzMg==&mid=403263974&idx=1&sn=b0315addbc47f3c38e65d9c633a12cd6&scene=23&srcid=0624JmL3K1ZcheyD1vKrnPe4#rd)
* [Android Bitmap 优化(1) - 图片压缩](http://anany.me/2015/10/15/bitmap1/)
* [Android drawable微技巧，你所不知道的drawable的那些细节](http://blog.csdn.net/guolin_blog/article/details/50727753)
* [Reducing PNG file Size](https://medium.com/@duhroach/reducing-png-file-size-8473480d0476#.9u2detoc5)
* [Reducing PNG file Size译文](https://github.com/xitu/gold-miner/blob/master/TODO/reducing-jpg-file-size.md)
* [Android Bitmap面面观](http://jayfeng.com/2016/03/22/Android-Bitmap%E9%9D%A2%E9%9D%A2%E8%A7%82/)


