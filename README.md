# bqst.github.io

## About

This is the source code for my personal website, which is hosted at [bqst.github.io](https://bqst.github.io).

## Setup

To run this project locally, you will need to have [Ruby](https://www.ruby-lang.org/en/) and [Bundler](https://bundler.io/) installed. Once you have those, run the following commands:

```bash
git clone
cd bqst.github.io
bundle install
bundle exec jekyll serve
```

## Adding a new post

To add a new post, create a new file in the `_posts` directory. The file name should be in the format `YYYY-MM-DD-title-of-post.markdown`. The file should start with the following header:

```markdown
---
layout: post
title:  "Title of post"
date:   YYYY-MM-DD HH:MM:SS +0100
categories: [category1, category2]
tags: [tag1, tag2]
---
```

The rest of the file should be written in [Markdown](https://www.markdownguide.org/).

## License

This project is licensed under the [MIT License](LICENSE).

## Acknowledgements

This project was built using [Jekyll](https://jekyllrb.com/), [GitHub Pages](https://pages.github.com/), and [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/).

## Contact

If you have any questions, feel free to [contact me](https://bqst.github.io/contact/).
