---
title: 動手DIY改造 Asp.net MVC- 建立自己ActionInvoker和Model綁定機制 (第28天)
date: 2019-10-09 10:00:00
tags: [C#,Asp.net,Asp.net-MVC,SourceCode,11th鐵人賽]
categories: [11th鐵人賽]
---

# Agenda<!-- omit in toc -->
- [前言](#%e5%89%8d%e8%a8%80)
- [建立自己的IActionInvoker(CustomerActionInvoker)](#%e5%bb%ba%e7%ab%8b%e8%87%aa%e5%b7%b1%e7%9a%84iactioninvokercustomeractioninvoker)
  - [進行呼叫測試](#%e9%80%b2%e8%a1%8c%e5%91%bc%e5%8f%ab%e6%b8%ac%e8%a9%a6)
- [改進GetValueTypeInstance方法(建立ValueProvider)](#%e6%94%b9%e9%80%b2getvaluetypeinstance%e6%96%b9%e6%b3%95%e5%bb%ba%e7%ab%8bvalueprovider)
  - [建立一個ValueProviderBase抽象類別](#%e5%bb%ba%e7%ab%8b%e4%b8%80%e5%80%8bvalueproviderbase%e6%8a%bd%e8%b1%a1%e9%a1%9e%e5%88%a5)
  - [似成相識IValueProvider介面](#%e4%bc%bc%e6%88%90%e7%9b%b8%e8%ad%98ivalueprovider%e4%bb%8b%e9%9d%a2)
- [小結:](#%e5%b0%8f%e7%b5%90)

## 前言

今天要分享對於`ActionInvoker`進行替換成自己客制化的`IActionInvoker`

在**MVC**原始碼中有個`CreateActionInvoker`方法來取得一個`IActionInvoker`物件,可以看到她會先透過`Resolver.GetService`從解析器中取得我們的`IActionInvoker`如果沒有在`new`一個`AsyncControllerActionInvoker`物件.

```csharp
protected virtual IActionInvoker CreateActionInvoker()
{
    IAsyncActionInvokerFactory asyncActionInvokerFactory = Resolver.GetService<IAsyncActionInvokerFactory>();
    if (asyncActionInvokerFactory != null)
    {
        return asyncActionInvokerFactory.CreateInstance();
    }
    IActionInvokerFactory actionInvokerFactory = Resolver.GetService<IActionInvokerFactory>();
    if (actionInvokerFactory != null)
    {
        return actionInvokerFactory.CreateInstance();
    }

    // Note that getting a service from the current cache will return the same instance for every request.
    return Resolver.GetService<IAsyncActionInvoker>() ??
        Resolver.GetService<IActionInvoker>() ??
        new AsyncControllerActionInvoker();
}
```

我們解析器一樣使用`Autofac`容器來幫我們完成(程式碼會基於昨天`Autofac`範例往上擴充)

## 建立自己的IActionInvoker(CustomerActionInvoker)

在取得`IActionInvoker`首先會透過`Resolver`解析器來取得,這就提供我們一個可替換接口.

藉由這個機制讓我們可以重寫自己`ActionInvoker`物件.

我們自行撰寫的`CustomerActionInvoker`支援簡單模型綁定(這個版本支援由`Request.Form`和`Request.QueryString`參數綁定)

1. 首先利用反射先取得呼叫`Action`方法資訊,我再呼叫`BindModel`利用`linq`對於`Action`方法需要參數進行動態綁定
2. `BindModel`方法中先判斷目前參數型別是否是字串型別,如果是透過`GetValueTypeInstance`從`ValueProvider`(`Request.Form`和`Request.QueryString`)取值,如果方法使用參數非簡單型別參數就會呼叫`SimpleModelBinding`方法
3. `SimpleModelBinding`利用反射動態建立此物件,取得此物件屬性資訊並一一把值給填充到屬性上.

> 在`SimpleModelBinding`會判斷屬性型別和可否寫入`!property.CanWrite || IsSimpleType(property)`來填值.

```csharp
public class CustomerActionInvoker : IActionInvoker
{
    public bool InvokeAction(ControllerContext controllerContext, string actionName)
    {
        //取得執行Action方法
        MethodInfo method = controllerContext.Controller
            .GetType()
            .GetMethods()
            .First(m => string.Compare(actionName, m.Name, StringComparison.OrdinalIgnoreCase) == 0);

        //取得Action使用的參數,並利用反射將值填充
        var parameters = method.GetParameters().Select(parameter =>
            BindModel(controllerContext, parameter.Name, parameter.ParameterType));

        ActionResult actionResult = method.Invoke(controllerContext.Controller, parameters.ToArray()) as ActionResult;

        actionResult.ExecuteResult(controllerContext);

        return true;
    }

    private object BindModel(ControllerContext controllerContext,string modelName, Type modelType)
    {
        if (modelType.IsValueType || typeof(string) == modelType)
        {
            object instance;
            if (GetValueTypeInstance(controllerContext, modelName, modelType, out instance))
            {
                return instance;
            }
            return Activator.CreateInstance(modelType);
        }

        return SimpleModelBinding(controllerContext, modelType);
    }

    private object SimpleModelBinding(ControllerContext controllerContext, Type modelType)
    {
        object modelInstance = Activator.CreateInstance(modelType);
        foreach (PropertyInfo property in modelType.GetProperties())
        {
           //針對基本型別或string型別給值
            if (!property.CanWrite || IsSimpleType(property))
            {
                object propertyValue;
                if (GetValueTypeInstance(controllerContext, property.Name, property.PropertyType, out propertyValue))
                {
                    property.SetValue(modelInstance, propertyValue);
                }
            }
        }

        return modelInstance;
    }

    private bool GetValueTypeInstance(ControllerContext controllerContext, string modelName, Type modelType, out object value)
    {
        var form = controllerContext.RequestContext.HttpContext.Request.Form;
        var queryString = controllerContext.RequestContext.HttpContext.Request.QueryString;

        string key = form.AllKeys.FirstOrDefault(x => string.Compare(x, modelName, StringComparison.OrdinalIgnoreCase) == 0);
        if (key != null)
        {
            value = Convert.ChangeType(form[key], modelType);
            return true;
        }

        string queryKey = queryString.AllKeys.FirstOrDefault(x => string.Compare(x, modelName, StringComparison.OrdinalIgnoreCase) == 0);
        if (queryKey != null)
        {
            value = Convert.ChangeType(queryString[queryKey], modelType);
            return true;
        }

        value = null;

        return false;
    }

    private static bool IsSimpleType(PropertyInfo property)
    {
        return property.PropertyType == typeof(string) || property.PropertyType.IsValueType;
    }
}
```

最後在`Autofac`中多註冊一組`IActionInvoker`,**MVC**就會使用`CustomerActionInvoker`而不是原本的`ControllerActionInvoker`

```csharp
builder.RegisterType<CustomerActionInvoker>().As<IActionInvoker>();
```

### 進行呼叫測試

我在`HomeController`下新增一個`About`方法傳入一個`Person`類別.

後面請求 `http:xxx/Home/About?name=daniel` 我們就可以看到方法使用`p`參數已經可以成功填值瞜

```csharp
public class Person
{
    public string Name{ get; set; }
}

public ActionResult About(Person p)
{
    ViewBag.Message = $"Member {p?.Name??string.Empty} Balance { _service.GetMemberBalance(123)}";

    return View();
}
```

## 改進GetValueTypeInstance方法(建立ValueProvider)

在`GetValueTypeInstance`方法中透過`Http`上請求獲取資料目前有兩種方式`Request.Form`和`Request.QueryString`,我們可以看到上面的方法有許多重複程式碼

這次要做動作是**重構**把上面重複程式碼提取到一個父類別(長出父類別或介面).

> 我覺得在物件導向程式設計介面和父類別是長出來,寫一寫code發現有重複的部分就可以考慮提取方法或提取成父類別.

首先我們先對於`GetValueTypeInstance`進行分析.

```csharp
private bool GetValueTypeInstance(ControllerContext controllerContext, string modelName, Type modelType, out object value)
{
    var form = controllerContext.RequestContext.HttpContext.Request.Form;
    var queryString = controllerContext.RequestContext.HttpContext.Request.QueryString;

    string key = form.AllKeys.FirstOrDefault(x => string.Compare(x, modelName, StringComparison.OrdinalIgnoreCase) == 0);
    if (key != null)
    {
        value = Convert.ChangeType(form[key], modelType);
        return true;
    }

    string queryKey = queryString.AllKeys.FirstOrDefault(x => string.Compare(x, modelName, StringComparison.OrdinalIgnoreCase) == 0);
    if (queryKey != null)
    {
        value = Convert.ChangeType(queryString[queryKey], modelType);
        return true;
    }

    value = null;

    return false;
}
```

發現到下面這段程式碼基本是重複的除了一個是透過`form`,另一個是透過`queryString`取得比對取得使用`key`.

```csharp
string key = form.AllKeys.FirstOrDefault(x => string.Compare(x, modelName, StringComparison.OrdinalIgnoreCase) == 0);
if (key != null)
{
    value = Convert.ChangeType(form[key], modelType);
    return true;
}
```

> 看到重複動作就可以考慮提取成抽象並把特徵交給子類別來實現或提供.

### 建立一個ValueProviderBase抽象類別

在下面有一個`GetValue`方法我們把上面重複的程式碼放進裡面,提供一個`abstract NameValueCollection nameValueCollection`抽象屬性給自類別提供實現.

> 因為`QueryString`和`Form`都是`NameValueCollection`型態的集合.

```csharp
public abstract class ValueProviderBase
{
    protected ControllerContext _controllerContext;

    public ValueProviderBase(ControllerContext controllerContext)
    {
        _controllerContext = controllerContext;
    }

    protected abstract NameValueCollection nameValueCollection { get; }

    public object GetValue(string modelName,Type modelType)
    {
        string key = nameValueCollection.AllKeys.FirstOrDefault(x => string.Compare(x, modelName, StringComparison.OrdinalIgnoreCase) == 0);

        if (key != null)
        {
            return Convert.ChangeType(nameValueCollection[key], modelType);
        }

        return null;
    }
}
```

建立兩個類別`FormValueProvider`,`QueryStringValueProvider`繼承於`ValueProviderBase`並實現`NameValueCollection`抽象屬性

1. `FormValueProvider`:提供`Request.Form`
2. `QueryStringValueProvider`:提供`Request.QueryString`

```csharp
public class FormValueProvider : ValueProviderBase
{
    public FormValueProvider(ControllerContext controllerContext) : base(controllerContext)
    {
    }

    protected override NameValueCollection nameValueCollection => _controllerContext.RequestContext.HttpContext.Request.Form;
}

public class QueryStringValueProvider : ValueProviderBase
{
    public QueryStringValueProvider(ControllerContext controllerContext) : base(controllerContext)
    {
    }

    protected override NameValueCollection nameValueCollection => _controllerContext.RequestContext.HttpContext.Request.QueryString;
}
```

最後在`GetValueTypeInstance`方法會改寫成

```csharp
private bool GetValueTypeInstance(ControllerContext controllerContext, string modelName, Type modelType, out object value)
{
    List<ValueProviderBase> _valueProvider = new List<ValueProviderBase>()
    {
        new FormValueProvider(controllerContext),
        new QueryStringValueProvider(controllerContext)
    };

    foreach (var valueProvider in _valueProvider)
    {
        value = valueProvider.GetValue(modelName, modelType);
        if (value != null)
            return true;
    }

    value = null;
    return false;
}
```

建立一個列表存放`ValueProvider`集合並使用迴圈來一個一個判斷是否有值匹配到.

> 改寫完後有沒有發覺`GetValueTypeInstance`方法比上面版本更好理解呢?

我把細部邏輯都封裝到類別中,閱讀上也變得更容易.

### 似成相識IValueProvider介面

還記得之前我們有介紹到一個`IValueProvider`介面提供一個重要方法`GetValue`如何從`Http`請求中取得資料藉由傳入`key`.

```csharp
/// <summary>
/// Defines the methods that are required for a value provider in ASP.NET MVC.
/// </summary>
public interface IValueProvider
{
	/// <summary>
	/// Determines whether the collection contains the specified prefix.
	/// </summary>
	bool ContainsPrefix(string prefix);

	/// <summary>
	/// Retrieves a value object using the specified key.
	/// </summary>
	ValueProviderResult GetValue(string key);
}
```

這次重構`IValueProvider`很類似之前介紹的`IValueProvider`介面,上面`List<ValueProviderBase>`就是之前介紹`ValueProviderFactories`工廠.

## 小結:

今天利用一個範例建立**自己的簡單模型綁定ActionInvoker**向大家分享如何建立自己的`ActionInvoker`只需要透過一個`Resolver`解析器和繼承`IActionInvoker`即可完成.

後面再利用重構技巧優化本次程式.希望今天使用到的技巧對於大家有所幫助

> 設計模式不是把程式碼變簡單而是整理得更有條理(程式碼可能會更複雜但卻很合理,更好去理解複雜邏輯)

一個房間很亂經過整理後東西不會變少(排除丟掉東西),但物品位置會變得更有條理

[Github範例程式原始碼](https://github.com/isdaniel/ItHelp_MVC_10th/tree/customerActionInvoker) `customerActionInvoker`分支上