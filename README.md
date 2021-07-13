# CEDA Technical Blog

Blog to publish informal and technical articles about activities at CEDA.

Contents:
- [Orientation](#orientation)
- [Quickstart](#quickstart)
- [Getting started with Jekyll](#getting-started-with-jekyll)
- [Writing and article](#writing-an-article)
- [Useful resources](#useful-resources)

## Orientation

The purpose of this blog is to publish information about activites at CEDA which might
be interesting for others to know about. The approach is less formal and provides a space
where we can talk about experimental and unfinished projects. The original motivation
for this was to share a project with interested external partners and provide space to explain
the reasons for certain decisions.

The blog posts are written using the [Jekyll](https://jekyllrb.com/) framework.
This is a static website writing framework which uses markdown and [liquid](https://jekyllrb.com/docs/liquid/) (a templating
syntax similar to jinja).

Blog posts are written in the `_posts` directory in [markdown](https://www.markdownguide.org/basic-syntax/) format. Posts can be
assigned categories by putting them in subfolders. For example, data management articles
could be put under `_posts/data_management`. The categories will then become part of the
url and can be used to group related articles together. 

Changes to the blog are pushed to their own branch and merged, via github pull request
to the `gh-pages` branch. Github then builds the blog and publishes it.

## Quickstart

1. `git clone https://github.com/cedadev/dev-blog`
2. `cd dev-blog`
3. `git checkout gh-pages`
4. `git checkout -b article/<article-bramch-name>`
5. Copy the example article in the `_drafts` directory
6. [Start writing](#writing-an-article)

If you want to be able to see the effect of you edits in real time, you'll want to 
look at [installing jekyll](#getting-started-with-jekyll). As articles are written
in markdown, you don't need to use Jekyll but it might be useful if you are also using
diagrams.

## Getting started with Jekyll

The jekyll documentation gives a [quick start guide](https://jekyllrb.com/docs/#instructions) to setting up Jekyll. It uses ruby
to provide local server which you can use when developing your articles.

## Writing an article

You should follow good git practice and develop your articles on a branch. Merging to `gh-pages` is 
protected and requires 1 approving review from senior management, just as a final check. Setting up
you environment to write an article is covered in [quickstart](#quickstart)

Articles are written in markdown. The content is up to you although it makes sense to follow
a [common structure](#article-structure). Feel free to get technical. Think about the audience
you are trying to reach.

The section at the top of the article is called [front matter](https://jekyllrb.com/docs/front-matter/).
You will need to fill in values for some of the front matter. i.e. `Author`, `date`, `tags`, `title`
```
---
layout: post
title:  "Search Futures"
author: Richard Smith
date:   2021-07-05 17:00:00 +0100
tags: search indexing stac
---
```

Using Jekyll's Liquid templating language, you can make use of various includes to aid in your
post formatting. [Current includes](#includes).

### Article Structure

Look at the example in the `_drafts` directory to get an idea of the structure.

### Article assets (images and such)

It is likely that you'll want to add images. Add your images to the `assets` directory.
Assets should be organised following the `_posts` structure. For example:

```
dev-blog/
├─ assets/
│  ├─ img/
│  │  ├─ data_management/
│  │  │  ├─ 2021-07-09-data/
│  │  │  │  ├─ amazing_img.png
├─ _posts/
│  ├─ data_management/
│  │  ├─ 2021-07-09-data.md

```

Images can be directly linked to using the normal markdown image link syntax:

`![amazing](/assets/img/data_management/2021-07-09-data/amazing_img.png)`

For convenience, I have added a figure template. To add an image with a caption.

```
{% include figure.html 
    image_url="/assets/img/data_management/2021-07-09-data/amazing_img.png"
    description="Amazing caption to go underneath the amazing image" 
%} 
```

You can look inside the `_includes` directory for other includes and discover their
options. This is where you can also add your own includes.

### Linking to other posts

In order to link to other posts without worrying about broken links, use the `post_url`
tag.

```
{% post_url 2010-07-21-name-of-post %}
```

## Includes

| Name | Description |
|------|-------------|
| [Figure](#figure) | Insert a figure into your article with a caption aligned centre underneath | 
| [Caption](#caption) | Standalone template to add a caption |


### Figure

Renders an image with a caption centre-aligned below.
Defaults to be 50% of article width.

![Figure Example](assets/img/figure_example.png)

#### Configuration

| Option | Description |
|------|-------------|
| image_url | `required` URL to the image, can be an asset or external link |
| description | `required` Caption to be placed under image |
| image_style | modifies the style for the image. default: `width: 50%; margin-left: auto; margin-right: auto; display: block`


## Useful Resources

- [Markdown Syntax Guide](https://www.markdownguide.org/basic-syntax/)

