---
layout: post
title: Creating a new blog with Jekyll and GitHub Pages
date: 20-08-2022
categories: programming
hero: https://res.cloudinary.com/dbzsk4ytb/image/upload/v1661027101/blog-images/JourneyPath_maza5m.jpg
---

Even though I've been in software development for almost my whole career, I've never had the chance to build a website
from scratch. I've created a WordPress site before - but that hardly counts as programming!

### Getting Started on My Journey

So when it came time to create a personal blog, I dove in head-first and learned a bunch of new things. Initially, I ran
across [Jamstack]( https://jamstack.org/) as a new way of building static sites. Jamstack seemed like a much better
technology stack for what I wanted to create than WordPress.

At first, I shied away from [markdown](https://en.wikipedia.org/wiki/Markdown) for content creation. While I had
used [Slack](https://slack.com/) for instant messaging, which utilizes markdown, I wasn't sure I would have enough
formatting control for a blog site. This reluctance led me to the idea of a headless CRM for content creation and a site
generator to pull from the CRM to create the pages.

### The first attempt

My initial technology stack utilized [Ghost](https://ghost.org/) for CRM and [Gatsby](https://www.gatsbyjs.com/) as the
site generator. Ghost's page editor is great - easy to use and powerful. But, the connection between Ghost and Gatsby
was a little clunky. I also struggled with the formatting and structure I wanted. I didn't have a solid background in
HTML and CSS, which made the jump to learning [React](https://reactjs.org/) difficult.

After spinning my wheels for a while, I eventually just pulled out Gatsby and went with a pure Ghost site. Controlling
the layout was more manageable, and I could make a structure I was happy with - pushing the site through Gatsby didn't
add much value.

But now I needed a server up 24x7. I spun up a Ghost instance on [AWS/Lightsail](https://aws.amazon.com/lightsail/),
moved my domain to [AWS/Route 53](https://aws.amazon.com/route53/), and connected the pieces. Everything worked great,
but I had monthly bills and a more complex stack than necessary for a simple site like this blog.

### The second try - progress!

After some more research, I ran across [GitHub Pages](https://pages.github.com). It offers free hosting within extremely
generous usage patterns. GitHub Pages uses [Jekyll](https://jekyllrb.com) as a site generator - getting me back to a
more Jamstack approach to building the site. As a developer, I already have a [GitHub](https://github.com) account and
am familiar with [git](https://git-scm.com), so pushing updates to GitHub to update the website was reasonable.

But I wanted a layout that wasn't immediately available as a Jekyll theme. So, it was time to fix my lack of depth in
HTML and CSS and design a site! Using the layout of my prior Ghost site as a template, I eventually figured out the
right combination of HTML, CSS, Jekyll templates, and a little Javascript to put the site together. You can find all the
pieces I put together in my site's repository
at: [https://github.com/mikehentges/mikehentges.github.io](https://github.com/mikehentges/mikehentges.github.io).

In the end, markdown is more than adequate as a content creation tool. Once I figured out how the Jekyll templates
worked, I could easily use CSS to format blog entries and pages the way I wanted. I can run the whole site locally under
Jekyll, so iterative development is easy. And the automatic push to Github to publish the site works great.

### Some Jekyll tips/tricks

I did have to solve a few problems before I had everything working. I wanted a dynamic navigation menu that showed
categories of posts on the site. I didn't want to have to re-code the Html for the menu when categories changed and
instead drove it dynamically out of a data file. Jekyll has a great set of available plug-ins for doing different
things ([Awesome Jekyll Plugins]( https://github.com/planetjekyll/awesome-jekyll-plugins)). I found
the [Jekyll Data]( https://github.com/ashmaroli/jekyll-data ) plug-in for reading data files and using them dynamically
in templates. I created a straightforward ```categories.yml``` file and used it for the main navigation menu.

Everything worked great locally, but I ran into problems when I pushed it to GitHub. GitHub Pages runs in a safe mode
and has a set of allowed plug-ins – it does not natively support the Jekyll Data plug-in. So when I pushed my site to
GitHub, it wouldn't generate the static site correctly. Fortunately, a custom GitHub
action, [Jekyll Deploy Action](https://github.com/jeffreytse/jekyll-deploy-action), solves this problem. The Jekyll
Deploy Action allows you to set up a custom GitHub Actions build environment with Jekyll and any plug-ins you'd like to
add. I added the right ```./.github/workflows/build-jekyll.yml``` file to trigger the GitHub Actions on an update to the
main branch. So now the sequence is:

1. I commit my new/changed markup files. ```git commit -m 'some new changes…' ```
2. I push the changes to GitHub. ```git push```
3. A GitHub Action triggers, creating a Jekyll run-time environment with the plug-ins I want/need all installed.
4. A set of generated static files (.html, .css, .js) and any file artifacts are put in the repository's gh-pages branch
   by the GitHub Action.
5. GitHub pages deploy the generated files, and the updated site is active!

Jekyll uses the liquid template language [https://shopify.github.io/liquid/](https://shopify.github.io/liquid/) to drive
the data substitution needed to create the static files from the collection of data and markup files. Mainly it involves
simple data substitution, using handlebars to place data elements within the Html files, {% raw %}```{{site.title}}```{%
endraw %} for example. Programmatic control is also available, so you can iterate over a collection to create things
like a list of posts. My home page has three cards for the most recent posts on the site. To make each one, I had to
find the correct syntax to iterate over the posts – but only three times. The syntax to do that is:

{% raw %}

```
    {% for post in site.posts limit:3 %}
      <div>
        repeating data goes here, 
        like {{post.content}}
      </div>
    {% endfor %}
```

{% endraw %}

### End Result

I now have a very lightweight and fast-loading site. Jekyll builds everything as static files. I only have a short
javascript routine for reacting to a mouse click on the front page cards. Everything else published for the site is Html
or CSS files that load very quickly. All the content and formatting are under my control to update or change. Learning
Jekyll, HTML, and CSS has been a nice side-effect of the journey, and I think I'm in a better spot to go back and tackle
React one day.

If you are contemplating putting together a blog or are working towards creating a Jekyll site, I hope the information
here is helpful.
