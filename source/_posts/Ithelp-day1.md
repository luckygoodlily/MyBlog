---
title: (開賽)Http 請求 Asp.net IIS伺服器架構 (第1天)
date: 2019-09-12 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---

# Agenda<!-- omit in toc -->
- [開賽前言：](#%e9%96%8b%e8%b3%bd%e5%89%8d%e8%a8%80)
  - [為什麼想要選擇此主題](#%e7%82%ba%e4%bb%80%e9%ba%bc%e6%83%b3%e8%a6%81%e9%81%b8%e6%93%87%e6%ad%a4%e4%b8%bb%e9%a1%8c)
  - [閱讀時建議事項](#%e9%96%b1%e8%ae%80%e6%99%82%e5%bb%ba%e8%ad%b0%e4%ba%8b%e9%a0%85)
  - [文章收穫](#%e6%96%87%e7%ab%a0%e6%94%b6%e7%a9%ab)
- [進入主題](#%e9%80%b2%e5%85%a5%e4%b8%bb%e9%a1%8c)
  - [瀏覽器請求IIS流程](#%e7%80%8f%e8%a6%bd%e5%99%a8%e8%ab%8b%e6%b1%82iis%e6%b5%81%e7%a8%8b)
  - [如何辨別是否為靜態檔案?](#%e5%a6%82%e4%bd%95%e8%be%a8%e5%88%a5%e6%98%af%e5%90%a6%e7%82%ba%e9%9d%9c%e6%85%8b%e6%aa%94%e6%a1%88)
  - [.Net CLR Httpmodule & Httphandler 核心模組](#net-clr-httpmodule--httphandler-%e6%a0%b8%e5%bf%83%e6%a8%a1%e7%b5%84)
  - [W3WP應用程式](#w3wp%e6%87%89%e7%94%a8%e7%a8%8b%e5%bc%8f)
- [小結](#%e5%b0%8f%e7%b5%90)

## 開賽前言：

三十篇文章架構基本遵循:

1. **前言:**前情提要，閱讀此文建議使用工具或知識.
2. 標出大主題(大字體+錨點)之後在細項列出要說明的細節
3. 小結：每篇都有一個小結快速總結今天重點

### 為什麼想要選擇此主題

選擇這個主題主要原因是

1. 沒有人整理一套較完整的Asp.net執行原始碼解析文章(從`Http`請求`IIS Server`,進入`CLR`前置動作),**asp.net mvc**原始碼解析
2. 台灣大部分的文章都是分享如何使用，很少文章有介紹如何運作．
3. 利用微軟開原後站在巨人肩膀上可以看更遠，理解**MVC**框架如何去設計具有一定的彈性.
4. 了解核心運作流程，更好改善或擴充現有專案架構（讓系統變得更有條理）

### 閱讀時建議事項

我在文章中會盡量寫出我看到精華部分,但此系列文可能對於**MVC**新手不太容易閱讀,因為**MVC**框架中運用到許多設計模式和OOP觀念(當初我在閱讀上也花了不少功夫)

個人覺得OOP有很重要一個點是盡量用**物件和物件關聯**，資料狀態轉移來了解程式碼.

> 簡白來說就是物件關聯和關係

### 文章收穫

我希望大家在閱讀完所有文章後可以獲得

1. **Http**對於**IIS Server**請求如何導向**Asp.net MVC**執行
2. **Asp.net MVC**原始碼有基本了解和知道哪幾個重要類別,了解後能依照系統需要替換改寫.
3. **Asp.net MVC**用到很多設計技巧(因為這是一個較大框架),希望大家能更了解設計模式如何運用在實戰中
4. 閱讀第一個框架原始碼會花不少時間,了解一個大框架後在去看其他框架閱讀時間會越來越少

## 進入主題

`Asp.net`基於`.NET Framework`框架所提供，開發**Web**應用程式的類別庫，封裝在`System.Web.dll`檔案中，提供使用者開發網頁，ASP.NET運行在安裝了.NET Framework的`IIS(Internet Information Services)`伺服器上

微軟大大近幾年也投入`Open Source`行列讓我們可以更方便來窺探，Asp.net運作原理. 這個[連結 Reference Source](https://referencesource.microsoft.com/) 可以查看微軟核心的`DLL`程式碼(這個網站是我們第一階段追code的好朋友)

`Asp.net`程式基本上是由`IIS`進行託管，介紹`Asp.net MVC`原始碼之前我們需要先了解`Asp.net`和`IIS`關係.

### 瀏覽器請求IIS流程

Web基於Http協定，它是一個無狀態協定，每次請求都是新的且不會紀錄之前請求的東西
下圖我畫出一個對於IIS請求基本會跑的流程圖.

![瀏覽器請求IIS流程](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/IIS_Asp.net_Process.png)

基本分為兩個區塊

* 粉紅色是`IIS Server`領域
  * 會有一個`Http.sys`的程式在監聽所有`Http`請求並交由`W3WP.exe`並透過`aspnet_isapi`來此次請求是否為靜態檔案.
* 藍色是`.Net CLR`領域由幾塊核心程式完成請求
  * ISAPIRunTime
  * HttpRuntime
  * HttpApplicationFactory
  * HttpApplication

之後會陸續介紹他們.

1. 請求**靜態檔案**透過路徑找尋**靜態檔案**並回傳.
2. 請求**非靜態檔案**透過`.Net CLR`執行返回結果.

### 如何辨別是否為靜態檔案?

如何辨別是否為靜態檔案,就需要談談`HttpHandler`的註冊表(後面有文章會說到)

基本上如果是請求`Html`,`css`,`js`...都會直接回傳不會在經過`.Net CLR`

### .Net CLR Httpmodule & Httphandler 核心模組

Asp.net所有應用程式都離不開兩個核心模組`Httpmodule & Httphandler`且最終會找到一個繼承於`IHttpHanlder`物件來處理請求.

在網路上看到一個很好地比喻**HttpModule & HttpHandler**

**Http**請求像是一個旅客身上帶著行李拿著票來搭火車.

* `HttpHandler` 是火車的終點站.
* `HttpModule` 是火車中途停靠的各站.

這個比喻可以很清楚知道每個請求透過`CLR`就是要找到一個`HttpHandler`來執行.

![圖片參考連結](https://www.codeproject.com/KB/web-image/thumbnailer/thumbnailer_pipeline.gif)

[圖片參考連結](https://www.codeproject.com/Articles/16120/Thumbnailer-HTTP-Handler)

### W3WP應用程式

當`IIS`在執行處理Http請求時工作管理員有一個`w3wp`應用程式在監聽.

此應用程式會依照`aspnet_isapi`模組來判斷此次請求是否走入`.net CLR`

![w3wp.PNG](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/itHelp/1/w3wp.PNG)

## 小結

今天我們了解到

1. 瀏覽器請求IIS基本流程
2. Asp.net核心模組**Httpmodule & Httphandler**
3. IIS有一個`Http.sys`程式在監聽所有`Http`請求
4. IIS透過一個`w3wp.exe`初步過濾判斷如何執行此請求.

瀏覽器發出**Http**請求給IIS,IIS透過`Http.sys`來監聽請求並交給`w3wp.exe`這個應用程式來判斷是否要交由`.net`託管處理此次請求.

下篇我們會來詳細講述`Httpmodule & Httphandler`
