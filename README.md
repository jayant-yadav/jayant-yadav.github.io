# DigiDiary

A place to put my thoughts onto the world. No GPT needed.

### Deployment

To run the theme locally, navigate to the theme directory and run `bundle install` to install the dependencies, then run `jekyll serve` or `bundle exec jekyll serve` to start the Jekyll server.

I would recommend checking the [Deployment Methods](https://jekyllrb.com/docs/deployment-methods/) page on Jekyll website.

### Dir structure

Directory	Role
_posts	Jekyll's special built-in collection for blog posts
_works	A custom collection (defined in _config.yml)
_pages	Another custom collection for static pages
_layouts	HTML templates/wrappers
_includes	Reusable HTML partials (like components)
_sass	SCSS source files, compiled into CSS
_data	YAML/JSON data files, accessible as site.data.*
_site	Jekyll's build output — the actual generated website. Never edit this manually.

Page flow: 
_posts/2025-09-04-my-post.md   ← you write here
        ↓ Jekyll processes
_site/blog/my-post/index.html  ← generated output, served at /blog/my-post

blog/index.html                ← listing page, served at /blog

Note to myself: write posts in _posts, the blog dir is just the index/listing page.