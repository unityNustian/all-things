# Blog Post File Naming Convention

This document outlines the file naming convention for all blog posts in this project. Adhering to this policy ensures consistency and helps with automated processing of the files.

## Directory

All blog posts must be located in the `_posts/` directory.

## Filename Format

The filename for each blog post must follow this format:

`yyyy-mm-dd-your_post_title.html`

### Components

1.  **Date (`yyyy-mm-dd`)**: The full date of the post, with a four-digit year, a two-digit month, and a two-digit day, separated by hyphens.
    *   Example: `2026-02-08`

2.  **Separator**: A single hyphen (`-`) must separate the date from the title.

3.  **Title (`your_post_title`)**:
    *   This should be a descriptive, lowercase title of the blog post.
    *   All spaces must be replaced with underscores (`_`).
    *   Avoid special characters other than underscores.

4.  **Extension**: The file must have an `.html` extension.

### Example

A blog post titled "My First Blog Post" written on February 8, 2026, should have the following filename:

`2026-02-08-my_first_blog_post.html`

## Working with the Gemini CLI Agent

When asking the Gemini CLI agent to create or rename a blog post, please refer to this policy to ensure the agent follows the correct naming convention. For example, you can say:

> "Please create a new blog post titled 'My Awesome Post' and make sure it follows the file naming policy."

By referencing this policy, you help the agent understand and enforce the correct file naming convention for this project.
