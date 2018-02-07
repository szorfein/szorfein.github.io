---
layout: post-detail
title: Configure Chrome
date: 2018-02-07
categories: chrome
description: How configure web browser based on chrome, work for chromium, vivaldi...
img-url: https://www.freepnglogos.com/uploads/google-chrome-png-logo/google-chrome-review-ebooks-png-logo-34.png
comments: true
---

We configure web browser based on chrome to enforce privacy and security with tor.  
Once installing, go to URL bellow:  

+ chrome -> chrome://settings/content
+ vivaldi -> vivaldi://settings/content

It will open thing like this: 
![content](/assets/imgs/vivaldi_01.jpg "chrome setting 1")

Next, you block every options, like the both screenshots.
![content](/assets/imgs/vivaldi_02.jpg "chorme settings 2")

Concerning javascript, you have too allow only site than you like visit.
![content](/assets/imgs/vivaldi_03.jpg "chrome settings javascript")

For cookies, we remove them when we close the browser:
![content](/assets/imgs/vivaldi_05.jpg "chrome cookie")

Security & privacy, only enable `Do not track`:
![content](/assets/imgs/vivaldi_06.jpg "security & privacy")

Another nice option, my browser often crashed without. You have to enable `strict site isolation`.
Go to `chrome://flags/#enable-site-per-process` and enable this feature.

![content](/assets/imgs/vivaldi_04.jpg "strict site isolation")

### Troubleshooting

Post an issue to [github](https://github.com/szorfein/szorfein.github.io/issue).
