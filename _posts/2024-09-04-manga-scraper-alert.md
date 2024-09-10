---
title: "Track new manga releases in Italy"
tags:
  - AWS
  - TDD
---

Hey there! For the actual app development part, I'm going to build something I always needed and that I could't ever find on the internet: an email alert application that notifies me whenever a new physical release of the mangas I'm following is going to come out soon. This app is going to scrape a set of predefined websites (only the official publishers of the italian releases of those mangas) and send an email to every address in the database with the release date of the new volume. 

# Language: Python
This is a forced choice: Python is hands down the best programming language for web scraping. 

# Scraping: requests
Since I'm going to be making simple GET requests to the publishers' websites and they don't seem to block me if I use Python's library ```requests```, there is really no need for me to use more complex and advanced tools like curl-cffi or Scrapy. I'm also going to be using BeatifulSoup to parse the HTML pages I will get. 

# TDD: pytest and unittest.mock
I'm going to be following a Test Driven Development approach, using pytest and unittest.mock to properly test every functionality of the application. I will aim to have at least 85% code coverage.

# Database: SQL 
No need to use a NoSQL database, I'll use a relational database with something like PostgreSQL or SQLite as DBMS. The following are the tables I'm probably going to need to setup the application: 

**Subscribers table - subscribers**
- id (Primary Key)
- email_address (string)

**Manga Releases table - manga_releases**
- id (Primary Key)
- manga_title (string)
- volume_number (string)
- release_date (date)
- publisher (string)
- alert_sent (boolean, to track whether an alert has already been sent for this release)

# System design: serverless event-driven vs server-based polling system

# TO BE CONTINUED 