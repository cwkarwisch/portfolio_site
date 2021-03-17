---
layout: post
title: "Benchmarking and Testing Database Performance in a Rails 6 App with Artillery"
date: 2021-03-09 12:40
image: '/assets/img/'
description: "On a recent project, I ran into an opportunity to benchmark our app's database performance. Specifically, I wanted to measure the effect of an upcoming change to our database's schema, which involved applying several indexes to a key table that was being frequently read and searched through. On a practical level, I wanted to test how fast our backend API responded to a sample set of queries where the majority of the time spent building a response to the query involved searching through the database to find the requested entries."
tags:
- Rails
- PostgreSQL
- database performance
- artillery
- performance testing
categories:
- Testing
---

# Benchmarking and Testing Database Performance in a Rails 6 App with Artillery

## Overview of Project

On a recent project, I ran into an opportunity to benchmark our app's database performance. Specifically, I wanted to measure the effect of an upcoming change to our database's schema, which involved applying several indexes to a key table that was being frequently read and searched through. On a practical level, I wanted to test how fast our backend API responded to a sample set of queries where the majority of the time spent building a response involved searching through the database to find the requested entries.

The application in question is Baby-Namr, which allows expectant parents to search through information on the historical popularity of baby names in the United States. As examples, the site allows users to search for the 100 most popular female names of 2019, see a visual representation of the historical popularity of the name David, or look at the most common names of babies born in the 1960s. The backend of Baby-Namr is an API built on Rails 6.1, connected to a PostgreSQL database, both of which are deployed on Heroku. The API provides JSON to a React frontend. If you'd like to take a look at the project, [here](http://baby-namr.herokuapp.com/) is a live version of the frontend, and [here](https://github.com/cwkarwisch/baby_namer_api) is a link to the backend's source code.

## Database Schema and Size

In this project, one of the key tables in the database is the `names` table. The `names` table contains records on the popularity of names from 1880 to 2019. The schema for the `names` table looks like the following:

| Column Name | Type |
| ----------- | ----------- |
| id      | integer       |
| name      | string       |
| sex   | string        |
| count      | integer       |
| popularity   | integer        |
| year      | integer       |
| country   | string        |
| created_at      | string       |
| updated_at   | string        |

This table holds 279,877 rows. It's not a huge table, but it's not insignificant either if we need to search through the entire database to respond to a query from the frontend. When I initially created this table, I didn't add any additional indexes, so the rows in the table were only indexed by the table's primary key, the `id` for each name. Since our API is frequently handling requests with several query parameters such as `/names?year=2019&sex=F&popularity=100` or `/names?name=David`, the `id` column of each row doesn't provide a helpful way to speed up searches through the database. For the sake of clarity, those last two example URLs are requests for the top 100 female names of 2019 and information on the popularity of David for every year in the dataset. Because the `names` table wasn't indexed by any of the columns that we used to search for entries that matched the relevant request, our app had to read through the entire database in the process of responding to any given request (ignoring for now the possibility of setting up a cache for popular queries).

## Read Heavy

Since the `names` table is very read heavy, we wanted to apply as many indexes as would be helpful in speeding up searches of the table. At most, new entries will be added to the `names` table once a year, when the Social Security Administration releases data on the most recent year's names, so we are not concerned about slowing down writes to this table.

## Performance Before Indexing

To get a sense of how our very naive database schema was performing, I used artillery to benchmark the API's performance when responding to a variety of different requests. Artillery is a "modern, powerful & easy-to-use performance testing toolkit." Artillery was a good choice for testing our API since artillery is specifically designed for testing backend systems. Even though artillery is written in JavaScript and installed via NPM, Artillery is language agnostic when it comes to testing a backend because it communicates with the application via HTTP.

Although there are many situations where you would want to do this kind of performance testing in a test or development environment, I wanted to run this particular test against our production environment. There were a few reasons for that decision. The main reason is that our production environment contains the full dataset that the `names` table needs to be able to respond to requests appropriately. In our test environment, in contrast, the test database is only populated with a small subset of our total data. Since the test database is created and populated each time we run the test suite, running our tests would become significantly slower if the database was created, filled with 279,877 rows in the `names` table, and torn down each time we ran our tests. Accordingly, using the test environment to benchmark our database performance would not provide an accurate picture of the time it takes our system to respond to requests since our backend would be searching through a much smaller dataset than is used in production.

