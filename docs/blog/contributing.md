---
hide:
  - navigation
  - toc
---

# Getting Started with Blogging

Are you wanting to contribute to the blog? This simple guide is just a quick "getting started" guide on creating a new blog post.

## Add yourself as an Author

To add yourself as an author simple add an entry for yourself in the `docs/blog/.authors.yml` file. The entry should consist of the following:

!!! example "New author"

    ``` yaml
      __GITHUB_USERNAME__:
        name: __ENTER_YOUR_NAME__  # Rquired
        description: __SIMPLE_DESCRIPTION__  # Rquired
        avatar: https://github.com/__GITHUB_USERNAME__.png  # Rquired
        github: __GITHUB_USERNAME__  # Rquired
        twitter: __TWITTER_USERNAME__  # Optional
        linkedin: __LINKEDIN_USERNAME__  # Optional
        website: __WEBSITE_IF_YOU_HAVE_ONE__  # Optional
    ```

!!! NOTE

    All options with a double underscore need to be replaced with your relevant information.

If you do not wish to have attribution you may skip this step.

## Create your post

All posts should be written in markdown format. We're using [mkdocs](https://squidfunk.github.io/mkdocs-material/) for our platform which has a lot of plugins available to it. Basically we're supporting code, codeblocks, and mermaid diagrams; however, as you get started with writting, if there's a plugin you'd like to use, send us a pull request to get it added.

### Create a new file

1. Add your file to the `docs/blog/posts` directory.
2. The file format should have the date within it and a string.

    !!! note

        The formatting should follow a year, month, day, and a string: `YYYY-MM-DD-STRING_NAME.md`.

### Post heading

The top of the article should have the following heading. This heading is what sets the creation date and applies the appropriate attribution.

!!! example "Minimal post header"

    ``` md
    ---
    date: YYYY-MM-DD
    authors:
      - __GITHUB_USERNAME__
    ---
    ```

!!! note

    If you are not interested in attribution, set the entry for "authors" to **rackerlabs**.

Blog post headers can be extended to provide readers a lot of information at a glance

!!! example "Complete post header"

    ``` md
    ---
    title: __title_string__
    date: YYYY-MM-DD
    authors:
      - __GITHUB_USERNAME__
    slug: __slug_string__
    description: __description_string__
    categories:
      - __category_string__
    ---
    ```

It is highly recommended that you add some categories to your post. If no category comes to mind, use the **General** category.

### Post body

While the formatting of the blog post itself is up to you, we would highly recommend adding the page break after the opening paragraph to the post.

!!! example "Post synopsis"

    ``` html
    Example opening paragraph

    <!-- more -->

    Content body...
    ```

Using this option will create a synopsis for our main page, allowing more articles to be seen at a glance.
