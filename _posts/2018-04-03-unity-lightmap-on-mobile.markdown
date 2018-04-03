---
layout:           post
title:            PC上烘焙的lightmap在移动设备上变暗的原因和解决方案
data:             2018-04-03
tags:
    - Unity
---

# 原因

Unity烘焙的lightmap以openEXR格式存储的HDR图，在该贴图导入时会转变为RGBM格式，因为Lightmap在Unity导入选项中的TextureImportSettiongs.rgbm默认为auto。RGBM把[0,8]范围的值压缩成[0,1]范围，并且把一个系数存储在Alpha通道里，最终的颜色值为RGB A 8。而Unity解析lightmap的shader源码如下：

~~~
// Decodes HDR textures
// handles dLDR, RGBM formats
// Called by DecodeLightmap when UNITY_NO_RGBM is not defined.
inline half3 DecodeLightmapRGBM (half4 data, half4 decodeInstructions)
{
    // If Linear mode is not supported we can skip exponent part
    #if defined(UNITY_COLORSPACE_GAMMA)
    # if defined(UNITY_FORCE_LINEAR_READ_FOR_RGBM)
        return (decodeInstructions.x * data.a) * sqrt(data.rgb);
    # else
        return (decodeInstructions.x * data.a) * data.rgb;
    # endif
    #else
        return (decodeInstructions.x * pow(data.a, decodeInstructions.y)) * data.rgb;
    #endif
}

// Decodes doubleLDR encoded lightmaps.
inline half3 DecodeLightmapDoubleLDR( fixed4 color )
{
    return 2.0 * color.rgb;
}

inline half3 DecodeLightmap( fixed4 color, half4 decodeInstructions)
{
#if defined(UNITY_NO_RGBM)
    return DecodeLightmapDoubleLDR( color );
#else
    return DecodeLightmapRGBM( color, decodeInstructions );
#endif
}

half4 unity_Lightmap_HDR;

inline half3 DecodeLightmap( fixed4 color )
{
    return DecodeLightmap( color, unity_Lightmap_HDR );
}
~~~
在移动设备上，通常不支持HDR图片，经过Unity的宏控制，最终代码走向下面这段：
~~~
// Decodes doubleLDR encoded lightmaps.
inline half3 DecodeLightmapDoubleLDR( fixed4 color )
{
    return 2.0 * color.rgb;
}
~~~
在线性空间下，按照RGBM算法，RGB A8材质最终的值和上面得出的值会相差4倍。由于线性空间的亮度是线性增长的，这样的差距会非常明显，这里也就是产生色差的原因。
# 解决方案
出现色差的原因就在这段代码
~~~
// Decodes doubleLDR encoded lightmaps.
inline half3 DecodeLightmapDoubleLDR( fixed4 color )
{
    return 2.0 * color.rgb;
}
~~~
为什么这里会直接乘以2，可能是因为这里的处理忽略了线性空间，只按照gamma空间来处理了。那解决方案就是增加针对线性空间的处理。修改后的代码如下：
~~~
inline half3 DecodeLightmapDoubleLDR( fixed4 color )
{
#if defined(UNITY_COLORSPACE_GAMMA)
    return 2.0 * color.rgb;
#else
    return unity_Lightmap_HDR.x * color.rgb;
#endif
}
~~~
这样的修改会丢失掉一部分lightmap的细节。但在对比iOS设备和PC效果美术同学说和看上去完全一样！。

#一些反思
* 我们是不准备开发PC版本的，但是美术一直工作在Unity的PC环境下，这样会增加一些踩到unity跨平台渲染的坑的可能性。大家一起工作在安卓环境下，可能是更好的选择。
* 现在的处理方式，应该不是正确的，但是它看起来是正确的，那么它就是正确的。（很讨厌图形学这一点啊）

#参考网址
* https://zhuanlan.zhihu.com/p/28728151
* http://gad.qq.com/article/detail/29196