Unlike the test environment, the database in our development environment does contain a fully populated `names` table. Since the development database was fully populated, it would have been reasonable, and likely best, to run the benchmark tests in the development environment. The development environment is likely the best choice for this kind of testing since it controls for network latency inherent in testing the production environment that isn't relevant in benchmarking how fast the database is performing. I'll paste the actual artillery test I used below, and to test in the development environment, rather than the production environment, the `target` URL should be updated to use `localhost:3000` in the case of a Rails app or whatever port your localhost is listening on.

While doing this sort of database benchmark testing is likely best done against the development environment, I did want to get an absolute sense of how the database was performing in production, so I decided to run the artillery tests against our deployed backend. I was comfortable doing this since the specific requests being sent in the test are purely `GET` requests that are asking for the server to send data. No updates or deletion of data are occurring in these tests, so there wasn't a concern about altering production data. This is also a small personal side project, so there wasn't a concern that these tests would degrade the deployed app's performance for actual users using the site. Accordingly, I made decision to test our production environment.

Running the artillery test pasted below twice, without any indexes in our `names` table other than for each row's `id` column, the median response time for a request was 3.5s and 4.5s, which is pretty abysmal. As written below, artillery will send 10 requests for each endpoint provided. Because we have listed four separate `GET` requests that we want to test, and provided query parameters after the route, the test script below will send 40 total requests.

```yml
config:
  target: "https://baby-namer-api.herokuapp.com" # replace with whatever resource name you want to test
  phases:
    - duration: 10
      arrivalRate: 1
scenarios:
  - flow:
      - get:
          url: "/names?year=1960"
      - get:
          url: "/names?year=2015"
      - get:
          url: "/names?name=Clay&Sex=M"
      - get:
          url: "/names?name=Camille&yearStart=1950&yearEnd=1999"
```

To run the test script, we need artillery installed, which can be done with `npm install -g artillery@1.6`, which will install version 1.6 of artillery globally on your system (this assumes you already have npm installed). While this test should ideally be baked into a rake task, the test can be run simply by executing the following shell command: `artillery run --output ./test/performance/report.json ./test/performance/artillery_performance_test.yaml`. The previous command assumes you want to save a copy of the test's report (indicated by the `--output` flag) as `report.json` to the `test/performance` directory of your rail's repository. The specific test script to be run in the case of the previous shell command is located at `/test/performance/artillery_performance_test.yaml`. Once the JSON version of the report has been generated, we can use artillery to convert the JSON to an easy to read HTML report that can be opened in your browser. To generate an HTML version of the report, simply run the following command: `artillery report --output ./test/performance/report.html ./test/performance/report.json`.

## Performance After Indexing

As we saw from the incredibly slow 3.5 and 4.5 second response times to our tests, our database was in fact desperately in need of some indexes to speed up searches. Thankfully, adding indexes to every column that we search by in our `names` table is straight forward in rails. After running `rails generate migration AddIndexesToNames` to generate a blank database migration, the actual migration itself only needed a few lines of code to add four new indexes:

```ruby
class AddIndexesToNames < ActiveRecord::Migration[6.1]
  def change
    add_index :names, [:name, :sex, :popularity, :year]
  end
end
```

After deploying our updated code, we need to run the database migration with `rails db:migrate` to add the indexes to our `names` table. After doing so, we can re-run our artillery test script posted above and generate new test results. After doing so on Baby-Namr and running the tests twice, the median response time came down to 662ms and 402 ms. While this is still not amazing performance, it is a vast improvement over the 3.5 and 4.5 second response times we were receiving without the indexes.

If you want to play around with Baby-Namr yourself, make sure to check out the deployed version of the React frontend [here](http://baby-namr.herokuapp.com/), or take a look at the github repo for the backend [here](https://github.com/cwkarwisch/baby_namer_api).
