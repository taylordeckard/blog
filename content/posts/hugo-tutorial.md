---
author: Taylor Deckard
title: Meta Post (Create a Blog with Hugo)
date: 2021-01-19
description: Setting up a blog with Hugo
---

As an introductory post, I'll detail the steps to create a blog like this.
It should only take a few minutes...

![It should only take a few minutes...](/blog/images/hugo-tutorial/dumb_and_dumber_watch.webp)

## Install Hugo

Follow the steps [here](https://gohugo.io/getting-started/installing/) to install Hugo on your system. For me, on macOS, the command was:
```bash
brew install hugo
```

## Create a Site

Next, you'll want to use the hugo binary to create the project scaffolding for your site. Open up a terminal and enter the following command (substitute `my_blog` with the name of the directory you want to create the site in):

```bash
hugo new site my_blog -f yml
```
Then `cd` into the directory:
```bash
cd my_blog
```
At this point, you'll want to [pick a theme](https://themes.gohugo.io/). I chose [PaperMod](https://github.com/adityatelange/hugo-PaperMod). Instructions for adding a theme to your project can usually be found on the theme's GitHub page. In my case, the install consisted of:
```bash
git clone https://github.com/adityatelange/hugo-PaperMod themes/PaperMod --depth=1
```
Also as part of the theme install, a line was added to config.yml:
```yaml
theme: "PaperMod"
```
At this point, your project should look something like this:
```
.
├── archetypes
│   └── default.md
├── config.yml
├── content
├── data
├── layouts
├── resources
│   └── _gen
├── static
└── themes
    └── PaperMod
```

## Configuring the Theme

Usually detailed on the theme's GitHub page, there are a variety of configuration options available. These can be set in `config.yml`. I copied the PaperMod example configuration and made a few adjustments.

```yaml
baseURL: https://www.taylordeckard.me/
languageCode: en-us
title: Taylor's Blog
theme: "PaperMod"

baseURL: https://www.taylordeckard.me/blog
enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false

minify:
  disableXML: true
  minifyOutput: true

params:
  env: production
  title: Taylor's Blog
  description: "Things that happen to me"
  keywords: [Blog]
  author: taylordeckard
  DateFormat: "January 2, 2006"
  defaultTheme: auto # dark, light
  disableThemeToggle: false

  ShowReadingTime: true
  ShowShareButtons: false
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  disableSpecial1stPost: false
  disableScrollToTop: false
  comments: false
  hidemeta: false
  hideSummary: false
  hideFooter: true
  showtoc: false
  tocopen: false

  label:
    text: "Home"

  # home-info mode
  homeInfoParams:
    Title: "Hi"
    Content: Welcome to my blog

  socialIcons:
    - name: github
      url: "https://github.com/taylordeckard"

  cover:
    hidden: true # hide everywhere but not in structured data
    hiddenInList: true # hide on list pages and home
    hiddenInSingle: true # hide on single page

  editPost:
    URL: "https://github.com/taylordeckard/blog/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link

  # for search
  # https://fusejs.io/api/options.html
  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    distance: 1000
    threshold: 0.4
    minMatchCharLength: 0
    keys: ["title", "permalink", "summary", "content"]
menu:
  main:
    - identifier: archives
      name: Archives
      url: /archives
      weight: 30
    - identifier: search
      name: Search
      url: /search
      weight: 30
    - identifier: taylordeckard
      name: taylordeckard.me
      url: https://taylordeckard.me
      weight: 30
outputs:
  home:
    - HTML
    - RSS
    - JSON
```

## Creating Content

All of the markdown files you will create for your blog will go in the top-level `/content` directory. Right now it should be empty. There are a couple of files that are specific to the PaperMod theme and need to be created: 

`content/archives.md`
```md
---
title: "Archive"
layout: "archives"
summary: "archives"
---
```
`content/search.md`
```md
---
title: "Search"
layout: "search"
---
```
Next, create a directory for all of your posts: `content/posts`. In this directory, you will have a markdown file for each post you make.

For my blog, for instance, I created `hugo-tutorial.md`:
```
content
├── archives.md
├── posts
│   └── hugo-tutorial.md
└── search.md
```

`hugo-tutorial.md` looks something like this:
```md
---
author: Taylor Deckard
title: Meta
date: 2021-01-19
description: Setting up a blog with Hugo
---

As an introductory post, I'll detail the steps to create a blog like this.
It should only take a few minutes...

![It should only take a few minutes...](/blog/images/hugo-tutorial/dumb_and_dumber_watch.webp)

## Install Hugo
...
```
## Running Locally
Once you have something similar, you should be able to run the blog locally. From the project root directory, run the following:
```bash
hugo serve
```
Then open a browser and navigate to [http://localhost:1313/blog]. (The **/blog** comes from the _baseURL_ property set in the `config.yml`)


## Next steps
This is a good start to a blog, but I still need to create a GitHub repo for it and configure a pipeline to deploy the blog automatically to my personal website. If you'd rather, you could use [GitHub Pages](https://pages.github.com/) to host your blog for free.
