# blog

These are files for my public blog that's on [https://blog.barbetta.me](https://blog.barbetta.me)

## Hugo

Hugo takes static Markdown files and automatically converts them into `html` documents. The `.md` files are located under content. The `static` folder contains images and stylesheets.

### `config.toml`

Contains the actual Hugo settings for the blog, including the theme.

### `nginx.conf`

Hugo itself doesn't serve web content, so that is handled by a barebone `nginx` container which is running on port `1313`. This sits behind a reverse proxy that handles TLS termination.
