---
title: 談談Controller幾個重要成員 (第12天)
date: 2019-09-23 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---
# Agenda<!-- omit in toc -->
- [前言](#%e5%89%8d%e8%a8%80)
- [ControllerBase(Controller基礎類別)](#controllerbasecontroller%e5%9f%ba%e7%a4%8e%e9%a1%9e%e5%88%a5)
  - [SessionStateTempDataProvider 控制儲存TempData](#sessionstatetempdataprovider-%e6%8e%a7%e5%88%b6%e5%84%b2%e5%ad%98tempdata)
- [ControllerBase Excute方法](#controllerbase-excute%e6%96%b9%e6%b3%95)
- [Controller](#controller)
  - [ExecuteCore](#executecore)
- [ControllerContext](#controllercontext)
  - [ControllerContext](#controllercontext-1)
  - [AuthorizationContext](#authorizationcontext)
- [小結：](#%e5%b0%8f%e7%b5%90)

## 前言

上篇得知MVC預設透過`DefaultControllerFactory`反射方式動態建立`Controller`物件

本篇會分享我們常用到`Controller`基礎類別和相關物件.

> 我有做一個可以針對於[Asp.net MVC Debugger](https://github.com/isdaniel/Asp.net-MVC-Debuger)的專案，只要下中斷點就可輕易進入Asp.net MVC原始碼.

## ControllerBase(Controller基礎類別)

`ControllerBase`具有如下幾個重要的屬性

1. `TempData`：將設置資料存於`Session`中,生命週期除了當下請求, 導頁後仍可續存.
2. `ViewBag`：儲存`Controller`向`view`傳遞資料或變數 (型別`dynamic`)
3. `ViewData`：儲存`Controller`向`view`傳遞資料或變數 (型別`ViewDataDictionary`)

雖說`ViewBag`和`ViewData`看起來使用不同的物件,但從程式碼了解到其實`ViewBag`也是使用`ViewData`引用.

```csharp
public abstract class ControllerBase : IController
{   
    public ControllerContext ControllerContext { get; set; }
    public TempDataDictionary TempData
    {
        get
        {
            if (ControllerContext != null && ControllerContext.IsChildAction)
            {
                return ControllerContext.ParentActionViewContext.TempData;
            }
            if (_tempDataDictionary == null)
            {
                _tempDataDictionary = new TempDataDictionary();
            }
            return _tempDataDictionary;
        }
        set { _tempDataDictionary = value; }
    }
    public dynamic ViewBag
    {
        get
        {
            if (_dynamicViewDataDictionary == null)
            {
                _dynamicViewDataDictionary = new DynamicViewDataDictionary(() => ViewData);
            }
            return _dynamicViewDataDictionary;
        }
    }
    public ViewDataDictionary ViewData { get; set; }
}
```

### SessionStateTempDataProvider 控制儲存TempData

上面說到`TempData`字典集合生命週期除了當下請求, 導頁後仍可續存.原因是在`SessionStateTempDataProvider`將資料存在`Session`中

```csharp
controllerContext.HttpContext.Session["__ControllerTempData"]
```

可以透過上面程式碼取得當前的`TempData`字典集合物件.

## ControllerBase Excute方法

`ControllerBase`這個類別繼承`IController`,前篇說到在**HttpHandler** `ProcessRequest`方法會透過反射找到一個符合Http請求`IController`介面物件.
並呼叫其`Execute`方法

在`Execute`做了幾件事情.

* 初始化`ControllerContext`物件,對於`RequestContext`簡易封裝.
* `ExecuteCore`呼叫`Hock`方法(`ExecuteCore`是一個抽象方法提供繼承他的物件實做,預設是`Controller`類別)

```csharp
protected virtual void Execute(RequestContext requestContext)
{
    if (requestContext == null)
    {
        throw new ArgumentNullException("requestContext");
    }
    if (requestContext.HttpContext == null)
    {
        throw new ArgumentException(MvcResources.ControllerBase_CannotExecuteWithNullHttpContext, "requestContext");
    }

    VerifyExecuteCalledOnce();
    Initialize(requestContext);

    using (ScopeStorage.CreateTransientScope())
    {
        ExecuteCore();
    }
}

void IController.Execute(RequestContext requestContext)
{
    Execute(requestContext);
}
```

## Controller

在專案中`Controller`是我們預設使用繼承控制器類別，此類別中定義了很多的輔助方法和屬性讓撰寫控制器變得簡單。
`Controller`類別除了直接繼承`ControllerBase`之外,`Controller`還顯式實現`IController`和`IAsyncController`介面，跟`ASP.NET MVC` 四大篩選器（`IAuthorizationFilter,IActionFilter、IResultFilter,IExceptionFilter`）的4個介面。

```Csharp
public abstract class Controller : 
	ControllerBase, 
	IActionFilter, 
	IAuthenticationFilter, 
	IAuthorizationFilter,
	IDisposable, 
	IExceptionFilter,
	IResultFilter,
	IAsyncController, 
	IAsyncManagerContainer
{
}
```

### ExecuteCore

在`Controller`重載實做`ExecuteCore`方法.

主要透過`GetActionName(RouteData)`取得執行的`Action`名稱,並透過`ActionInvoker`取得要Invoker的`ActionInvoker`.

* `PossiblyLoadTempData`：建立載入`TempData`
* `PossiblySaveTempData`：儲存`TempData`的資料

```csharp
protected override void ExecuteCore()
{
    // If code in this method needs to be updated, please also check the BeginExecuteCore() and
    // EndExecuteCore() methods of AsyncController to see if that code also must be updated.
    PossiblyLoadTempData();
    try
    {
        string actionName = GetActionName(RouteData);
        if (!ActionInvoker.InvokeAction(ControllerContext, actionName))
        {
            HandleUnknownAction(actionName);
        }
    }
    finally
    {
        PossiblySaveTempData();
    }
}
```

## ControllerContext

第一次初始話`ControllerContext`利用建構子

```csharp
public ControllerContext(RequestContext requestContext, ControllerBase controller) 
```

在`ControllerBase.Initialize`方法對於`ControllerContext`初始化,這個上下文資料封裝了許多此次請求的資料.

```csharp
protected virtual void Initialize(RequestContext requestContext)
{
    ControllerContext = new ControllerContext(requestContext, this);
}
```

後面對於繼承`ControllerContext`的`Context`傳入第一次初始化`ControllerContext`物件,
在建構子函數把傳入`ControllerContext`的`RequestContext`資料填入繼承`ControllerContext`物件中

下面是**MVC**有繼承`ControllerContext`類別

* AuthorizationContext
* ExceptionContext
* AuthenticationChallengeContext
* ResultExecutedContext
* ViewContext
* ResultExecutingContext

### ControllerContext 

在原始碼中可以看到`ControllerContext(ControllerContext controllerContext)`很巧妙把自身類別當作建構子方法參數傳入.

```csharp
Controller = controllerContext.Controller;
RequestContext = controllerContext.RequestContext;

```

主要是要把`RequestContext`值給填充,之後就可以利用`RequestContext`取得理面一些資料.

```csharp
public class ControllerContext
{
	internal const string ParentActionViewContextToken = "ParentActionViewContext";
	private HttpContextBase _httpContext;
	private RequestContext _requestContext;
	private RouteData _routeData;

	// parameterless constructor used for mocking
	public ControllerContext()
	{
	}

	protected ControllerContext(ControllerContext controllerContext)
	{
		if (controllerContext == null)
		{
			throw new ArgumentNullException("controllerContext");
		}

		Controller = controllerContext.Controller;
		RequestContext = controllerContext.RequestContext;
	}

	public ControllerContext(HttpContextBase httpContext, RouteData routeData, ControllerBase controller)
		: this(new RequestContext(httpContext, routeData), controller)
	{
	}

	public ControllerContext(RequestContext requestContext, ControllerBase controller)
	{
		if (requestContext == null)
		{
			throw new ArgumentNullException("requestContext");
		}
		if (controller == null)
		{
			throw new ArgumentNullException("controller");
		}

		RequestContext = requestContext;
		Controller = controller;
	}
	//....
}
```

### AuthorizationContext

我們看一下`Authorization`filter用到的參數`AuthorizationContext`.

在`InvokeAuthorizationFilters`方法將`AuthorizationContext`初始化

```csharp
AuthorizationContext context = new AuthorizationContext(controllerContext, actionDescriptor);
```

其中傳入參數`controllerContext`是第一次透過`ControllerBase.Initialize`初始化`Context`.

```csharp
public class AuthorizationContext : ControllerContext
{
	// parameterless constructor used for mocking
	public AuthorizationContext()
	{
	}

	public AuthorizationContext(ControllerContext controllerContext)
		: base(controllerContext)
	{
	}

	public AuthorizationContext(ControllerContext controllerContext, ActionDescriptor actionDescriptor) : base(controllerContext)
	{
		if (actionDescriptor == null)
		{
			throw new ArgumentNullException("actionDescriptor");
		}

		ActionDescriptor = actionDescriptor;
	}

	public virtual ActionDescriptor ActionDescriptor { get; set; }

	public ActionResult Result { get; set; }
}
```

之後再呼叫`base(controllerContext)`利用`ControllerContext`建構子把資料填充.

## 小結：

`Asp.net MVC`對於為了方便我們使用控制器所以對於`Controller`進行許多資料封裝,讓我們只要繼承`Controller`就可以方便使用許多屬性.

下圖是`Controller`核心類別關係圖.`Controller`類別左右兩側有本次沒介紹到類別(之後會介紹到)

![relationship_pic.PNG](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/itHelp/12/controller_uml.png)

當我看到`ControllerContext`的設計時讓我驚艷的,因為他把**MVC**用到`Context`都關聯綁定到一個類別中.

因為在商業邏輯中會有許多`Model`類別,且這些類別資料存在一定的相關性,我覺得這個設計可以使用可以大大改善資料傳遞上的麻煩,讓程式寫起來更安全,簡單

之後我會把上面的**UML**圖慢慢畫出來,一步一步揭開`Asp.net MVC`面紗.
