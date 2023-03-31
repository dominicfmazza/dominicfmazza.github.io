---
layout: page
title: AVC Wiki
description: Documentation Website for MSU's Autonomous Vehicle Club
importance: 1
img: assets/img/wiki_project/project_image.png
category: work
---

## Overview

As a part of my responsibilities as Project Lead for MSU's AVC, I wanted to make sure that our
documentation system is up-to-date and as informative as it can be. AVC uses ROS to develop our vehicle
code, and while I am very familiar with it, I know that new members cannot be expected to develop
that same level of familiarity on their own.

To faciliate that, I wrote a bunch of wiki entries for our wiki on MSU's own self-hosted GitLab instance,
but there were multiple areas where I felt the wiki could be improved/extended to provide better
navigation and ease of use. Rather than try to extend GitLab's [gollum](https://github.com/gollum/gollum)
based wiki, I wanted to use a static site generator to generate the website from markdown, similar to my own
personal website (this one). After trying out multiple options, I decided on using [retype](https://retype.com/),
which provides a no-frills way to create and update a wiki using only markdown and .yml.

The resulting product is the [AVC Wiki](https://msuavc.github.io/avc_wiki/)!

## Implementation

As retype is primarily markdown-based, it was quite trivial to port markdown files over to the new system,
and from there I used retype's specific features to jazz it up a bit. Firstly, I aligned the wiki to use our own logos
and branding, as well as organized the sidebar to be easy to navigate:

<div class="container">
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/wiki_project/branding_corner.png" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm">
      {% include figure.html path="assets/img/wiki_project/sidebar.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
</div>
<div class="caption">
  Branding and collapsed sidebar.
</div>

I then got into the meat and bones of the overhaul, retype's components. Retype's components allow you to spice up
the presentation of everything.

Firstly, I went through and made sure to properly display alerts for important information:

<div class="container">
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/wiki_project/alert_1.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/wiki_project/alert_2.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/wiki_project/alert_3.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
</div>
<div class="caption">
  Examples of alerts used in the wiki.
</div>

I also cleaned up the linking of pages to each other, and made sure that the information displayed was relevant to our build process.

Additionally, I made sure to use retype's own syntax highlighting to ensure that the code snippets displayed
for example purposes were cleanly highlighted and easy to read.

<div class="container">
  <div class="row">
    <div class="col-sm">
      {% include figure.html path="assets/img/wiki_project/code_snippet.png" class="img-fluid rounded z-depth-1" %}
    </div>
  </div>
</div>

## Final Thoughts

Overall, I want to let the [wiki](https://msuavc.github.io/avc_wiki/) speak for itself! I'm quite proud of it, as it turned out very
clean and much easier to navigate than the GitLab wiki.
