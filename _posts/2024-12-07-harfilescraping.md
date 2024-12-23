---
title: Makeshift Scraping with HAR Files
date: 2024-12-07 00:00:00 +0000
categories: [Software Engineering]
tags: [scraping, c#]
img_path: /assets/img
---
I recently had to scrape a few photo albums from facebook, there wasn't many albums, but each album had hundreds of photos. I may not be able to fix a car or build a house, but this I can find a solution for!

To start with, I knew Facebook's APIs are extremely complicated & obfuscated so I ruled that option out pretty quickly. I then considered using a browser UI testing/automation tool like [Puppeteer](https://pptr.dev/) or [Playwright](https://playwright.dev/), but Facebook even obfuscates the HTML DOM so much that I knew this would be difficult, and I would likely wish to use this tool again in the future so I wouldn't be happy if it was broken by one of Facebook's monthly redesigns.

I came across a Reddit comment suggesting flicking through the whole album then saving the webpage, which would then contain all the images. I thought this sounded nice and simple so I tried it but sadly the saved webpage only outputs the assets referenced by the current DOM, so the only photo from the album saved was the one I was viewing at the end.

But then, the idea came! 💡

# The Idea
What we need is very simple; we need to know the URL of each image, once we know that, we can download them all!

You may have heard of HAR files previously, this is an abbreviation of *HTTP Archive*. They store a record of web requests in order, keeping track of all related data such as IP addresses, timings, caching, response/request format, body, headers, status codes.. They're absolutely great for getting users to record one for a problematic interaction and sending over for inspection, though you must be careful with who you send HAR files to; as they contain everything, this includes authentication data!

There are multiple tools available online to inspect & view HAR files, or you can load them into the devtools of your browser, right next to where you would export one.

![A HAR file viewed with http://www.softwareishard.com/har/viewer/](harscraping_viewer.png)
_A HAR file viewed with the [softwareishard tool](http://www.softwareishard.com/har/viewer/)._

Generating a HAR file for the Facebook photo album was as simple as:
1. Go to the album.
2. Open the browser devtools network tab, make sure it's recording.
3. Open the first photo in the album.
4. Hold the right arrow key of the keyboard to flick through every photo.
5. Once the end is reached, export the HAR file.

![A screenshot of the Chrome devtools network tab after flicking through a Facebook photo album.](harscraping_devtools.png)
_571 requests after going through an album with 35 photos.. No wonder Facebook feels so sluggish._

Now after exporting we have a 40 megabyte HAR file with all of the data we need! The next step is to fish out all of the URLs of the photo album's images.
HAR files are actually just (very long) JSON inside, I was initially planning to get the URLs I needed by crafting a Regex, but a quick Google revealed a .NET library *[HarSharp](https://github.com/giacomelli/HarSharp)* for parsing & traversing HAR files.
