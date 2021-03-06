---
layout: post
title: "Benchmarking and Testing Database Performance in a Rails 6 App with Artillery"
date: 2021-03-06 08:54
image: '/assets/img/'
description: 'Trial Run of Post to Test Things Out'
tags:
- jekyll
- template
categories:
- Blogging
twitter_text: 'Check out the first post'
---

Working on a recent side project, I ran into an opportunity to benchmark our app's database performance so that we could measure the effect of applying several indexes to the main table that holds our data. Specifically, I wanted to test how fast our backend API responded to a sample set of queries. The backend API is a Rails application running on Rails 6.1 and the database is PostgreSQL, both of which are deployed on Heroku.