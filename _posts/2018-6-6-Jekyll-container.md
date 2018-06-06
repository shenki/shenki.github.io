---
layout: post
title: Jekyll container
---

This blog uses [jekyll-now](https://github.com/barryclark/jekyll-now), which I found in a search for a method to have a
simple blog. The highlight is it provides instructions for you to create and
edit the blog using the Github interface.

After not long, I was attempting to track down an issue with the stylesheet
where it would but two drop shadow boxes around my code snippets. To be able
to test locally, I'd have to install Ruby so I could run a ruby gem, which I
didn't want to do on this machine.

It turns out you can do it all in a container:

```
$ docker pull jekyll/builder
$ docker run --rm  --volume="$PWD:/srv/jekyll" -p 4000:4000 -it jekyll/builder \
     jekyll serve --watch
```

And then point your browser at localhost:4000, ta-da!
