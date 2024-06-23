## convert to SDK style if necessary

* tool: https://github.com/hvanbakel/CsprojToVs2017
* note: need to specify `<Project Sdk="Microsoft.NET.Sdk.Web">` to implicitly use `AspNetCore` reference ([.NET project sdks overview](https://docs.microsoft.com/en-us/dotnet/core/project-sdk/overview))

## change target framework to net6 in `*.csproj` files

* `HttpApplication` in `Global.asax ` cannot be used after this change, then need to transform the project startup mechanics from `Global.asax` to `Startup` ([app startup in .NET5](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup?view=aspnetcore-5.0) and [app startup in .NET6](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/startup?view=aspnetcore-6.0))
  
  * some projects use Autofac as IoC container 
    * [Autofac startup example](https://autofac.readthedocs.io/en/latest/integration/aspnetcore.html#startup-class)
    * add `.UseServiceProviderFactory(new AutofacServiceProviderFactory())` in `Program.cs`
    * some registration are [named service](https://autofac.readthedocs.io/en/latest/advanced/keyed-services.html#named-and-keyed-services) and can be resolved with `[KeyFilter]` attributes
    * clean up obsolete code ([difference from ASP.NET classic using Autofac](https://autofac.readthedocs.io/en/latest/integration/aspnetcore.html#differences-from-asp-net-classic))
  * [exception filter](https://docs.microsoft.com/en-us/aspnet/core/mvc/controllers/filters?view=aspnetcore-6.0)
    * change inheritance from `IExceptionFilter` under `Microsoft.AspNetCore.Mvc.Filters` namespace
    * add exception filters as options into `.AddControllers()`
  * pass JSON serializer settings into `.AddNewtonsoftJson()`
  * [migrate message handlers and modules to middleware](https://docs.microsoft.com/en-us/aspnet/core/migration/http-modules?view=aspnetcore-6.0)
    * migrate customized [message handlers](https://docs.microsoft.com/en-us/aspnet/web-api/overview/advanced/http-message-handlers) to middlewares
    * migrate [application events](https://docs.microsoft.com/en-us/dotnet/api/system.web.httpapplication?view=netframework-4.8) such as `Application_BeginRequest`, `Application_EndRequest` to middlewares
  * [routing](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-6.0)
    * no need to call `UseRouting` or `UseEndpoints` because `WebApplicationBuilder`configures a middleware pipeline between `UseRouting` and `UseEndpoints`, unless you'd like to modify middleware ordering(e.g. not between routing and endpoints)
    * [routing difference between ASP.NET MVC and ASP.NET Core](https://docs.microsoft.com/en-us/dotnet/architecture/porting-existing-aspnet-apps/routing-differences)
    * inside controller
      * inherit from `ApiController` to `ControllerBase` and add `[ApiController]` attribute on the class
      * remove `[FromUri]` or change `[FromUri]` to `[FromQuery]` and add `[FromBody]` on request body
      * change return type to `IActionResult` which returns `Ok()`, `NotFound()`, etc
    * some projects use `RouteConfig` file to configure routes, and this file can be reused
      * add `app.UseEndpoints(RouteConfig.RegisterRoutes)` in `Startup` and change `RouteConfig` to use `IEndpointRouteBuilder` and `MapContollerRoute`
    * Why remove `StringOutputFormatter`? when it returns BadRequest("errmsg"), it can be encoded JSON string instead of raw string. After remove `StringOutputFormatter`, the default formatter would be JSON formatter.
    * Add `AddXmlSerializerFormatters` can formate response to xml when client send `Accept: application/xml` header.
    * Consider checking BadRequest, Forbidden, InternalServerError response before migration, use `ActionFilter` or middleware to rewrite response after migration. e.g. before migration, bad request will be wrapped in a object, `{"message": "err msg"}`, after migration, it could become `"err msg"`.
  
  * use `MapHealthChecks` method provided by `AspNetCore` instead of health controller 
  
    * health check reponse template
  
      ```json
      {
          "name": "The Service Name",
          "version": "1.0.0-20220704-0908",
          "status": "Success",
          "results": [
              {
                  "name": "sqlserver",
                  "status": "Success",
                  "description": null
              }
          ]
      }
      ```
  
  * use `Swashbuckle.AspNetCore` package for swagger
  
    * add `services.AddSwaggerGen()`  in `ConfigureServices` method and `app.UseSwagger()` `app.UseSwaggerUI()` in `Configure` method
    * [add `launchSettings.json` and specify `ASPNETCORE_ENVIRONMENT` as development](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/environments?view=aspnetcore-6.0)
    * delete `SwaggerConfig` file

### Configuration
* use .NET Core configuration
* use `appSettings.json` for development app configuration
* TODO: non-development environment add KeyValut configuration source
* Create application configuration class for better readabililty, e.g. use `TheServiceConfig.DBConnectionString` instead of `Configuration["DBConnStr"]`
* Register application configuration instance and bind with .NET Core configuration

### [HttpClient](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-6.0)

Instead of registering the HttpClient class, register [typed HttpClient](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-6.0#typed-clients) by calling `services.AddHttpClient<>()` and provide configuration such as base address in the typed clients constructor

#### chunked request to external service
If you need `JsonContent`, consider using `await theJsonContent.LoadIntoBufferAsync()` before sending request, this call will buffer json content, because it's buffered, the content length can be determined. The object is required `getter` and `setter` otherwise, only empty object will be loaded into buffer, that means content length is incorrect.

Another option is to use `StringContent` and specify encoding, mediaType
```csharp
new StringContent(JsonConvert.SerializeObject(request), Encoding.UTF8, MediaTypeNames.Application.Json);
```

### HttpClientFactory

#### HttpClient issue

The implementation of HttpClient in .NET Core based on SocketHttpHandler, which will keep DNS cache, see DNS behavior [here](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines), you should consider whether DNS cache is an issue or not, it depends on projects.

#### Use factory

HttpClientFactory creates short-lived HttpClient, which is not suffer DNS cache issue.

We recommended Typed http client regisration, because it's suitable with our client implementation.

With Typed http client, in `JobDispatcherProviderClient`, you can request a `HttpClient` object in constructor, it will be created and managed by DI.


```csharp
services.AddHttpClient<IJobDispatcherProvider, JobDispatcherProviderClient>();
```


The interface is used for testing, e.g.

```csharp
var mockedJobDispatcherProvider = new Mock<IJobDispatcherProvider>();
services.AddTransient(typeof(IJobDispatcherProvider), x => mockedJobDispatcherProvider.Object);
```

Reference:

* https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines

### Web.config template

**Notice:** disable web config transformation when you run `dotnet publish`, you can add args in dotnet cli or add to `csproj` file, e.g. `dotnet publish /p:IsWebConfigTransformDisabled=true`



use web.config to remove sensitive headers:

* Server
* X-Powered-By
* X-AspNet-Version

This is a sample web.config file, modify to your service
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.web>
      <httpRuntime requestValidationMode="2.0" enableVersionHeader="false"/>
    </system.web>
    <system.webServer>
      <httpProtocol>
        <customHeaders>
          <remove name="X-Powered-By"/>
        </customHeaders>
      </httpProtocol>
      <security>
        <requestFiltering removeServerHeader="true" />
      </security>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified" />
      </handlers>
      <aspNetCore processPath="dotnet" arguments=".\TheService.dll" stdoutLogEnabled="false" stdoutLogFile=".\logs\stdout" hostingModel="inprocess" />
    </system.webServer>
  </location>
</configuration>
```
### Web config transformation

official docs: https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/transform-webconfig?view=aspnetcore-6.0

useful case(same with us): https://dotnetthoughts.net/asp-net-core-web-config-transform-for-production/

### NLog.config template

integrate NLog with ASP.NET Core: https://github.com/NLog/NLog/wiki/Getting-started-with-ASP.NET-Core-6

missing targets and rules in existing Nlog.config
```
<target xsi:type="Console" name="lifetimeConsole" layout="${MicrosoftConsoleLayout}" />
```
and 
```
<logger name="Microsoft.Hosting.Lifetime" minlevel="Info" writeTo="lifetimeConsole, ownFile-web" final="true" />

<!--Skip non-critical Microsoft logs and so log only own logs (BlackHole) -->
<logger name="Microsoft.*" maxlevel="Info" final="true" />
<logger name="System.Net.Http.*" maxlevel="Info" final="true" />
```

### System.Web related functionalities
`System.Web.*` should be removed during migration. After deleted `System.Web`, you could fins some functionalities are failed.

Suggestion:
  * Install [analyzer nuget package](https://www.nuget.org/packages/Microsoft.DotNet.UpgradeAssistant.Extensions.Default.Analyzers/), build solution, the analyzer will warn you what you should migrate, e.g. if you reference System.Web, you'll get a warn, so keep warn as error could be a good idea.

#### UrlEncode/UrlDecode
Use `Uri.EscapeDataString` and `Uri.UnescapeDataString`

#### Query string
```csharp
var queryStrings = HttpUtility.ParseQueryString(uriBuilder.Query);
queryStrings[parameterName] = value;
uriBuilder.Query = queryStrings.ToString();
```
should replace with `QueryHelpers` and `QueryString`
```csharp
var queryStrings = QueryHelpers.ParseQuery(uriBuilder.Query);
if (queryStrings.ContainsKey(parameterName))
{
    queryStrings[parameterName] = value;
}
else
{
    queryStrings.Add(parameterName, value);
}
uriBuilder.Query = QueryString.Create(queryStrings).ToString();
```

## upgrade Nuget packages or remove indirectly-used Nuget packages
* remove unused and indirectly-used packages
  * in most cases, if the package is used can be checked by searching the `using ${PackageName}` keyword, but also can be checked by whether the removal of the package cause a build fail
  * Reference nuget packages checklist: https://docs.google.com/spreadsheets/d/1YFbxf_WFuHfcAwcifbZrp8mefZVGFCzhPkby3F7nevQ/edit#gid=0

* upgrade third party packages
  * make sure the package supports any version of netstandard or net6, if not, upgrade the package or find replacement to serve the same functionality
  * note: the packages' dependencies may have version conflict
* upgrade self-managed packages
  * the target framework for self-managed packages should be netstandard 2.0
  * may need to add pipeline or pipeline stages for some packages that lack recent maintainance
  * the compability of the packages' dependencies version needs to be considered as well, so the latest version are normally not recomended

## DB migration project
### Fluent Migration
Use version: 3.2.15 since deployment pipeline uses Fluent Migration Console 3.2.15 to run db migration.

### Target framework
Use .NET Standard 2.0, since fluent migration console does not support .NET5/6 now, reference: https://github.com/fluentmigrator/fluentmigrator/issues/1544

### Include fluent migration console?

#### Why some migration projects included the console tools?
myData UT pipeline use the console tools to run migration, deployment pipeline does not use that console tools.

#### How to include?
Consider the following way to include fluent migration console, this will add `tools` to output directory, if not, do not need to include the console tools.

1. install fluent migration console 3.2.15 nuget package
2. csproj file
```xml
 <ItemGroup>
    <Content Include="$(PkgFluentMigrator_Console)\tools\net461\any\**" LinkBase="tools" CopyToOutputDirectory="PreserveNewest" />
  </ItemGroup>
```
### Other modification
1. migration project powershell script for pipeline.
2. myData UT pipeline script, it could download the migration project to build ut db.

## ApplicationName and ApplicationVersion

### Display application name and version
When you access `healthcheck` api, you will get application name, version, dependencies state.

### How to record application name, verison?
Recommended use `app.config` for .NET Core application.

### Migration from web.config

1. create app.config
2. move configurtion part(e.g. appsettings) to app.config, keep same node name and level
3. application name is used for retrieving config from azure keyvalue, so please keeep same as before.
4. In pipeline script, it could update application version to pipeline build number, make sure it update `app.config`

## Single Page Application
### Example(partial)

home page is : `/` or `/index.html`, it returns html content

front-end uses hash: `/#/main`, `func`

back-end provides APIs

Setup webroot since it's null by default
```csharp
var builder = WebApplication.CreateBuilder(new WebApplicationOptions
{
    Args = args,
    WebRootPath = webRoot
});
```
**Notice**: the args in the parameter in your Main function, why pass it to this build is because it is useful when setup WebApplicationFactory for testing, it use args to pass `ContentRootPath`, if this path is not correct, all your requests in testing could be 404.

Add two middlewares, these should keep such ordering, and before `app.UseRouting()`

```csharp
app.UseDefaultFiles();
app.UseStaticFiles();
```

Put the `index.html` file under `webroot`, the static file should be returned, and the middleware terminates, no requests hit the back-end.

### Example(Full)
Any routes, maybe except `/api/`, returns html content, front-end then routing.
Please reference myId codebase, not migrated to dotnet 565 

## convert test project

Targets:
1. run in parallel
2. less memory usage

### Run in parallel
remove all `static`

### less memory usage
Consider xunit `class fixture` or `collection fixture`

### test database
  * Nhibernate 5.3.0 change the date time type from datetime to text, therefore, the sqlite connection string need to be modified
  * Read as form data: `FormDataCollection` is not available in .NET Core, use `new FormReader(await request.Content.ReadAsStringAsync()).ReadForm()` as replacement.

### testing background service/hosted service
  * no need to use `WebApplicationFactory<Program>`
  * example fact base class for Consent pdf worker
  ```csharp
  public class WorkerFactBase: TestBase
  {
    private readonly ServiceProvider serviceProvider;

    public WorkerFactBase()
    {
        IServiceCollection serviceCollection = new ServiceCollection();
        serviceCollection.ConfigureWorkerServices(); // register service via Microsoft DI
        serviceCollection.AddScoped<MockHttpServer>();
        serviceCollection.AddSingleton(sp => new HttpClient(sp.GetService<MockHttpServer>()));
        serviceProvider = serviceCollection.BuildServiceProvider();
    }

    protected T GetService<T>()
    {
        return serviceProvider.GetService<T>();
    }
  }
  ```

## Migrate background service
### Native

```csharp
services.AddHostedService<Worker>();
```

The `Worker` implements `BackgroundService`, the main work is in `ExecuteAsync` function.

### Quartz.NET

You can use Quartz hosted service

```csharp
services.AddQuartz(config =>
{
    config.UseMicrosoftDependencyInjectionJobFactory();
    config.AddTrigger...
    config.AddTrigger...
});
services.AddQuartzHostedService();
```

### Service + Web

use web application builder, add hosted service, run with url

```csharp
var builder = WebApplication.CreateBuilder(webApplicationOptions);

builder.ConfigureServices(services => {
    services.AddHostedService<MyPwCWorkerEntry>();    
});

var app = builder.Build();

app.Run("http://localhost:9098");
```



## Request and Response in .NET Core
The body of Request and Response is a special stream, it can only be read once, if you need to read them multiple times, you can do like this.

### Request body

#### use case: proxy the request to other service

Write a attr implements `IResultFilter`, enable buferring, add attr to the action you need, e.g.
```csharp
public class EnableRequestBodyBufferingAttribute: Attribute, IResourceFilter
{
    public void OnResourceExecuting(ResourceExecutingContext context)
    {
        // enable buffering means the request body can be read multiple times
        context.HttpContext.Request.EnableBuffering();
    }

    public void OnResourceExecuted(ResourceExecutedContext context)
    {
        
    }
}
```

#### use case: model binder
model binder will not be affected since the request body will not be read before the model binder execution.

### Response body

#### use case: record response

Please reference "append JSON hijaking prefix", just ignore modification for response body.

#### use case: append JSON hijaking prefix(chars, `)]}`)

Replace response body before invoking next middleware, do your modification, then replace the stream back.

```csharp
public class CustomJsonResponseHandler : IMiddleware
{
    public const string PREFIX = ")]}',";

    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        var originalBody = context.Response.Body;
        var newBody = new MemoryStream();
        context.Response.Body = newBody;
        
        await next.Invoke(context);
        
        var httpResponseMessage = context.Response;
        if (!string.IsNullOrEmpty(httpResponseMessage.ContentType) && new ContentType(httpResponseMessage.ContentType).MediaType == MediaTypeNames.Application.Json)
        {
            newBody.Seek(0, SeekOrigin.Begin);
            var content = await new StreamReader(newBody).ReadToEndAsync();

            if (content != SiteAssociation.JsonContent)
            {
                context.Response.ContentLength += Encoding.UTF8.GetByteCount(PREFIX);
                await (await new StringContent(PREFIX).ReadAsStreamAsync()).CopyToAsync(originalBody);
            }
        }

        newBody.Seek(0, SeekOrigin.Begin);
        await newBody.CopyToAsync(originalBody);
    }
}
```

### AllowEmptyInputInBodyModelBinding
By default, `false`, it does not means empty request body will raise an exception. There is a common mistake, you get a http status code 415 when you don't provide request body.

The 415 status code is returned by UnsupportContentTypeFilter, this filter only cares UnsupportedContentTypeException, and this exception is raised when you do not provide Content-Type header.

When you provided Content-Type, let's say application/json, but no content(a.k.a content-length is 0):

* AllowEmptyInputInBodyModelBinding=true, you **will not** get a model binder error(check model state dict), you will get null for complex type
* AllowEmptyInputInBodyModelBinding=false, you **will** get a model binder error(check model state dict), you will get null for complex type

## Data Protection API

We introduced Data Protection API in some projects, because we used antiforgery library in .NET Core. Antiforgery depends on Data Protection API.

Configure Data Protection API requires two things:

* where to store encryption key
* how to protect the encryption key

Microsoft provides some kinds of implementations, e.g. store on Azure Storage Account, and protect it with Azure KeyVault Key. It can be customized where to store and how to protect, just implements the interfaces.

## Recommended articles
.NET Core 6 fundamentals: https://docs.microsoft.com/en-us/aspnet/core/fundamentals/?view=aspnetcore-6.0&tabs=windows

Explore .NET6: https://andrewlock.net/series/exploring-dotnet-6/



![BrandImg](https://picx.zhimg.com/v2-1dd6e1ed2f348db47ce0cde38d545ae9_r.webp?source=172ae18b&consumer=ZHI_MENG)
