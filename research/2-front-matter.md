# YAML Front Matter System

Jekyll processes any file that contains a YAML front matter block at the top. The front matter is denoted by a pair of triple-dashed lines `---` at the very beginning of the file. Between these lines, YAML or JSON can be written to define variables for that page. For example:

```markdown
---
layout: post
title: Blogging Like a Hacker
tags: [tech, writing]
---
Markdown **content** goes here.
```

In this example, `layout`, `title`, and `tags` are front matter variables which Jekyll will parse (YAML keys and values) and attach to the page. Your SSG must be able to read this block, parse it as YAML (or JSON), and store these variables for use during templating. Anything below the `---` closing the front matter is treated as the page's main content (which will be rendered via Markdown or other converters).

## Parsing Rules

The front matter must appear at the very top of the file, before any content or whitespace. It is delimited by `---` (three hyphens) on a line by itself. Jekyll will read everything between the first and second set of `---` as YAML. (Jekyll also supports JSON front matter: if the first character after `---` is a `{`, it will parse the block as JSON instead. In practice, YAML is far more common.)

If a file does not have front matter, Jekyll usually treats it as a static asset (unless it's a known convertible file like a Sass file, which uses empty front matter to trigger conversion).

## Optional Front Matter

It's not mandatory to have variables in the front matter. A file can have an "empty" front matter (just the `---` lines with nothing in between) to signal Jekyll to process it. For instance, CSS or JS files can be processed by Liquid tags by putting an empty front matter at the top. Your SSG should recognize an empty front matter block as a valid indicator that the file needs processing.

## Predefined Variables

Jekyll recognizes certain front matter keys that have special meaning:

* `layout`: Which layout file (from `_layouts/`) to wrap this page with. If set to `null` or omitted, the page can be rendered standalone (no layout). (Note: In Jekyll 3.5+, using `layout: none` on a post/page forces no layout, whereas `null` can be overridden by default layouts.)
* `title`: Title of the page (commonly used in templates).
* `date`: (For posts) Overrides the date derived from the filename. Useful if you need a specific time or to correct ordering. Should be in `YYYY-MM-DD HH:MM:SS +/-TTTT` format (time and timezone optional).
* `categories` or `category`: For posts, adds the post to specified categories. Can be a YAML list or space-separated string. By default, Jekyll can also derive a category if the post is in a folder (see Section 8) but the front matter is a direct way to set it.
* `tags`: Similar to categories, a list of tags for the post.
* `permalink`: Overrides the default URL path for this page/post. This can be a URL structure or filename (e.g. `permalink: /about/` to output as `/about/index.html`).
* `published`: If set to `false`, Jekyll will exclude this page/post from the build output. This is a way to keep a post's source in the project but not publish it. (The SSG should honor this by skipping output for such items unless a flag like `--unpublished` is used.)
* Various other keys like `author`, `excerpt`, etc., which may be used by plugins or theme templates (for example, Jekyll SEO Tag plugin looks for `description`, `image`, etc. in front matter).

In addition to these, any custom key in front matter is allowed – Jekyll will simply make it available via Liquid as `page.yourKey`. Your Rust SSG should similarly allow arbitrary metadata in the front matter and make it accessible in templates.

## Default Front Matter Values

Jekyll provides a mechanism for setting default front matter values via the configuration file. Under the `_config.yml`, a `defaults` section can define rules to apply certain front matter to files that match a scope. For example, you can specify that all posts should use a given layout by default, or all files in a certain directory get a specific tag. Here is an example from Jekyll's docs:

```yaml
defaults:
  - scope:
      path: ""      # an empty string means all files
      type: "posts" # only for posts (collection type)
    values:
      layout: "default"
```

This default sets the layout to "default" for all posts, so you don't need to put `layout: default` in every post's front matter. You can also target other collections or page types. The `scope` can match a `path` (folder path or pattern) and/or a `type` (like `posts`, `pages`, or a custom collection name). Multiple defaults can be defined; more specific scopes override less specific ones. For example, you might have a default for all pages, and another default for pages in a `projects/` subfolder that overrides the layout or adds an author.

Your SSG needs to implement this resolution of default metadata: when building the page/post data, first apply any matching defaults (in order of specificity as Jekyll does), then apply the page's own front matter (which overrides defaults). This ensures that if a user has configured a default e.g. `published: false` for future-dated posts, or a default `author` name, it will be applied unless the individual page sets its own.

Additionally, Jekyll supports **glob patterns** in the default scope path (with `*` wildcards) as of v3.7.0. For instance, `path: "section/*/special-page.html"` could match any `special-page.html` file one level under the `section` directory. This allows flexible targeting but note that it can affect build performance (globbing many files, especially on Windows, is slower).

## Validation and Strict Mode

The front matter must be valid YAML/JSON. If it's not, Jekyll will usually error out during build. There is a **strict front matter** mode that can be enabled (via CLI `--strict_front_matter` or config `strict_front_matter: true`) which causes the build to **fail** upon encountering any page with malformed front matter.

In practice, even without strict mode, an invalid YAML (e.g. bad indentation or missing a closing quote) will raise a parsing error and prevent the site from generating. Your SSG should similarly catch YAML parser exceptions and report them, ideally halting the build for a truly invalid file. The `--strict_front_matter` option in Jekyll adds "additional scrutiny" – for example, it might treat a page with front matter delimiters but no valid YAML between as an error rather than just ignoring the file. To mirror this, you can offer a strict mode to enforce that all files with `---` have properly formatted YAML and perhaps no stray `---` within content, etc.

## Additional Rules

All files in Jekyll (posts, pages, drafts, collections) require the front matter to be at the very top. Ensure your parser disregards any byte order mark (BOM) or whitespace before the `---`. (Jekyll specifically warns that UTF-8 BOMs can break the parser.)

Also, note that the presence of front matter is the trigger for processing. For example, a Markdown file with no front matter will **not** be converted to HTML by Jekyll – it will just copy as-is (or possibly be ignored if in an excluded folder). To force processing of a Markdown file, it needs front matter (even if empty). This behavior should be duplicated in the Rust SSG to match Jekyll's output.

## Implementation Summary

Implement a robust front matter reader that can:

* Detect and parse YAML/JSON front matter.
* Handle empty front matter gracefully.
* Merge in site-wide default values as Jekyll would.
* Validate syntax and handle a "strict" mode.
* Expose all front matter variables for use in the templating phase. 