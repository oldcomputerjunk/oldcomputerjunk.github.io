# How to add a blog entry

1. Create a new file in the [`posts/`](_posts/) directory
2. Filename is `YYYY-MM-DD-article.markdown`
3. File has following top matter:
    ```
    ---
    layout: post
    slug: lower-hyphen-case-name
    title:  "Sentence title"
    date:   YYYY-MM-DD 01:00:00
    categories:
    - tech
    tags:
    - javascript
    - html5
    - mapbox
    ---
    ```

4. Note that date might affect publishing...
5. For convention `article` in the filename should match the value of `slug` in the YAML
6. Push to github

Links:

[https://help.github.com/articles/using-jekyll-as-a-static-site-generator-with-github-pages/]()

[http://stackoverflow.com/questions/24098792/how-to-force-github-pages-build]()

[https://pages.github.com/]()

[https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet#tables]()

[https://gist.github.com/clintel/1155906/71b2ab2e4444cb826b45f9988d067d1dd1edcc2c]()

Emoji: [https://gist.github.com/roachhd/1f029bd4b50b8a524f3c]() and [https://gist.github.com/rxaviers/7360908]()
