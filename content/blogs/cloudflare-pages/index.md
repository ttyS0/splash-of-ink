---
title: "Cloudflare Pages"
summary: "Using Cloudflare Pages For Hugo Hosting"
showSummary: true
heroStyle: "basic"
date: "2022-12-07"
series: ["Using Cloudflare"]
series_order: 2
tags: ["Cloudflare","SelfHosted"]
---

### Overview

I've had a rocky time with blogging in general. While I've been collecting domains for a long time, and writing things periodically, the fiddling with the blogging software has been a persistent distraction. I've tried any number of things, including multiple static site generators, and a few years ago I settled on a self-hosted [Ghost](https://ghost.org/docs/install/) instance. I also split content into different sites based on type, which seemed like a good idea at the time. As it turns out, both of those decisions ended up creating new barriers to creating content instead of facilitating it. 

Another interesting side effect of self hosting my sites came in the form of [Mastodon](https://skj.social) verification. I also self-host my Mastodon instance, and since the blog instances were inside the same Kubernetes cluster the DNS rebinding caused verification to fail. Frankly, this was a minor issue, but it did get me thinking again about how I wanted to handle things. 

After running down a number of familiar dark alleys, I finally settled (again) on using [Hugo](https://gohugo.io). I had aspirations of self-hosting the site, especially since it's just static files. While I may still do that someday, I also realized that I was again creating a problem to solve as a way of avoiding writing down content. Eventually I settled on the idea of not self-hosting the site. After running down a few more rabbit holes, I eventually settled on using [Cloudflare Pages](https://pages.cloudflare.com). I'm already using Cloudflare for my domain registration (mostly), DNS, and [Cloudflare Tunnels]({{<ref "/blogs/cloudflare-tunnel">}} "Cloudflare Tunnels"), so adding Pages into the mix seemed reasonable. 

### Setup

Basically Cloudflare Pages allows a GitHub or GitLab repository to be hooked up to it, and then a build command is defined. The output of the build is then distributed to the Cloudflare CDN, making it available from either the Cloudflare Pages FQDN, or via a DNS alias. As with my other Cloudflare usage, outside of the domain registration, I'm only using free tier services. For my little site, the free tier is more than adequate. 

Setup via the Cloudflare Pages web UI is straightforward. The first step of creating a deployment involves connecting up a GitHub or GitLab repository, which requires setting up access, in the case of GitHub, a GitHub App configuration. The walk through is clear, and can be scoped to a single repository, which is what I did. In [that repository](https://github.com/ttyS0/splash-of-ink/) I have all the Hugo config and content. I `.gitignore` the Hugo output, since Cloudflare Pages generates the output when it runs. 

Cloudflare Pages can also be setup to watch one or more git branches, with one being designated as the `Production` branch. Other branches are designated `Preview`. This means that if I want to try something out, I can do it in a `drafts` branch. When those changes get pushed to GitHub, Cloudflare Pages builds that branch, and produces a specific URL to view the output. If I'm happy with it, then I can merge to my `main` branch, and push. That will cause Cloudflare Pages to build and push to the `Production` deployment, which is where the site's DNS entry can then be pointed.

Another important area of configuration is in the `Environment variables` section. The container image that is used to do the Hugo build pulls in a very old version of Hugo when the container is instantiated. Setting an environment variable of `HUGO_VERSION` instructs the container to pull in a specified version. Environment variables are also separated by `Production` and `Preview`. This means that when a new version of Hugo is released, I can update the version used for my `drafts` branch, and verify that it builds correctly before setting it for my `main` branch build. 

There are a number of Serverless components that can be added, and that's the area where money comes into play. For me, I'm not interested in any of that, and just want to have my static site served. 

When everything is setup, I'm able to use my favorite writing application ( currently [Obsidian](https://obsidian.md) ) to create content. I have Hugo installed locally if I want to see how things look immediately, and once I do a `git push` the new content is available to everyone else. At the moment, I'm still doing the git operations via the terminal, but I've started playing with the [Git plugin for Obsidian](https://github.com/denolehov/obsidian-git). I suspect that I'll soon be comfortable with performing the requisite git operations directly from the editor. 

### Gitea

The first wrinkle I encountered came from the fact that I self-host a [Gitea](https://gitea.io) instance, and consider it my "source of truth" for my git repositories. Ideally, I would like Cloudflare Pages to  interact directly with my Gitea instance. That, unfortunately, is not an option. The solution, however, is simple. 

I've had a GitHub account for an extremely long time, and self hosting my repos is a relatively new development. Add to that the fact that I'd just as soon have people hitting GitHub instead of my Gitea instance by default. Thankfully Gitea provides a per-repository configuration option to specify a scheduled `push mirror`. For the majority of my repositories, I have Gitea push all commits to a corresponding GitHub repository once or twice a day. Since I want the cycle to be a bit faster with the blog, I have it configured to once an hour. Gitea also provides a `Synchronize Now` button, which is handy if I want the content to be deployed immediately. 

Once the content is synced to GitHub, the Cloudflare build happens automatically. So, unless I want something to post immediately, I know that sometime within a couple hours of running a `git push` my new blog content will be available without me needing to do anything. 

### Final Thoughts

In general, when I find a new thing, I tend to slowly migrate _everything_ to that new thing. At some point that always becomes problematic, and I start looking for alternate solutions. I suspect I'll reach that stage with Cloudflare as well, but for now I'm happy with Cloudflare Pages being the hosting endpoint for my Hugo based site. I'm also happy that my content has returned to being file instead of database stored. 

There's a lot more I want to do, and the odds are high that I will eventually pull this site back into a self hosted environment. For now, though, I'm focussing first on creating more content, and trying to not get too bogged down into the procrastination tar pit of fiddling with the hosting and deployment parts. So far Cloudflare Pages has done a good job of providing me an "easy enough button". 