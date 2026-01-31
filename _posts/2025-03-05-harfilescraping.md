---
title: Makeshift Scraping with HAR Files
date: 2025-05-03 00:00:00 +0000 # YYYY-MM-DD
categories: [Software Engineering]
tags: [scraping, c#]
media_subpath : /assets/img
---
![A cartoonish diagram showing the HAR scraping process described in this article](harscraping.png)

When building the [Tasmania LAN party photo archive](/posts/lanphotosarchive/) last year all of the files were in a small private Facebook group. There wasn't many albums, but each album had hundreds of photos. I searched the internet for scraping solutions but due to large companies like Facebook changing their interface so often nothing open source was up to date, and other options want your credit card info..

I knew I'd have to hack something together myself. To start with, I knew Facebook's APIs are extremely complicated & obfuscated so I ruled that option out pretty quickly. I then considered using a browser UI testing/automation tool like [Puppeteer](https://pptr.dev/) or [Playwright](https://playwright.dev/), but Facebook obfuscates the HTML DOM so much that I knew this would be difficult & slow, and I would likely wish to use this tool again in the future so I wouldn't be happy if it was broken by one of Facebook's monthly redesigns.

I came across a Reddit comment suggesting flicking through the whole album then saving the webpage, which would then contain all the images. I tried it but sadly the saved webpage only outputs the assets referenced by the current DOM, so the only photo from the album saved was the one I was viewing at the end.

But then, the idea came! 💡

# The Idea
What we need is very simple; we need to know the URL of each image, once we know that, we can download them all!

You may have heard of HAR files previously, this is an abbreviation of *HTTP Archive*. Typically created from a web browser, they store a record of web requests in order, keeping track of all related data such as IP addresses, timings, caching, response/request format, body, headers, status codes.. They're great for getting users to record a problematic interaction and sending over for inspection, though you must be careful with who you send HAR files to; as they contain everything, this includes authentication data! There are multiple tools available online to inspect & view HAR files, or you can load them into the devtools of your browser, right next to where you would export one.

![A HAR file viewed with http://www.softwareishard.com/har/viewer/](harscraping_viewer.png)
_A HAR file viewed with the <i>softwareishard</i> tool._

To get what we need in the HAR file we just need to:
1. Go to the photo album.
2. Open the browser devtools network tab, make sure it's recording.
3. Open the first photo in the album.
4. Hold the right arrow key of the keyboard to flick through every photo (no need to wait for each photo to load).
5. Once the end is reached, export the HAR file.

![A screenshot of the Chrome devtools network tab after flicking through a Facebook photo album.](harscraping_devtools.png)
_571 requests after going through an album with 35 photos.. No wonder Facebook feels so sluggish._

Now after exporting we have a 40 megabyte HAR file with record of all web requests. The next step is to fish out all of the URLs of the photo album's images.
Inside HAR files is just (very long) JSON, I was initially planning to get the URLs I needed by crafting a Regex, but a quick Google revealed a .NET library *[HarSharp](https://github.com/giacomelli/HarSharp)* which contains types for the HAR JSON.

Start by inspecting the HAR file to find the urls you want, and figure out what is common across all of them. Obviously websites load many images, so you need to filter it to just those you're interested in. For me in this case filtering to URLS where the path contains `.jpg?_nc_cat=` did the trick.

# The Implemenation

All we need is a little console app.. 

1. Load & deserialize the HAR file we saved.
2. Fish out & filter the urls.
3. Download them all (in parallel for speed).

Here's a stripped back explanation:
```csharp
var har = HarConvert.DeserializeFromFile(harPath);

var urls = har.Log.Entries
    .Where(e => e.Request.Url.AbsoluteUri.Contains(".jpg?_nc_cat="))
    .Select(e => e.Request.Url)
    .DistinctBy(url => url.AbsolutePath)
    .ToList();

await Parallel.ForEachAsync(urls, parallelOptions, async (url, cancellationToken) =>
{
    var path = Path.Combine(destinationPath, Path.GetFileName(url.AbsolutePath));
    var bytes = await httpClient.GetByteArrayAsync(url, cancellationToken);
    await File.WriteAllBytesAsync(path, bytes, cancellationToken);
});
```

Obviously this isn't the fastest or cleanest solution, but building a custom scraper for every site you need one for can get time consuming quick... This is a makeshift method to use in a pinch - like the time I used a dress tie as a fanbelt, get the job done! 😊

I always save code on GitHub now in case I need it later, and as it contains nothing personal I've made this one public. Feel free to use it as a reference, or clone it to customize to your current situation! [github.com/Issung/HarFileScrapeTemplate](https://github.com/Issung/HarFileScrapeTemplate).

Have you ever slapped anything together in a pinch? I love these kinds of stories and I think most people do. It's a bit like MacGyver 😅