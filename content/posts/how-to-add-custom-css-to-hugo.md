---
title: "How to Add Custom Css to Hugo"
date: 2023-03-15T13:59:05-04:00
draft: false
---

1. Create a CSS file under `static/css/`.
2. Add your styles.
3. Figure out which partial file name is used by your theme and create a file with the same name under `layouts\partials` in the site root. For this site, it is `themes\ananke\layouts\partials\head-additions.html`.
4. In `config.toml`, add the file paths to `custom_css` under `[params]
     ```toml
   [params]
   custom_css = ["css/styles.css"]
     ```
5. In the partial file, add
     ```html
   {{ range .Site.Params.custom_css -}}
   <link rel="stylesheet" href="{{ . | absURL }}">
   {{- end }}
     ```
6. Use the styles.