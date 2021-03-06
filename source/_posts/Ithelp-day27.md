---
title: 動手DIY改造 Asp.net MVC- 自己動作建立一個DependencyResolver解析器(Autofac) (第27天)
date: 2019-10-08 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---

# Agenda<!-- omit in toc -->
- [前言](#%e5%89%8d%e8%a8%80)
- [Aufofac依賴注入容器](#aufofac%e4%be%9d%e8%b3%b4%e6%b3%a8%e5%85%a5%e5%ae%b9%e5%99%a8)
- [IDependencyResolver介面](#idependencyresolver%e4%bb%8b%e9%9d%a2)
  - [建立CustomerDependencyResolver(IDependencyResolver)](#%e5%bb%ba%e7%ab%8bcustomerdependencyresolveridependencyresolver)
  - [CustomerControllerActivator(IControllerActivator)](#customercontrolleractivatoricontrolleractivator)
- [在Application_Start中MVC替換成自己的解析器](#%e5%9c%a8applicationstart%e4%b8%admvc%e6%9b%bf%e6%8f%9b%e6%88%90%e8%87%aa%e5%b7%b1%e7%9a%84%e8%a7%a3%e6%9e%90%e5%99%a8)
- [在Controller使用注入](#%e5%9c%a8controller%e4%bd%bf%e7%94%a8%e6%b3%a8%e5%85%a5)
- [小結：](#%e5%b0%8f%e7%b5%90)

## 前言

產生`Controller`物件相關物件關係如下面**UML圖**

![relationship_pic.PNG](https://raw.githubusercontent.com/isdaniel/MyBlog/master/source/images/itHelp/13/IOC_Asp.netMVC.png)

透過`ControllerFactory`建立一個`Controller`控制器物件.而`ControllerFactory`依賴`IControllerActivator`物件產生`Controller`.

上面`IControllerActivator`可以透過建立使用我們的依賴注入容器來替換原本反射產生物件.

`DependencyResolver`是**MVC**提供的一個可替換物注入點,今天我們會藉由他來我們實現注入**MVC**方式.

## Aufofac依賴注入容器

在實現自己的`DependencyResolver`前先談談Autofac容器做甚麼用的?

我之前有寫一篇[IOC(控制反轉)，DI(依賴注入) 深入淺出~~](https://isdaniel.github.io/ioc-di.html),講述**IOC(控制反轉)，DI(依賴注入)**這兩個設計技巧的理念核心.

> 言簡意賅可以統一交由容器來幫忙管理物件生命週期和建立方式,也管理物件相依性,兩個重點我們使得只需要提供使用類別的特徵(型別或其他可辨別特徵),容器就提供給我們相對應的物件.

在`Autofac`有需多使用方式這裡就不一一介紹,有興趣讀者可以上網`google`或是查閱`Autofac`官方文件

## IDependencyResolver介面

`DependencyResolver`是一個靜態物件,**MVC application**使用同一個解析器(`DefaultDependencyResolver`)而他有一個`SetResolver`方法可以替換成其他`DependencyResolver`

`IDependencyResolver`有兩個方法需要實現.

```csharp
public interface IDependencyResolver
{
    object GetService(Type serviceType);
    IEnumerable<object> GetServices(Type serviceType);
}
```

**MVC**依賴於`GetService`和`GetServices`,取得物件實例並提供一個抽象提供外部提供修改或擴充.

預設使用(`DefaultDependencyResolver`)這個解析器來取得我們物件(`DefaultDependencyResolver`解析器使用`Activator.CreateInstance(serviceType);`建立物件)

### 建立CustomerDependencyResolver(IDependencyResolver)

這邊我們利用`autofac`來完成建立物件動作,先建立一個`ILifetimeScope _container`由建構子注入此物件.

```csharp
public class CustomerDependencyResolver : IDependencyResolver
{
    private readonly ILifetimeScope _container;

    public CustomerDependencyResolver(ILifetimeScope container)
    {
        if (container == null)
            throw new ArgumentNullException(nameof (container));

        _container = container;
    }

    public object GetService(Type serviceType)
    {
        return _container.ResolveOptional(serviceType);
    }

    public IEnumerable<object> GetServices(Type serviceType)
    {
        return (IEnumerable<object>) _container.ResolveOptional(typeof (IEnumerable<>).MakeGenericType(serviceType));
    }
}
```

在`GetService`呼叫`ResolveOptional`方法透過`Type`到容器中搜尋匹配的物件並返回.

### CustomerControllerActivator(IControllerActivator)

`IControllerActivator`有一個`Create`方法,`ControllerFacotry`靠它來幫我們產生使用`Controller`物件,而我們在這邊建立自己`IControllerActivator`並在`Create`方法中實現自己得邏輯.透過`DependencyResolver`來產生物件(替換成`CustomerDependencyResolver`)

```csharp
public class CustomerControllerActivator : IControllerActivator
{
    public IController Create(RequestContext requestContext, Type controllerType)
    {
        return (IController) DependencyResolver.Current.GetService(controllerType);
    }
}
```

我們會在`Autofac`容器註冊目前`Assembly`所有繼承`IController`物件.

```csharp
//注入typeof(MvcApplication).Assembly 中所有繼承IController物件.
builder.RegisterControllers(typeof(MvcApplication).Assembly);
```

在上面`CustomerControllerActivator.Create`會透`Autofac`解析器幫我們建立`Controller`

## 在Application_Start中MVC替換成自己的解析器

1. 首先利用`ControllerBuilder`的`SetControllerFactory`方法,重新替換使用`ControllerFacotry`.

2. 在利用`builder.RegisterControllers`注入`typeof(MvcApplication).Assembly`中所有繼承`IController`物件.

3. 註冊`IMemberService`介面物件(裡面有一個`int GetMemberBalance(int memberId);`方法來模擬取得會員餘額)

4. `DependencyResolver.SetResolver(new CustomerDependencyResolver(builder.Build()))`替換成我們使用的解析器

> 因為`ControllerFacotry`預設使用`DefaultControllerActivator`,而我們需要替換成自己建立得`CustomerControllerActivator`並利用容器來幫我們注入.

```csharp
public class MvcApplication : HttpApplication
{
    protected void Application_Start()
    {
        RouteConfig.RegisterRoutes(RouteTable.Routes);
        BundleConfig.RegisterBundles(BundleTable.Bundles);
        //把DefaultControllerFactory 中的IControllerActivator替換成我們自己寫的CustomerControllerActivator
        ControllerBuilder.Current.SetControllerFactory(
            new DefaultControllerFactory(new CustomerControllerActivator()));
        AutofacRegister();
    }

    private static void AutofacRegister()
    {
        ContainerBuilder builder = new ContainerBuilder();
        //注入typeof(MvcApplication).Assembly 中所有繼承IController物件.
        builder.RegisterControllers(typeof(MvcApplication).Assembly);
        builder.RegisterType<MemberService>().As<IMemberService>();
        //替換成自己的DependencyResolver
        DependencyResolver.SetResolver(new CustomerDependencyResolver(builder.Build()));
    }
}
```

## 在Controller使用注入

`HomeController`控制器中在建構子注入,並呼叫`IMemberService.GetMemberBalance`方法

執行專案請求`Home/About`頁面可以看到`ViewBag.Message`已經成功顯示一個**HardCode**餘額了.

```csharp
public class HomeController : Controller
{
    private readonly IMemberService _service;

    public HomeController(IMemberService service)
    {
        _service = service;
    }

    public ActionResult About()
    {
        ViewBag.Message = $"Member Balance { _service.GetMemberBalance(123)}";

        return View();
    }
}
```

## 小結：

`DefaultControllerActivator`使用反射建立一個`Controller`物件

然而`IControllerActivator`提供一個產生`Controller`接口,而我們可以藉由實現此介面並使用`DependencyResolver`靜態物件產生`Controller`物件(藉由容器框架產生).

最後會把`Controller`依賴物件藉由**依賴注入容器**注入進去.

[Github範例程式原始碼](https://github.com/isdaniel/ItHelp_MVC_10th/tree/CustomerContainer) `CustomerContainer`分支上