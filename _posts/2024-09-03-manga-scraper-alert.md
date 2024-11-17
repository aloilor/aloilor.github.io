---
title: "Track new manga releases in Italy"
tags:
  - AWS
  - TDD
  - Manga Alert Italia
---

Hey there! I'm going to build something I always needed and that I could't ever find on the internet: an email alert application that notifies me whenever a new physical release of the mangas I'm following is going to come out soon. This app is going to scrape a set of predefined websites (only the official publishers of the italian releases of those mangas) and send an email to every address in the database with the release date of the new volume. 

# Language: Python
This is a forced choice: Python is hands down the best programming language for web scraping. 

# Scraping: requests
Since I'm going to be making simple GET requests to the publishers' websites and they don't seem to block me if I use Python's library ```requests```, there is really no need for me to use more complex and advanced tools like curl-cffi or Scrapy. I'm also going to be using BeatifulSoup to parse the HTML pages I will get. 

# TDD: pytest and unittest.mock
I'm going to be following a Test Driven Development approach, using pytest and unittest.mock to properly test every functionality of the application. I will aim to have at least 85% code coverage.

# Database: SQL 
No need to use a NoSQL database, I'll use a relational database with something like PostgreSQL, MySQL or SQLite as DBMS. The following are the tables I'm probably going to need to setup the application: 

**Subscribers table - subscribers**
- id (Primary Key)
- email_address (string)
- subscriptions (string) -- NORMALIZED IN ANOTHER TABLE

**Manga Releases table - manga_releases**
- id (Primary Key)
- manga_title (string)
- volume_number (string)
- release_date (date)
- publisher (string)
- alert_sent (boolean, to track whether an alert has already been sent for this release)

**Subscribers' subscriptions table -- subscribers_subscriptions**
- subscriber_id (integer)
- manga_title (string)

I decided to normalize the ```subscriptions``` field from the table ```subscribers``` to make everything cleaner, and also to make queries more efficient: at each new manga release I'm going to query the database searching for subscribers to that manga title so having a whole new column with the manga titles associated to a subscriber will make thing a lot cleaner and easier. 

# System design: serverless event-driven vs server-based polling system
You could argue that a classic server-based polling system would be highly inefficient compared to a serverless event-driven approach, that uses AWS RDS events to trigger email alerts only when changes on the Database happen; and you are 100% right. In the classic approach the EC2 instance would be waiting for most of the time and would poll the database every [x] (probably 6 or something, I'm still not sure) hours, but still I'm going to use this approach for two main reasons: I want to learn how to deploy and orchestrate Docker containers using AWS ECS and also I already know how to use AWS Lambdas in production environments so the serverless approach wouldn't be really interesting from a "learning perspective".

# cron, Docker and ECS
I'm going to build both the scraper and the email alerter as cron jobs (ECS Scheduled Task) that run on Docker container through AWS ECS. I'll probably write another blog post on that, since it's going to be quite a long task.

# TO BE CONTINUED - Logging and monitoring: AWS CloudWatch and Python's logging module

