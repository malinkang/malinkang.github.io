---
title: 创建 Android Studio Template
date: 2016-06-15 10:40:23
draft: false
---

AndroidStudio 为我们提供了很多创建Activity以及其他类和文件夹的Template。

![](/images/android-studio-template-1.png)

<!--more-->

AndroidStudio虽然提供了大多常见的的模板，但是不可能满足每个人的需求，因此自定义模板就显得很有必要。下面我们将分析系统提供的模板，并自己来实现模板。

系统提供的Template存放在`/Applications/Android Studio.app/Contents/plugins/android/lib/templates`目录下，activities目录下是Actiivty模板。


下面我们以EmptyActivity为例进行分析

template.xml 代码如下：

```xml

<?xml version="1.0"?>
<template
    format="5"
    revision="5"
    name="Empty Activity"
    minApi="7"
    minBuildApi="14"
    description="Creates a new empty activity">

    <category value="Activity" />
    <formfactor value="Mobile" />

    <parameter
        id="activityClass"
        name="Activity Name"
        type="string"
        constraints="class|unique|nonempty"
        suggest="${layoutToActivity(layoutName)}"
        default="MainActivity"
        help="The name of the activity class to create" />

    <parameter
        id="generateLayout"
        name="Generate Layout File"
        type="boolean"
        default="true"
        help="If true, a layout file will be generated" />

    <parameter
        id="layoutName"
        name="Layout Name"
        type="string"
        constraints="layout|unique|nonempty"
        suggest="${activityToLayout(activityClass)}"
        default="activity_main"
        visibility="generateLayout"
        help="The name of the layout to create for the activity" />

    <parameter
        id="isLauncher"
        name="Launcher Activity"
        type="boolean"
        default="false"
        help="If true, this activity will have a CATEGORY_LAUNCHER intent filter, making it visible in the launcher" />

    <parameter
        id="backwardsCompatibility"
        name="Backwards Compatibility (AppCompat)"
        type="boolean"
        default="true"
        help="If false, this activity base class will be Activity instead of AppCompatActivity" />
    
    <parameter
        id="packageName"
        name="Package name"
        type="string"
        constraints="package"
        default="com.mycompany.myapp" />

    <!-- 128x128 thumbnails relative to template.xml -->
    <thumbs>
        <!-- default thumbnail is required -->
        <thumb>template_blank_activity.png</thumb>
    </thumbs>

    <globals file="globals.xml.ftl" />
    <execute file="recipe.xml.ftl" />

</template>

```
这里的`<parameter>`标签代表创建的时候输入的参数。

globals.xml.ftl中主要定义全局常量。
```xml
<?xml version="1.0"?>
<globals>
    <global id="hasNoActionBar" type="boolean" value="false" />
    <global id="parentActivityClass" value="" />
    <global id="simpleLayoutName" value="${layoutName}" />
    <global id="excludeMenu" type="boolean" value="true" />
    <global id="generateActivityTitle" type="boolean" value="false" />
    <#include "../common/common_globals.xml.ftl" />
</globals>
```

recipe.xml.ftl代码如下。
```xml
<?xml version="1.0"?>
<recipe>
    <#include "../common/recipe_manifest.xml.ftl" />
<#if generateLayout>
    <#include "../common/recipe_simple.xml.ftl" />
    <open file="${escapeXmlAttribute(resOut)}/layout/${layoutName}.xml" />
</#if>
    <instantiate from="root/src/app_package/SimpleActivity.java.ftl"
                   to="${escapeXmlAttribute(srcOut)}/${activityClass}.java" />

    <open file="${escapeXmlAttribute(srcOut)}/${activityClass}.java" />
</recipe>
```






### 参考
* [Android Studio自定义模板 写页面竟然可以如此轻松](http://blog.csdn.net/lmj623565791/article/details/51635533)
* [How to create a group of File Templates in Android Studio – Part 3](https://riggaroo.co.za/custom-file-template-group-android-studiointellij/)
