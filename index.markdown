---
title: "主页"
layout: splash
permalink: /
date: 2016-03-23T11:48:41-04:00
header:
  overlay_color: "#000"
  overlay_filter: "0.2"
  overlay_image: /assets/images/unsplash-image-1.jpg
  actions:
    - label: "文章汇总"
      url: /year-archive/
  
excerpt: "种一棵树最好的时间是十年前，其次是现在"

feature_row:
  - image_path: assets/images/15445pro0lowcp.jpg
    alt: "placeholder image 1"
    title: "CMU 15-445 Project0 总结"
    url: "https://sonicnapalm.xyz/CMU-15445-project0/"
    btn_label: "阅读"
    btn_class: "btn--primary"

  - image_path: assets/images/15445pro11lowcp.jpg
    alt: "placeholder image 1"
    title: "CMU 15-445 Project1 总结"
    url: "https://sonicnapalm.xyz/CMU-15445-project1/"
    btn_label: "阅读"
    btn_class: "btn--primary"

  - image_path: assets/images/project2/1lowcp.jpg  
    alt: "placeholder image 1"
    title: "CMU 15-445 Project2 总结"
    url: "https://sonicnapalm.xyz/CMU-15445-project2/"
    btn_label: "阅读"
    btn_class: "btn--primary"
#  - image_path: /assets/images/unsplash-gallery-image-2-th.jpg
    
#    alt: "placeholder image 2"
#    title: "Placeholder 2"
#    excerpt: "This is some sample content that goes here with **Markdown** formatting."
#    url: "#test-link"
#    btn_label: "Read More"
#    btn_class: "btn--primary"
#  - image_path: /assets/images/unsplash-gallery-image-3-th.jpg
#    title: "Placeholder 3"
#    excerpt: "This is some sample content that goes here with **Markdown** formatting."
#feature_row2:
##  - image_path: /assets/images/unsplash-gallery-image-2-th.jpg
#    alt: "placeholder image 2"
#    title: "Placeholder Image Left Aligned"
#    excerpt: 'This is some sample content that goes here with **Markdown** formatting. Left aligned with `type="left"`'
#    url: "#test-link"
#    btn_label: "Read More"
#    btn_class: "btn--primary"
#feature_row3:
#  - image_path: /assets/images/unsplash-gallery-image-2-th.jpg
#    alt: "placeholder image 2"
#    title: "Placeholder Image Right Aligned"
#    excerpt: 'This is some sample content that goes here with **Markdown** formatting. Right aligned with `type="right"`'
#    url: "#test-link"
#    btn_label: "Read More"
#    btn_class: "btn--primary"
#feature_row4:
#  - image_path: /assets/images/unsplash-gallery-image-2-th.jpg
#    alt: "placeholder image 2"
#    title: "Placeholder Image Center Aligned"
#    excerpt: 'This is some sample content that goes here with **Markdown** formatting. Centered with `type="center"`'
#    url: "#test-link"
#    btn_label: "Read More"
#    btn_class: "btn--primary"
---

{% include feature_row id="intro" type="center" %}

{% include feature_row %}

{% include feature_row id="feature_row2" type="left" %}

{% include feature_row id="feature_row3" type="right" %}

{% include feature_row id="feature_row4" type="center" %}
