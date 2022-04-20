---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: splash
header:
  image: /assets/images/banner.jpg
  og_image: /assets/images/E.png
intro:
  - excerpt: 'Empora Title Engineering is a career changing organization of first class engineering talent.'
feature_row1:
  - image_path: /assets/images/image_1.png
    alt: "Empora Engineering"
    title: "An Introduction to Empora Engineering"
    excerpt: 'We prize ownership, understanding, and long-term productivity in our team.'
    url: "/introduction"
    btn_label: "Read More"
    btn_class: "btn--primary"
feature_row2:
  - image_path: /assets/images/image_2.png
    alt: "Blogs"
    title: "Our Engineering Blog"
    excerpt: 'We think carefully about the changes we make, and then we share them!'
    url: "/posts"
    btn_label: "Our Posts"
    btn_class: "btn--primary"
---

{% include feature_row id="intro" type="center" %}

{% include feature_row id="feature_row1" type="left" %}

{% include feature_row id="feature_row2" type="right" %}

