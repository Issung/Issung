---
title: Hello World!
date: 2024-02-03 5:00:00 +0000
categories: [Blogging]
tags: [blogging, opensource, jekyll]
img_path: /assets/img
---
Hello world! Welcome to my new blog! Something I had been meaning to get around to for a while now, my old blog was written in .NET Framework, and used Razor, it was made before I got my first job. Over the last few years I've learnt much more about using the right tool for the job, now I see that that is completely overkill for a simple static blog, especially one that will only get touched & read a handful of times a year.

This new blog is powered by a collection of opensource technologies and exploitation of the small amount of good-will of a large corporation. This first post will detail the "stack" that makes it possible.

# Static Site Generation And You

First thing is first, I knew I wanted to use static site generation. The benefit of static site generation is that you can dynamically add content or pseudo-components and then generate flat HTML output. This allows for multiple benefits, for example creating a make-shift index of all your existing posts for a wholly-client side search function, and more importantly being able to host your site completely for free using a host like GitHub, Netlify or Cloudflare.

The go-to static site generator for blogs is [Jekyll](https://jekyllrb.com/), GitHub has official support for processing Jekyll sites, even before GitHub Actions was released! Jekyll has been widely adopted since then and even has its own sort of ecosystem for plugins and themes!

# Themes

I didn't want to create a whole design and styling for a blog again so I went looking for a theme, [Klise](https://github.com/piharpi/jekyll-klise) initially took my eye but it had a restriction of not being able to use GitHub pages due to one of the pre-processing it uses steps not being supported by GitHub. After more searching I came across [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy), which had all the features I was looking for.

![Chirpy theme](helloworld-chirpytemplate.png)
_What a familiar looking template..._

The maintainer provides a template repo ([chirpy-starter](https://github.com/cotes2020/chirpy-starter)), which is a GitHub feature allowing you to easily copy the repo into another, and receive updates to the template over time. The template isolates the nasty guts of the theme away so you can focus on writing posts. I might want to perform more modifications to the blog source code but for the time being this sounded like the best option to try it out easily.

# Running With Ruby
After cloning the repo I needed to check it out locally and change some configuration and write this post!

Jekyll is a [Ruby](https://www.ruby-lang.org/en/) project meaning we have to install it to install gems (packages) and run it locally. For this sort of task I now use [scoop.sh](https://scoop.sh/) which is the best/closest thing to a package manager on Windows I've found. Installing was as easy as `scoop install main/ruby` and then following the ruby installer's follow-up actions for installing MSYS2.

# Finishing touches
From here all you need to do is enable actions and pages in the repo settings on GitHub, and set some configuration in a _config.yml file which is necessary for path tricks and some customisation like the logo, title and description. I also really wanted to have comments, which was available by just plugging in some [Giscus](https://giscus.app/) settings. 
I think Giscus is fantastic, a way to get comments on your site for completely free, it's funny the ways people find to squeeze all the functionality they can out of a free service. The comment thread on each post is really just a different discussion on the GitHub repo.

# Housekeeping
I have archived my old blog's repository now and made it public at [https://github.com/Issung/OldBlog](https://github.com/Issung/OldBlog). I might transfer some old posts over from the other blog in the coming weeks. The first new posts are likely to be about the creation of the PS2IODB project currently in development. Stay tuned!
