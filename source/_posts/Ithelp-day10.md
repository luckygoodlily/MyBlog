---
title: 透過MvcRouteHandler取得呼叫IHttphandler (第10天)
date: 2019-09-21 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---
# Agenda<!-- omit in toc -->
- [前言](#%e5%89%8d%e8%a8%80)
- [MVC取得使用HttpHandler (IHttpHandler)](#mvc%e5%8f%96%e5%be%97%e4%bd%bf%e7%94%a8httphandler-ihttphandler)
- [MVC呼叫的HttpHandler (MvcHandler)](#mvc%e5%91%bc%e5%8f%ab%e7%9a%84httphandler-mvchandler)
- [小結](#%e5%b0%8f%e7%b5%90)

## 前言

前一篇介紹路由封裝了Http請求路徑資訊可以讓我們找到相對應的`Action`和`Controller`並呼叫執行外，也可透過`MapPageRoute`來將請求教給`.aspx`實體檔案來處理請求.

`Route`甚至可以讓我們自己客製化處理`HttpHandler` 在Route中建立處理客製化HttpHandler可謂很有彈性

本篇介紹`Route`物件建立`MvcRouteHandler`物件且如何取到`IHttpHandler`.

我有做一個可以針對於[Asp.net MVC Debugger](https://github.com/isdaniel/Asp.net-MVC-Debuger)的專案，只要下中斷點就可輕易進入Asp.net MVC原始碼.

## MVC取得使用HttpHandler (IHttpHandler)

之前說到我們透過`MapRoute`擴展方法加入一個`Route`物件給`RouteCollection`**全域路由集合**.

在`Route`使用的`IRouteHandler`介面是由`MvcRouteHandler`來實現

```csharp
Route route = new Route(url, new MvcRouteHandler())
{
    Defaults = CreateRouteValueDictionaryUncached(defaults),
    Constraints = CreateRouteValueDictionaryUncached(constraints),
    DataTokens = new RouteValueDictionary()
};
```

`IRouteHandler`最重要的是`IHttpHandler IRouteHandler.GetHttpHandler(RequestContext requestContext)`會取得一個`IHttpHandler`物件.

```csharp
public class MvcRouteHandler : IRouteHandler
{
    private IControllerFactory _controllerFactory;

    public MvcRouteHandler()
    {
    }

    public MvcRouteHandler(IControllerFactory controllerFactory)
    {
        _controllerFactory = controllerFactory;
    }

    protected virtual IHttpHandler GetHttpHandler(RequestContext requestContext)
    {
        //設置Session使用
        requestContext.HttpContext.SetSessionStateBehavior(GetSessionStateBehavior(requestContext));
        return new MvcHandler(requestContext);
    }

    protected virtual SessionStateBehavior GetSessionStateBehavior(RequestContext requestContext)
    {
        string controllerName = (string)requestContext.RouteData.Values["controller"];
        if (String.IsNullOrWhiteSpace(controllerName))
        {
            throw new InvalidOperationException(MvcResources.MvcRouteHandler_RouteValuesHasNoController);
        }

        IControllerFactory controllerFactory = _controllerFactory ?? ControllerBuilder.Current.GetControllerFactory();
        return controllerFactory.GetControllerSessionBehavior(requestContext, controllerName);
    }

    #region IRouteHandler Members

    IHttpHandler IRouteHandler.GetHttpHandler(RequestContext requestContext)
    {
        return GetHttpHandler(requestContext);
    }

    #endregion
}
```

上面程式碼可以看到**Mvc**使用`IHttpHandler`是`MvcHandler`

## MVC呼叫的HttpHandler (MvcHandler)

`MvcHandler`類別中主要核心的程式碼做了幾件事情.

1. 使用一個`Adapter`對於`HttpContext`物件把他轉成可以繼承於`HttpContextBase`的`HttpContextWrapper`類別.
2. 透過`ProcessRequestInit`取得執行`controller`物件並且呼叫執行方法.
3. 最後透過`ReleaseController`釋放之前使用過資源

```csharp
protected virtual void ProcessRequest(HttpContext httpContext)
{
    HttpContextBase httpContextBase = new HttpContextWrapper(httpContext);
    ProcessRequest(httpContextBase);
}

protected internal virtual void ProcessRequest(HttpContextBase httpContext)
{
    IController controller;
    IControllerFactory factory;
    //取得 控制器工廠(預設DefaultControllerFactory) 和 要執行的Controller
    ProcessRequestInit(httpContext, out controller, out factory);

    try
    {
        controller.Execute(RequestContext);
    }
    finally
    {
        factory.ReleaseController(controller);
    }
}
```

## 小結

今天我們知道MVC使用`HttpHandler`是`MvcHandler`透過並`MvcRouteHandler`物件來返回.

下圖簡單展現MVC使用的`HttpModule`和`HttpHandler`關係

![relationship_pic.PNG](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/itHelp/10/relationship_pic.PNG)

1. 在`UrlRoutingMoudule`註冊事件.
2. 取得符合Http請求`Route`物件
3. 呼叫`MvcRouteHandler`取得`MvcHandler`物件
4. 執行`MvcHandler`的`ProcessReqeust`方法

下面會陸續介紹**MVC**是如何取得`Controller`物件
