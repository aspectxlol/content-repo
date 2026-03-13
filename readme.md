# content-repo

haha you found it. this is where i keep all my blog posts -- using github as a headless CMS because it's free, has unlimited storage, and comes with version control built in. why would i pay for anything else.

the portfolio fetches posts from here at build time via the github contents API.

---

## folder structure

```
content-repo/
└── post/
    ├── my-post-slug.md
    ├── another-post.md
    └── assets/
        ├── my-post-slug/
        │   └── cover.jpg
        └── another-post/
            └── cover.jpg
```

posts go in `/post`. images go in `/post/assets/<slug>/`. that's it.

---

## frontmatter

every post needs a frontmatter block at the top. here's the full spec:

```yaml
---
title: "Your Post Title"
excerpt: "One sentence that describes the post. Shows up on the blog card."
coverImage: "/assets/your-post-slug/cover.jpg"
date: "2026-03-13T05:35:07.322Z"
tags: ["Tag One", "Tag Two"]
---
```

| field | required | notes |
|---|---|---|
| `title` | yes | shows as the page `<h1>` and og:title |
| `excerpt` | yes | blog card subtitle and og:description |
| `coverImage` | yes | path relative to `/post` -- starts with `/assets/...` |
| `date` | yes | ISO 8601 format, used for display and og:published_time |
| `tags` | no | array of strings, renders as cyan badges on the post page |

---

## adding a new post

1. create a new file in `/post/` -- the filename becomes the slug

   ```
   post/my-new-post.md
   ```

2. add the frontmatter block at the top (copy the template above)

3. if you have a cover image, drop it in `/post/assets/my-new-post/cover.jpg` and make sure `coverImage` in the frontmatter matches

4. write the post in markdown below the frontmatter

5. commit and push -- the portfolio will pick it up on next build/deploy

that's genuinely all there is to it.