---
title: Asp.Net支柱 IHttpMoudle & IHttphandler (第2天)
date: 2019-09-13 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---
# Agenda<!-- omit in toc -->
- [前言：](#%e5%89%8d%e8%a8%80)
  - [IHttpHandler和HttpModule關係](#ihttphandler%e5%92%8chttpmodule%e9%97%9c%e4%bf%82)
- [Asp.net_Application_Event生命週期](#aspnetapplicationevent%e7%94%9f%e5%91%bd%e9%80%b1%e6%9c%9f)
  - [Event事件名稱：](#event%e4%ba%8b%e4%bb%b6%e5%90%8d%e7%a8%b1)
  - [事件方法說明：](#%e4%ba%8b%e4%bb%b6%e6%96%b9%e6%b3%95%e8%aa%aa%e6%98%8e)
- [IHttpHandler](#ihttphandler)
- [IHttpModule](#ihttpmodule)
- [自己建立一個 IHttpHandler](#%e8%87%aa%e5%b7%b1%e5%bb%ba%e7%ab%8b%e4%b8%80%e5%80%8b-ihttphandler)
    - [Web.Config註冊上面撰寫的IHttpHandler](#webconfig%e8%a8%bb%e5%86%8a%e4%b8%8a%e9%9d%a2%e6%92%b0%e5%af%ab%e7%9a%84ihttphandler)
- [自己建立一個 IHttpModule](#%e8%87%aa%e5%b7%b1%e5%bb%ba%e7%ab%8b%e4%b8%80%e5%80%8b-ihttpmodule)
    - [建立一個類別實現IHttpModule](#%e5%bb%ba%e7%ab%8b%e4%b8%80%e5%80%8b%e9%a1%9e%e5%88%a5%e5%af%a6%e7%8f%beihttpmodule)
    - [Web.Config註冊上面撰寫的IHttpModule](#webconfig%e8%a8%bb%e5%86%8a%e4%b8%8a%e9%9d%a2%e6%92%b0%e5%af%ab%e7%9a%84ihttpmodule)
  - [程式碼](#%e7%a8%8b%e5%bc%8f%e7%a2%bc)
    - [建立一個類別實現IHttpHander](#%e5%bb%ba%e7%ab%8b%e4%b8%80%e5%80%8b%e9%a1%9e%e5%88%a5%e5%af%a6%e7%8f%beihttphander)
- [ApplicationHost.config 設定擋](#applicationhostconfig-%e8%a8%ad%e5%ae%9a%e6%93%8b)
  - [IIS預設設定Module在哪邊?](#iis%e9%a0%90%e8%a8%ad%e8%a8%ad%e5%ae%9amodule%e5%9c%a8%e5%93%aa%e9%82%8a)
    - [IIS](#iis)
    - [IISExpress](#iisexpress)
- [小結](#%e5%b0%8f%e7%b5%90)
- [參考資料：](#%e5%8f%83%e8%80%83%e8%b3%87%e6%96%99)

## 前言：

上一篇文章我們有說 Asp.net 有兩個核心組件 `IHttpMoudle & IHttphandler`

`Asp.net` 是一個`pipeline`的模型，理解這些管道除了可以讓我們節省許多不必要的資源浪費也可讓我們程式架構更加有條理.

就像我們在`Asp.net MVC`如果要寫權限驗證程式碼,雖然可以寫在`Controller`的`Action`中

但更好做法是我可以寫一個類別繼承`AuthorizeAttribute`並`override` `OnAuthorization`方法並掛上此標籤.

更了解這些原理可以讓我們寫程式事半功倍.

### IHttpHandler和HttpModule關係

Http請求像是一個旅客身上帶著行李拿著票來搭火車.

* `HttpHandler` 是火車的終點站.
* `HttpModule` 是火車中途停靠的各站.

## Asp.net_Application_Event生命週期

我們已經知道`Asp.net`是一個請求處理響應的管道而這個管道中微軟有提供許多點可以讓我們進行客製化的擴充程式撰寫

> 事件可藉由`IHttpModule`來擴充註冊

### Event事件名稱：

官網列出可用事件很多，下面列出我有用過的幾個事件和其功用.

* BeginRequest
* AuthorizeRequest
* PostResolveRequestCache
* MapRequestHandler
* AcquireRequestState
* PreRequestHandlerExecute
* PostRequestHandlerExecute
* EndRequest

### 事件方法說明：

* `BeginRequest`: 已經啟動要求。如果要在要求前執行某個動作 (例如, 在每頁頁首顯示廣告橫幅), 請同步處理這個事件。
* `AuthorizeRequest`: 您可以在內部使用這個事件, 以實作授權機制 (例如, 將存取控制清單 (ACL) 儲存在資料庫, 而非存入檔案系統)。您也可以覆寫這個事件, 但無此必要。
* `PostResolveRequestCache`：當 ASP.NET 略過目前事件處理常式的執行並允許快取模組從快取中服務要求時發生。
* `MapRequestHandler`：ASP.NET 基礎結構會使用事件來判斷目前要求的要求處理常式。 如需詳細資訊
* `AcquireRequestState`: 工作階段狀態是擷取自狀態儲存區。如果要建置自已的狀態管理模組, 則可以同步處理這個事件, 以便從狀態儲存區擷取「工作階段」狀態。
* `PreRequestHandlerExecute`: 這個事件會在執行 HTTP 處理常式之前產生。
* 在介於`PreRequestHandlerExecute`和`PostRequestHandlerExecute`事件之間會執行`HttpHandler`程式碼.
* `PostRequestHandlerExecute`: 這個事件會在執行 HTTP 處理常式之後產生。
* `EndRequest`: 要求已完成。您可能想要建置偵錯模組, 以便收集要求的全部資訊, 然後再將資訊寫入網頁中。

`IHttpHandler`和`IHttpModule`關係如 [Implementing HTTPHandler and HTTPModule in ASP.NET](https://www.codeproject.com/Articles/335968/Implementing-HTTPHandler-and-HTTPModule-in-ASP-NET) 文章提到

![img](https://1.bp.blogspot.com/--nEyvA5pGLk/WXn6YudHYzI/AAAAAAACWLg/7N81JLv5SfYPMvYmGdUnNHg4_p3Shb5DACLcBGAs/s1600/overview.jpg)

每個請求一定會通過所有被註冊的`IHttpModule`,而最終會執行一個`IHttpHandler`後進行返回.

我們常聽到的`Asp.net` `Webform`，`MVC`都是經過管道`Module`並執行相對應的`Handler`.

> 所以HttpHanlder 和 HttpMoudule 搭配使用達到更強大的功能.


## IHttpHandler

[MSDN](https://docs.microsoft.com/zh-tw/dotnet/api/system.web.ihttphandler?view=netframework-4.8)說明
> 您可以撰寫自訂的 HTTP 處理常式來處理特定的預先定義的任何 Common Language Specification (CLS) 標準的語言中的 HTTP 要求的類型。 可執行程式碼中定義HttpHandler類別，而不是傳統的 ASP 或 ASP.NET Web 網頁，這些特定的要求回應。 HTTP 處理常式提供您一種低層級的要求和回應服務的 IIS Web 伺服器互動，以及大部分 ISAPI 擴充程式類似，但使用簡單的程式設計模型提供的功能。

`IHttpHandler`是一個可以讓我們實現的介面
裡面包含：

**屬性:**

```csharp
public bool IsReusable { get; }
```

> 取得值，指出另一個要求是否可以使用 `IHttpHandler` 執行個體。

**方法:**

```csharp
public void ProcessRequest(HttpContext context)
```

> 以實作 IHttpHandler 介面的自訂 HttpHandler 來啟用 HTTP Web 要求的處理。



## IHttpModule

[MSDN](https://support.microsoft.com/en-us/help/307985/info-asp-net-http-modules-and-http-handlers-overview)說明

> Modules are called before and after the handler executes. Modules enable developers to intercept, participate in, or modify each individual request. Modules implement the IHttpModule interface, which is located in the System.Web namespace.

處理常式 (Handler) 在執行前後，會呼叫模組 (Module)。 模組可以讓開發人員攔截、參與或修改每個要求。

更印證了

>　如果把Http請求當作火車那
> * `IHttpHandler`是火車的終點
> * `IHttpModule`是沿路經過的站點

要查看有哪寫`IHttpModule`或`IHttpHandler`被註冊可以看`applicationhost.config`檔案
> 路徑：C:\Users\[user]\Documents\IISExpress\config\applicationhost.config



## 自己建立一個 IHttpHandler

在前面有說到每個Http請求的最終是為了給一個`HttpHander`來執行處理.

像我們常看到的

* ASP.NET page (*.aspx)
* Web service (*.asmx)
* Generic Web  (*.ashx)

甚至是`MVC (MvcHandler)`都是實現於`IHttpHander`介面

這邊介紹如果要如何建立自己`HttpHander`.

#### Web.Config註冊上面撰寫的IHttpHandler

我們要在`Web.Config`中設定我們撰寫的`HttpHandler`

這是一個範例:

```xml
<configuration>
    <system.webServer>
    <handlers>
        <add verb="*" name="MyHttpHandler" path="*.cspx"  type="HttpHandler_HttpModule.MyHttpHandler"/>
    </handlers>
    </system.webServer>
</configuration>
```

把`handlers`加入在`system.webServer`結點中.

裡面有幾個`Attribute`

* `verb`：請求動作 `GET,POST,PUT...`如果是`*`代表全部請請動作都合用.
* `path`：請求那些副檔名會執行此`HttpHandler`
* `type`：註冊的`HttpHandler`類型.

其中最要注意的是type Attribute.

```xml
 <add verb="*" name="MyHttpHandler" path="*.cspx"  type="(namespace).(classname)"/>
```

最後我們就可以請求 `http://xxxx/Mypage.cspx` 來試試看我們的結果.

我們將請求`*.cspx`副檔名的所有請求都當作要透過`CLR`來執行.



## 自己建立一個 IHttpModule

每個被註冊的`HttpModule`是Http請求必經之路.

* `Asp.net MVC` 是透過`System.Web.Routing.UrlRoutingModule` 這個`HttpModule`來完成切入的.

使用`IHttpModule`須完成幾個步驟:

1. 建立一個類別實現`IHttpModule`
2. `Web.Config`註冊上面撰寫的`IHttpModule`

#### 建立一個類別實現IHttpModule

這個範例會在頁面上顯示 IIS Pipeline Event的執行順序.

1. `public void Init(HttpApplication context)` 把 `HttpApplication` 中的event做擴充.

```csharp
public class MyHttpModule:IHttpModule
{
    public void Init(HttpApplication context)
    {
        context.BeginRequest += (sender, args) => ShowStep(sender, "BeginRequest");

        context.AuthorizeRequest += (sender, args) => ShowStep(sender, "AuthorizeRequest");

        context.PostResolveRequestCache += (sender, args) => ShowStep(sender, "PostResolveRequestCache");

        context.MapRequestHandler += (sender, args) => ShowStep(sender, "MapRequestHandler");

        context.AcquireRequestState += (sender, args) => ShowStep(sender, "AcquireRequestState");

        context.PreRequestHandlerExecute += (sender, args) => ShowStep(sender, "PreRequestHandlerExecute");

        //這中間執行IHttpHandler.

        context.PostRequestHandlerExecute += (sender, args) => ShowStep(sender, "PostRequestHandlerExecute");

        context.EndRequest += (sender, args) => ShowStep(sender, "EndRequest");

        context.PreSendRequestHeaders += (sender, args) => ShowStep(sender, "PreSendRequestHeaders");
    }

    private void ShowStep(object app,string eventName)
    {
        var http = (HttpApplication)app;
        http.Response.Write($"Step {eventName}<br/>");
    }

    public void Dispose()
    {
    }
}
```

#### Web.Config註冊上面撰寫的IHttpModule

註冊方法和`IHttpHander`很類似,一樣在`system.webServer`節點下加入`modules`

```xml
<configuration>
    <system.webServer>
    <modules>
      <add name="MyHttpModule" type="HttpHandler_HttpModule.MyHttpModule"/>
    </modules>  
    </system.webServer>
</configuration>
```

-----


### 程式碼

使用`IHttpHandler`須完成幾個步驟:

1. 建立一個類別實現`IHttpHander`
2. `Web.Config`註冊上面撰寫的`IHttpHandler`

#### 建立一個類別實現IHttpHander

繼承完`IHttpHandler`我們會實現兩個方法.

1. `ProcessRequest(HttpContext context)` 執行此次請求動作.
2. `bool IsReusable { get; }` 是否要將此次請求加入快取中重用.

```csharp
public class MyHttpHandler : IHttpHandler
{
    public void ProcessRequest(HttpContext context)
    {
        context.Response.ContentType = "text/html";
        context.Response.Write("==================<br/>");
        context.Response.Write("Hello World<br/>");
        context.Response.Write("==================<br/>");
    }

    public bool IsReusable { get; }
}
```



## ApplicationHost.config 設定擋

前面有說到`aspnet_isapi.dll`會去判斷此次請求要怎麼處理,她會去讀取`handlers`註冊節點的對應表來判斷是否要透過`CLR`處理此次請求.

### IIS預設設定Module在哪邊?

她是透過 `ApplicationHost.config` 來抓取，`HttpHandler` 和 `HttpModule`設定.

這邊分為兩個部分:

#### IIS

IIS可借由`HandlerMappings`和`Module`來設置

![瀏覽器請求IIS流程](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/IISModule_Hanlder.PNG)

-----

* HandlerMappings

![瀏覽器請求IIS流程](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/Handler.PNG)

可以看到有許多預設要處理的附檔名(`.aspx`,`.svc`)在裡面都可以看到.

* Module

![瀏覽器請求IIS流程](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/Module.PNG)

IIS預設幫忙載入許多`Moudle`我們自己客製化的`Module`也可以在這邊設定

-----

#### IISExpress

基本上`IISExpress` 全域 config 會放在這個路徑

> C:\Users\%USERPROFILE%\Documents\IISExpress\config

如果VS版本高於2015以上執行Web專案時，會在每個專案上建立一個`.vs`資料夾並把`ApplicationHost.config`複製一份放到下面影片的路徑中

![瀏覽器請求IIS流程](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/IISExpress.gif)

-----

[範例原始碼下載](https://github.com/isdaniel/HttpHanlder_Vs_HttpModule)

## 小結

今天我們學到

1. 自己建立一個`Httpmodule`
2. 自己建立一個`Httphandler` 
3. `ApplicationHost.config`設定擋和`Httpmodule & Httphandler`關係且如何設定.

今天先丟一個問題給大家那，為什麼Asp.net可以依賴`IHttphandler`介面來完成請求這是怎麼實現的?

下篇我們會來回答上面的答案.

##  參考資料：

* [HTTP Handlers and HTTP Modules Overview](https://docs.microsoft.com/en-us/previous-versions/aspnet/bb398986(v=vs.100)#Features)
* [Implementing HTTPHandler and HTTPModule in ASP.NET](https://www.codeproject.com/Articles/335968/Implementing-HTTPHandler-and-HTTPModule-in-ASP-NET)
