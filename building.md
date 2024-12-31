# Designing and Building my Personal Blog

## Background

A few months ago I was looking to start blogging, not because anything I have anything profound or unique to say but because I thought it would be a fun side project and be a motivator to make more side projects(stay sharp and learn some things). 

First step was to investigate how different people build/host personal websites. There seems to be two main camps;

 * blogs where everything is taken care of for you 
 * blogs where everything is bespoke
 
 Using wordpress or medium is the way to go if you are interested in things like ease of use, maintainability, and discoverability. My blog will lean much more heavily on the bespoke side of the spectrum

There are some real advantages to building a unicorn blog 

* customization is limited by my skill
* play with libraries/ideas I can't in my day job

After some research I wrote this high level set of requirements. Here's what I came up with, preserved in all its overwrought glory
```
## Goals
-  build a personal website
   - which I can publish blogs without code changes
   - looks pretty and is secure
- build the infra to host the website 
  - terraform cloud infra
  - build for zero touch day 2 ops
- observability
  - implement tracing and metrics
  - build for avalibility
  - define slo/sli
  - display uptime in blog 7-30-90 view
  - monitor and alert to keep the blog to meet slo
- cicd
  - quick iteration loop
  - CI/CT/CD 
  - static analysis
  - vuln analysis

# blog design
- go backend
- html/js/htmx frontend
- tabs
  - home default
    - up to 5 latest articles with titles and previews
  - blog
    - pageable list of arcticles with previews -> click on title to naviate to full articles
  - uptime
    - list of slo's in 7 - 30 - 90 view
  - about
    - static page about author
- publishing article 
  - I want to be able to publish blob articles without making a code changes
  - the articles should be agnostic of any code
  - articles should support embedding images
  - articles should be hosted outside of website

# infra 
- k8 cluster
- how cheap can I get a managed service
- maybe standup cluster use free tier vms as nodes
- publish infra metrics and alerts
- consider day 2 maitenance
- dynatrace or something that supports opentelemtry traces
```

The observability section stands out now - I actually went down this rabbit hole, spinning up a Jaeger server, Prometheus, and Grafana stack before reality (and my wallet) kicked in. Turns out, maintaining an observability platform for a personal blog is expensive, hard, insecure, and of questionable value. If you are ever tasked with observability at your job; just pay for datadog. My original intent with these requirements was to show ability to maintain a software system; I may end up implementing most of this but I need to ship something out the door first.

Stripping away all the enterprise architecture fever dreams, I realized I really just had a few core requirements:

* I need to be able to blog without a ton of friction
* I don't want it to fall over
* I want it to be a little unique

Time to actually build this thing, before I convince myself it needs a service mesh.

## Backend

The core architecture is fairly straightforward - components which handle different concerns like content managment, telemetry, or http requests orchestrated by what I called the BlogServer. 

```
func main() {
    bs, err := blog.NewBlogServer(
        blog.WithConfig("BLOG_"),
    )
    if err != nil {
        log.Fatal(err)
    }
    err = bs.Start()
    if err != nil {
        log.Fatal(err)
    }
}
```
When the BlogServer starts, it:

1. Initializes the telemetry storage and export pipeline
2. Starts the listener for content updates
3. Triggers an initial content update
4. Starts the HTTP server
5. Manages graceful shutdown when signaled

These were the big design decisions that I ended up making

* use environment variables for all configuration
* store content in git rather than a database
* pre-render html
* I don't need to persist any state


Rather than building a CMS with a database and admin interface, I decided to use Git as my content store. The workflow is simple: write markdown files, commit them to a repository, and the blog can pull and render them. This solved my "friction-free blogging" requirement - I can write content in my favorite editor and publish with a simple git push. The BlogManager watches for updates (SIGHUP) and handles pulling new content, converting markdown to HTML, and updating the article index.

For observability, after abandoning my enterprise-grade dreams, I built a simple in-memory telemetry system. It maintains only the most recent trace, a time object for calculating uptime and a counter for articles served. The implementation is bare bones - each request gets traced, article views get counted, and everything disappears on restart. Is it production-grade? No. Does it give me a warm fuzzy feeling seeing traces in my UI? Yes.

Content updates happen concurrently without blocking the server - a mutex ensures safe access to the articles while updates are processing. When new content arrives, the BlogManager pre-renders both the individual articles and the article list, so serving requests is just a matter of returning the cached HTML.
The actual HTTP routing is minimal - serve static files, handle article requests, expose some basic telemetry endpoints. Nothing fancy, but it gets the job done without overcomplicating things.

## Frontend

For the frontend, I kept things deliberately minimal. Not for any principled reasons 

The entire UI is just vanilla HTML with a dab of JavaScript for tab management and HTMX for dynamic content loading; Just a single HTML file and some CSS. Rather than server-side rendering everything or building a full SPA, I used HTMX to progressively enhance the page. 

The tab system is probably the most complex part of the frontend, and even that's just ~50 lines of JavaScript. It handles both click navigation and direct URL access via hash fragments.

I think it looks quite sharp, runs reasonably quick, and is simple enough a frontend neophyte like myself can understand whats happening.

## Deploying

I just completly skipped doing this properly with cicd and automating in any sort of fashion. I just copied the binary to an aws lightsail instance and called it a day. I will come back again but at this point I just want to see something on the public internet. I've procastinated way too much already - leaving it running in a tmux pane is a good enough for day 0.

## Retro

Some decisions that did work well was

* git is a good cms
* not persisting state simplified development immensely

The big lesson I had to relearn throughout this whole project was stop trying to find perfection and get the mininmium done. Have something to show and iterate on; is always more important than getting it all done perfectly in one big push.

This behavior is always easier to spot in someone else. Especially if you have any emotional attachment to the code. Show the world your frankenstein creation and actually follow through with the iteration.

What I will look at next is automating the deployment, setting up some monitoring, and making some actual tests. When I get that done I hope to add some feature enhancements like an rss feed or a sitemap to help with discoverability. Beyond that I would like to profile the blog and see how much performance I can squeeze out of it.
