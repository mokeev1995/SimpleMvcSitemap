SimpleMvcSitemap
=============
A minimalist library for creating sitemap files inside ASP.NET Core applications.

SimpleMvcSitemap lets you create [sitemap files](http://www.sitemaps.org/protocol.html) inside action methods without any configuration. 
It also supports generating [sitemap index files](http://www.sitemaps.org/protocol.html#index). 
Since you are using regular action methods you can take advantage of caching and routing available in the framework.

## Table of contents
 - [Requirements](#requirements)
 - [Installation](#installation)
 - [Examples](#examples)
 - [Sitemap Index Files](#sitemap-index-files)
 - [DateTime Format](#datetime-format)
 - [Google Sitemap Extensions](#google-sitemap-extensions)
   - [Images](#images)
   - [Videos](#videos)
   - [News](#news)
   - [Alternate language pages](#translations)
 - [XSL Style Sheets](#style-sheets)
 - [Custom Base URL](#base-url)  
 - [Unit Testing and Dependency Injection](#di)
 - [License](#license)


## <a id="requirements">Requirements</a>
 - .Net Core 3.1 and newer
 - ASP.NET Core 3.1 and newer

## <a id="installation">Installation</a>

### .Net Core

Install the [NuGet package](https://www.nuget.org/packages/SimpleMvcSitemap/) on your MVC project.

```powershell
Install-Package SimpleMvcSitemap
```

Add to DI Container

```csharp
public class Startup
{
    // ...
    public void ConfigureServices(IServiceCollection services)
    {
        // ...
        services.AddSingleton<ISitemapProvider, SitemapProvider>();
        // ...
    }
    // ...
}
```

### .NET Framework

Support for .NET Framework and ASP.NET MVC has been dropped by version 4. Use [version 3](https://github.com/uhaciogullari/SimpleMvcSitemap/tree/v3) if you need to support this scenario.

## <a id="examples">Examples</a>

You can use SitemapProvider class to create sitemap files inside any action method. You don't even have to provide absolute URLs, SimpleMvcSitemap can generate them from relative URLs. Here's an example:
```csharp
public class SitemapController : Controller
{
    private readonly ISitemapProvider _sitemapProvider;

    public SitemapController(ISitemapProvider sitemapProvider)
    {
        _sitemapProvider = sitemapProvider;
    }

    public ActionResult Index()
    {
        List<SitemapNode> nodes = new List<SitemapNode>
        {
            new SitemapNode(Url.Action("Index","Home")),
            new SitemapNode(Url.Action("About","Home")),
            //other nodes
        };

        return _sitemapProvider.CreateSitemap(new SitemapModel(nodes));
    }
}
```

SitemapNode class also lets you specify the [optional attributes](http://www.sitemaps.org/protocol.html#xmlTagDefinitions):
```csharp
new SitemapNode(Url.Action("Index", "Home"))
{
    ChangeFrequency = ChangeFrequency.Weekly,
    LastModificationDate = DateTime.UtcNow,
    Priority = 0.8M
}
```	

## <a id="sitemap-index-files">Sitemap Index Files</a>

Sitemap files must have no more than 50,000 URLs and must be no larger then 10MB [as stated in the protocol](http://www.sitemaps.org/protocol.html#index). If you think your sitemap file can exceed these limits you should create a sitemap index file. If you have a logical seperation, you can create an index manually.

 ```csharp
List<SitemapIndexNode> sitemapIndexNodes = new List<SitemapIndexNode>
{
    new SitemapIndexNode(Url.Action("Categories","Sitemap")),
    new SitemapIndexNode(Url.Action("Products","Sitemap"))
};

return new SitemapProvider().CreateSitemapIndex(new SitemapIndexModel(sitemapIndexNodes));
```

If you are dealing with dynamic data and you are retrieving the data using a LINQ provider, SimpleMvcSitemap can handle the paging for you. A regular sitemap will be created if you don't have more nodes than the sitemap size.

![Generating sitemap index files](http://i.imgur.com/ZJ7UNkM.png)

This requires a little configuration:

```csharp
public class ProductSitemapIndexConfiguration : SitemapIndexConfiguration<Product>
{
    private readonly IUrlHelper urlHelper;

    public ProductSitemapIndexConfiguration(IQueryable<Product> dataSource, int? currentPage, IUrlHelper urlHelper)
        : base(dataSource,currentPage)
    {
        this.urlHelper = urlHelper;
    }

    public override SitemapIndexNode CreateSitemapIndexNode(int currentPage)
    {
        return new SitemapIndexNode(urlHelper.RouteUrl("ProductSitemap", new { currentPage }));
    }

    public override SitemapNode CreateNode(Product source)
    {
        return new SitemapNode(urlHelper.RouteUrl("Product", new { id = source.Id }));
    }
}
```
Then you can create the sitemap file or the index file within a single action method.

```csharp
public ActionResult Products(int? currentPage)
{
    var dataSource = products.Where(item => item.Status == ProductStatus.Active);
    var productSitemapIndexConfiguration = new ProductSitemapIndexConfiguration(dataSource, currentPage, Url);
    return new DynamicSitemapIndexProvider().CreateSitemapIndex(new SitemapProvider(), productSitemapIndexConfiguration);
}
```

## <a id="datetime-format">DateTime Format</a>

You should convert your DateTime values to local time. Universal time format generated by .NET [is not accepted by Google](https://github.com/uhaciogullari/SimpleMvcSitemap/issues/16). You can use [.ToLocalTime()](https://docs.microsoft.com/en-us/dotnet/api/system.datetime.tolocaltime) method to do the conversion.

## <a id="google-sitemap-extensions">Google Sitemap Extensions</a>

You can use [Google's sitemap extensions](https://support.google.com/webmasters/topic/6080646?hl=en&ref_topic=4581190) to provide detailed information about specific content types like [images](https://support.google.com/webmasters/answer/178636), [videos](https://support.google.com/webmasters/answer/80471), [news](https://support.google.com/news/publisher/answer/74288?hl=en&ref_topic=4359874) or [alternate language pages](https://support.google.com/webmasters/answer/2620865). You can still use relative URLs for any of the additional URLs.

### <a id="images">Images</a>

```csharp
using SimpleMvcSitemap.Images;

new SitemapNode(Url.Action("Display", "Product"))
{
    Images = new List<SitemapImage>
    {
        new SitemapImage(Url.Action("Image","Product", new {id = 1})),
        new SitemapImage(Url.Action("Image","Product", new {id = 2}))
    }
}
```

### <a id="videos">Videos</a>

By version 4, multiple videos are supported. Start using Videos property if you are upgrading from v3 to v4.

```csharp
using SimpleMvcSitemap.Videos;

new SitemapNode("http://www.example.com/videos/some_video_landing_page.html")
{
    Videos = new List<SitemapVideo>
    { 
        new SitemapVideo(title: "Grilling steaks for summer",
                         description: "Alkis shows you how to get perfectly done steaks every time",
                         thumbnailUrl: "http://www.example.com/thumbs/123.jpg", 
                         contentUrl: "http://www.example.com/video123.flv")
    }
}
```

### <a id="news">News</a>

```csharp
using SimpleMvcSitemap.News;

new SitemapNode("http://www.example.org/business/article55.html")
{
    News = new SitemapNews(newsPublication: new NewsPublication(name: "The Example Times", language: "en"),
                           publicationDate: new DateTime(2014, 11, 5, 0, 0, 0, DateTimeKind.Utc),
                           title: "Companies A, B in Merger Talks")
}
```

### <a id="translations">Alternate language pages</a>

```csharp
using SimpleMvcSitemap.Translations;

new SitemapNode("abc")
{
    Translations = new List<SitemapPageTranslation>
    {
        new SitemapPageTranslation("http://www.example.com/deutsch/", "de"),
		new SitemapPageTranslation("http://www.example.com/english/", "en")
    }
}
```

## <a id="style-sheets">XSL Style Sheets</a>

SimpleMvcSitemap supports XSL style sheets by version 3. Keep in mind that XML stylesheets are subjected to the [same origin](https://en.wikipedia.org/wiki/Same-origin_policy) checks.

```csharp
using SimpleMvcSitemap.StyleSheets;

new SitemapModel(new List<SitemapNode> { new SitemapNode("abc") })
{
    StyleSheets = new List<XmlStyleSheet>
    {
        new XmlStyleSheet("/sitemap.xsl")
    }
};
```
You can see how you can utilize multiple XSL style sheets in [this tutorial](http://www.ibm.com/developerworks/library/x-tipstyl/).

## <a id="base-url">Custom Base URL</a>

SimpleMvcSitemap can generate absolute URLs from the relative URLs using the HTTP request context. If you want to customize this behaviour, you can implement IBaseUrlProvider interface and pass it to the SitemapProvider class.

```csharp
public class BaseUrlProvider : IBaseUrlProvider
{
    public Uri BaseUrl => new Uri("http://example.com");
}

var sitemapProvider = new SitemapProvider(new BaseUrlProvider());
```


## <a id="di">Unit Testing and Dependency Injection</a>

SitemapProvider class implements the ISitemapProvider interface which can be injected to your controllers and be replaced with test doubles. All methods are thread safe so they can be used with singleton life cycle.
```csharp
public class SitemapController : Controller
{
    private readonly ISitemapProvider _sitemapProvider;

    public SitemapController(ISitemapProvider sitemapProvider)
    {
        _sitemapProvider = sitemapProvider;
    }
	
    //action methods
}
```

## <a id="license">License</a>

SimpleMvcSitemap is licensed under [MIT License](http://opensource.org/licenses/MIT "Read more about the MIT license form"). Refer to license file for more information.
