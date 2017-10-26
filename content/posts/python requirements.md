---
title: "Python Requirements"
date: 2017-10-25T18:49:22+05:30
tags: ["python"]
draft: true
---

In python for dependencies we have a requirements.txt and with pip we can install it `pip3 install -r requirements.txt`
and you can populate this file with `pip3 freeze >> requirements.txt`.  
At the time of the post I needed [https://github.com/heroku/salesforce-bulk](https://github.com/heroku/salesforce-bulk) and pk-chunking feature was not a part of the release just a merged PR 5 days ago. So I learned today that you can install with pip by specifying  `commit hash, branch name, tag.`  

###### hash
`$ pip install git+git://github.com/aladagemre/django-notification.git@2927346f4c513a217ac8ad076e494dd1adbf70e1`

###### branch-name
`$ pip install git+git://github.com/aladagemre/django-notification.git@cool-feature-branch`  
or from source
`$ pip install https://github.com/aladagemre/django-notification/archive/cool-feature-branch.tar.gz`

###### tag
`$ pip install git+git://github.com/aladagemre/django-notification.git@v2.1.0`


But still the pip freeze did not add with hash in requirements.txt file
The solution to that was
`-e git+https://github.com/heroku/salesforce-bulk.git@e106adaa0b673aabe7d62caf2cc2ab823bc7a021#egg=MyProject`
