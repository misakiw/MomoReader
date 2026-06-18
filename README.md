# [English](English.md) [中文](README.md)

<div align="center">
<img width="125" height="125" src="https://github.com/misakiw/MomoReader/raw/master/app/src/main/res/mipmap-xxxhdpi/ic_launcher.png" alt="MomoReader"/>  
  
MomoReader
<br>
MomoRader is a free and open source novel reader for Android.
</div>

# API [![](https://img.shields.io/badge/-API-F5F5F5.svg)](#API-)
* 提供了2种方式的API：`Web方式`和`Content Provider方式`。您可以在[这里](api.md)根据需要自行调用。 
* 可通过url唤起阅读进行一键导入,url格式: MomoReader://import/{path}?src={url}
* path类型: bookSource,rssSource,replaceRule,textTocRule,httpTTS,theme,readConfig,dictRule,[addToBookshelf](/app/src/main/java/io/legado/app/ui/association/AddToBookshelfDialog.kt)
* path类型解释: 书源,订阅源,替换规则,本地txt小说目录规则,在线朗读引擎,主题,阅读排版,添加到书架


<a href="#readme">
    <img src="https://img.shields.io/badge/-返回顶部-orange.svg" alt="#" align="right">
</a>
