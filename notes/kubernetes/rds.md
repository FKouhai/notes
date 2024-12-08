---
id: rds
aliases: []
tags:
  - aws
---

## Create db in rds
RDS -> Create Database
Standard Create -> PostgresSQL
Free Tier
DB instance identifier: expatx-feed-staging
Self managed -> Auto generate password
Public Access
VPC Security Groups: Access Postgres Anywhere
Additional Configuration at the bottom-Initial Database Name: expatx_feed_staging
Copy Password after you create the database
AWS Secrets Manager -> Feed Service -> Retrieve Value -> Edit
Add new password to FEED_DB_SOURCE


