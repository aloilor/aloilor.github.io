---
title: "Security and billing concerns over Internet access"
tags:
  - AWS
---

Hey there! In this post I'm going to talk about the security concerns I encountered when designing the main backend and frontend of the application (especially the frontend part which allowed user input). 

# Prepared statements to prevent SQL Injections
Not much to talk about this, it's been known since forever that prepared statements prevent SQL Injections: they allow an application to precompile SQL statements before executing them with actual data. In other words, using them allows us to separate the actual SQL Logic from the data. 

# 