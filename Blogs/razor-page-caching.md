---
blurb: How to implement Response and In Memory cache in your ASP.NET core Razor Pages application to dramatically improve performance.
title: Razor Page Caching
path: razor-page-caching
tags:
    - Caching
    - Dotnet
    - ASP.NETCores
    - Razor
published: 2021-11-21
length: 6
---

I previously posted about the [performance](./reengineering-my-blog){target="__blank"} of my website and wanted to take the next logical step, caching. Perusing the ASP.NET documentation revealed two methods of caching useful for my website, 'Response' and 'In Memory' caching.

## Response Caching

Response Caching sets the `Cache-Control` of the response and configures the `max-age` property. This property tells the client it does not need to fetch the page again until this timeout has expired. The page is instead fetched from the browser cache.

One gotcha I encountered whilst testing  this  is the 'refresh' button ignores the `max-age` property, this is intended behaviour  and if you think about it is  exactly what you are asking  for.  Testing therefore only requires revisiting  the page(s) and navigating around the site.

Add the following to your configure services in `startup.cs`

```c#
public void ConfigureServices(IServiceCollection services)
{
    ....
    services.AddResponseCaching();
    ...
}
```

Configure the Response Cache by adding an annotation to the top of your page model. In my case I also  needed to use `VaryByQueryKeys` otherwise the browser will not know it needs to request the page for different search and page numbers.

```c#
[ResponseCache(Duration = 60 * 60 * 3, VaryByQueryKeys = new[] { "Search", "PageNum" })]
public class BlogListModel : PageModel
{
    ...
}
```

## In Memory Cache

My website content is read-only  and mostly static. There is also no variation between viewers of a particular blog post. The highest computation when viewing a blog post is the translation from markdown into html. While the library I am using is quite fast, caching the result is a solid optimisation. An In-Memory cache will take the computed html (assuming it is already in cache) and send it to other users who request the same blog thereby increasing the speed of the response substantially and reducing server load.

Add the following to the `startup.cs` file. The `MemoryCacheService` will be a singleton service required for the lifetime of the application.

```c#
public void ConfigureServices(IServiceCollection services)
{
    ...
    services.AddMemoryCache();
    services.AddSingleton<IMemoryCacheService, MemoryCacheService>();
    ...
}

```

Create the `IMemoryCache` Interface.

```c#
using Microsoft.Extensions.Caching.Memory;

namespace PersonalBlog.Services
{
    public interface IMemoryCacheService
    {
        MemoryCache Cache { get; }
    }
}
```

The implementation of the `MemoryCacheService` is below. The 'timespan' and 'SizeLimit' are two important properties which must be set. Incorrect setup of these options can lead to unusual side effects, consult the [docs](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/memory?view=aspnetcore-5.0#use-setsize-size-and-sizelimit-to-limit-cache-size-1){target="__blank"} accordingly.

```c#
using Microsoft.Extensions.Caching.Memory;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

namespace PersonalBlog.Services
{
    public class MemoryCacheService : IMemoryCacheService
    {
        public MemoryCache Cache { get; private set; }
        public MemoryCacheService()
        {
            var a = new MemoryCacheEntryOptions();
            a.SetSlidingExpiration(TimeSpan.FromHours(6));
            Cache = new MemoryCache(new MemoryCacheOptions
            {
                SizeLimit = 1024,
            });
        }
    }
}
```

'Response Caching' configured previously was set to 3 hours and I set 'In Memory Cache'  to 6 hours, because the Server knows better than the client. This setup allows the server to give the most up to date information to the client. Pages are updated no more than once per day so this configuration suits my use case.

Caching logic is applied to the `OnGet` method of the page model. If the blog is already in the cache, retrieve and  use it, otherwise perform the full process to read the file from the file system and store in cache at the end.

```c#
        public async Task<IActionResult> OnGetAsync(string? postTitle)
        {
            if (postTitle == null)
            {
                return NotFound();
            }
            // Attempt to retrieve value from cache.
             if (!_cache.TryGetValue(postTitle, out BlogPost cachedBlog))
             {
                var filePath = $"Content/Blogs/{postTitle}.md";
                BlogPost  cachedBlog = new BlogPost();
                if (!System.IO.File.Exists(filePath))
                {
                    return RedirectToPage("Error404");
                }

                var markdownFile = await System.IO.File.ReadAllTextAsync(filePath);
                var post = new MarkDownDocBuilder<BlogPost>(markdownFile)
                    .WithYamlData()
                    .WithContent()
                    .Build();

                cachedBlog = post.YamlData;

                if (cachedBlog == null)
                {
                    return RedirectToPage("Error404");
                }
                var cacheEntryOptions = new MemoryCacheEntryOptions()
                        .SetSize(1);

                cachedBlog.Content = post.Document;

                 // Add Blog to Cache.
                _cache.Set(postTitle, cachedBlog, cacheEntryOptions);
             }
            Blog = cachedBlog;
            return Page();
        }
```

I used the In Memory cache for the blog page, however I did not use it for the 'Blog List' page because it becomes messy. I need to deal with the extra complexity in terms of pagination, search and sharing a subset of the blog list component on the home page. I did not have the a solid answer to the following questions and decided to leave the page out of the cache.

* Do I setup another cache for the home page blogs subset?
* Do I perform search queries on every blog and cache the result?
* Do I perform the query on blogs already in the cache?

With these improvements my website has <1ms load time and server resources are substantially optimised. You can find the source code on my [GitHub](https://github.com/kaelanhr/Personal-Blog){target="__blank"}.

For further information on caching I would recommend checking out the [official ASP.NET documentation](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/response){target="__blank"}.
