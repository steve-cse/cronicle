---
title: I tried HTMX
date: July 26 2025
excerpt: Read about my wholesome experience moving from Hugo to HTMX, complete with the mistakes I made and lessons learned along the way.
author: Steve Boby George
---


You might be wondering, why make a blog post on HTMX this year rather than last year when the hype was at its peak? Well, better late than never. I've been looking to rewrite the code for this blog from scratch since a long time and why not rewrite it in the most hyped/powerful HTML extension even though it's [dead](https://htmx.org/essays/future/). At first I was pretty skeptical whether I could code with minimal JavaScript because I've been using React, Next and Vue for every single project and it was getting boring. The added advantage of HTMX is maintaining a single code base. So I went with Java Spring Boot for the backend along with Thymeleaf for templating (should have gone for JTE but live and learn I guess). 

## Getting Used to the HTMX Mental Model

![horse](/images/i-tried-htmx/horse.jpg)


At first it was quite weird to be honest to get around the mental model of HTMX but after working with it to render a few pages you get used to it pretty quickly, and the documentation for HTMX is excellent. So what's the mental model? Well instead of returning JSON to update the page you send the required HTML from the backend server, embracing it, and HTMX does its magic by swapping the response, a partial view with the old elements in the DOM. 

```html
<body>
<div id="container"
     hx-get="/posts"
     hx-trigger="load">
</div>
</body>
```

The example above demonstrates the `hx-get` and `hx-trigger` attributes, which basically loads the content (i.e. HTML) from the specified endpoint on initial page load. OK, this seems easy enough right? 

## Markdown Parsing and Frontend Libraries

Well now with HTMX basics down, I need a parser to parse my markdown to HTML. I scoured the internet and stumbled upon [flex mark](https://github.com/vsch/flexmark-java) a java library which has a multitude of parsers, it even has a [GitLab](https://github.com/vsch/flexmark-java/tree/master/flexmark-ext-gitlab) parser—how cool is that!

Next, for the frontend experience, I want a library that displays math on the frontend [KATEX](https://katex.org/) was my go to here and [highlight.js](https://highlightjs.org/) for code highlighting. 

## The Tailwind CSS Mistake

Now here's where I started making mistakes. I started using tailwind for my styling, the Tailwind Play CDN to be specific. About 50% of my UI was done and then I read the fine print "The Play CDN is designed for development purposes only, and is not **intended for production**". So I switched over to using the [tailwind CLI](https://tailwindcss.com/docs/installation/tailwind-cli) (I miss you npm) which has to be on during active development since it scans your HTML for styles and creates the corresponding `output.css` file—I learned that after 5 minutes from the installation when my styles weren't being applied. 

## Navigation Struggles

Well now my CSS is sorted and now we can head on over to navigation. Whenever a user clicks on a new post he/she has to be routed to a new page with the posts content. After reading the HTMX documentation I thought lets use `hx-push-url` attribute which allows me to push a URL into the browser location history but here's the problem—whenever the browsers forward and backward buttons are pressed it breaks the page and returns the partial view rather than the full one.

Searching for a solution I landed on the official subreddit for HTMX where many comments wrote just check the response header on the backend and return the full or partial view based on it. This worked but on page reload only the partial post content was returned and the main layout of the page is not loaded as HTMX cache is cleared. `hx-push-url` is good for custom AJAX interactions that should update the URL—an example for this could be a filtered search. For my use case `hx-boost` is the attribute to go for since when the server returns the HTML, HTMX intercepts and extracts just the required portion of the page. So `hx-boost` basically turned my `<a>` tags into AJAX requests thereby preventing reload while navigating. 

## Mobile Responsiveness with Alpine.js

Now that navigation is up and running, it's time to make the site mobile responsive. 

![jim](/images/i-tried-htmx/jim.jpg)

Now since this is a small scale site I kept the mobile responsiveness for last as I was still skeptical whether I would make this far. Since HTMX doesn't play a vital role in client side rendering I need another library for this. I chose [Alpine.js](https://alpinejs.dev/) 

```html
<header x-data="{ open: false }">
  <nav class="flex justify-between items-center py-4 px-6">
    <a href="/" class="text-red-500 font-bold text-xl">
      Bytecron Labs
    </a>

    <!-- Desktop Nav -->
    <div class="hidden md:flex gap-4">
      <a href="#" class="text-white hover:text-red-500">Github</a>
      <a href="#" class="text-white hover:text-red-500">Twitter</a>
      <a href="#" class="text-white hover:text-red-500">LinkedIn</a>
      <a href="#" class="text-white hover:text-red-500">Email</a>
    </div>

    <!-- Mobile Menu Button -->
    <button @click="open = !open" class="md:hidden text-white">
      <svg x-show="!open" class="w-6 h-6" fill="none" stroke="currentColor">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 6h16M4 12h16M4 18h16"/>
      </svg>
      <svg x-show="open" class="w-6 h-6" fill="none" stroke="currentColor">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
      </svg>
    </button>
  </nav>

  <!-- Mobile Menu -->
  <div x-show="open" class="md:hidden flex flex-col gap-2 pb-4 px-6">
    <a href="#" class="text-white hover:text-red-500">Github</a>
    <a href="#" class="text-white hover:text-red-500">Twitter</a>
    <a href="#" class="text-white hover:text-red-500">LinkedIn</a>
    <a href="#" class="text-white hover:text-red-500">Email</a>
  </div>
</header>
```

With a little bit of tailwind media queries and alpine state variables we have a responsive navbar. Here we maintain an `open` state variable and display the menu whenever the user clicks on the hamburger icon. 

## Backend Optimizations

Finally, some backend optimizations. I'm a big fan of the DRY principle since I use it a lot at work so to maintain a common layout using Thymeleaf I used [Thymeleaf Layout Dialect](https://github.com/ultraq/thymeleaf-layout-dialect?tab=readme-ov-file#thymeleaf-layout-dialect)

```html
<!DOCTYPE html>  
<html lang="en" xmlns:th="http://www.thymeleaf.org"  
      xmlns:layout="http://www.ultraq.net.nz/thymeleaf/layout"  
      layout:decorate="~{layout}">  
  
<head>  
    <title>Posts</title>  
</head>  
<body>  
<div layout:fragment="content"  id="container"  
     hx-get="/posts"  
     hx-trigger="load">  
</div>  
</body>  
</html>
```

Here we specify that the current template should use another template as its base layout using the `layout:decorate` attribute. `layout:fragment="content"` will create a fragment that will get replaced by the content placeholder in the layout template. That solves the repetition problem. 

Now for the last optimization we have caching since each time a user hits a request the backend parses the markdown files which can be solved using Spring Boot's default cache manager.

```java
@Cacheable(value="renderedPosts", key = "#slug") 
public String getPost(String slug) throws IOException {
 /* Method to get post content */
}
```

Here the post's slug can be used as a key for eviction since it's unique. 

```java
@Cacheable("posts") 
public List<Post> getPosts() throws IOException { 
/* Method to list posts */
}
```

In the code above I have nothing unique to specify—in scenarios like this we can use a cron job for daily cache eviction at a certain interval. 

And that's it—this was my wholesome experience rewriting my site from Hugo to HTMX. As a conclusion I want to quote [Carson Gross](https://x.com/htmx_org), the unhinged creator of HTMX: "javascript fatigue: longing for a hypertext already in hand"