---
layout: post
title: "PostgreSQL: Scheduling a Task with pg_cron"
date: 2023-06-17 14:45:10 +0100
categories: [database, web development]
tags: [postgresql, pg_cron, scheduling, task]
author: bqst
summary: "How to use pg_cron to schedule tasks in PostgreSQL."
---

In any application architecture, the ability to efficiently manage resources is pivotal. One common scenario requiring careful resource management is when a specific user is working on a task, such as editing a blog post, and you want to prevent other users from accessing the same resource simultaneously. This requirement, for example, may come up in a Content Management System (CMS) where multiple users have access to edit articles.

To avoid potential conflicts or data inconsistencies, the solution would be to 'lock' the resource, making it accessible only to the user currently interacting with it for a specific duration. If the user performs no action on the resource within this timeframe, the resource 'unlocks,' making it accessible to others. This use case led me to explore the potential of pg_cron, a [PostgreSQL](https://www.postgresql.org/) extension that schedules tasks such as this one.

## What is pg_cron?

`pg_cron` is an [open-source job scheduling extension for PostgreSQL](https://github.com/citusdata/pg_cron), created by Citus Data, a subsidiary of Microsoft. With pg_cron, you can schedule PostgreSQL commands directly from your database. The extension uses the same syntax as the Unix 'cron' job scheduler, which simplifies its usage for those familiar with Unix-like systems. It provides an effective method to schedule tasks like periodic data aggregation, data reporting, and indeed, the automated locking and unlocking of resources, as in our scenario.

## How to Set Up pg_cron

Before you can use pg_cron, you need to install it on your PostgreSQL server. As of PostgreSQL 9.5, you can add pg_cron to your shared_preload_libraries in your postgresql.conf file, and then restart your PostgreSQL server.

Then, to enable the extension, run the following SQL command in your PostgreSQL database:

```sql
CREATE EXTENSION pg_cron;
```

Remember, only superusers or users with the necessary privileges can add extensions.

## Setting Up the Database

Before we start locking resources, we need to set up the `blog_posts` table. This table will contain the following fields: `id`, `content`, `is_locked`, and `locked_until`. The `is_locked` field indicates if the post is currently locked, and the `locked_until` field stores the timestamp until which the post is locked.

Here's a basic SQL command to create this table:

```sql
CREATE TABLE blog_posts (
    id SERIAL PRIMARY KEY,
    content TEXT,
    is_locked BOOLEAN DEFAULT false,
    locked_until TIMESTAMP
);
```

## Using pg_cron for Resource Locking

Let's consider a simple example where we have a blog_posts table with the fields `id`, `content`, `is_locked` and `locked_until`. The `is_locked` field indicates if the post is currently locked, and the `locked_until` field stores the timestamp until which the post is locked.

Now, let's say you want to lock a post for a duration of one hour. You'd first update the post to set `is_locked` to `true` and `locked_until` to the current time plus one hour.

The SQL query might look like this:

```sql
UPDATE blog_posts 
SET is_locked = true, 
    locked_until = NOW() + INTERVAL '1 hour' 
WHERE id = <post_id>;
```

Here, replace `<post_id>` with the id of the blog post.

Now, to automate the unlocking process, you can schedule a cron job with pg_cron to run every 15 minutes. This job would set `is_locked` back to `false` and `locked_until` to `NULL` for this post if the current time is past the `locked_until` timestamp.

To add this job, you'd run a command like:

```sql
SELECT cron.schedule('*/15 * * * *', $$UPDATE blog_posts SET is_locked = false, locked_until = NULL WHERE id = <post_id> AND locked_until <= NOW()$$);
```

In this command, replace `<unique_job_name>` with a unique name for this job, and `<post_id>` with the id of the blog post.

## List All Scheduled Jobs

To list all scheduled jobs, you can run the following command:

```sql
SELECT * FROM cron.job;
```

## Unschedule a Job

To unschedule a job, you can run the following command:

```sql
SELECT cron.unschedule('<unique_job_name>');
```

## Conclusion

Using pg_cron to schedule resource locks in PostgreSQL is a powerful and flexible solution. It simplifies managing resource availability, especially in multi-user environments. With pg_cron, you have granular control over when tasks are executed and can easily automate routine database operations. However, as with any tool, proper usage and understanding are essential to

## References

- [pg_cron GitHub repository](https://github.com/citusdata/pg_cron)
- [supabse pg_cron documentation](https://supabase.com/docs/guides/database/extensions/pgcron)
- [supabase blog post on pg_cron](https://supabase.com/blog/postgres-as-a-cron-server)
- [cron schedule syntax editor](https://crontab.guru/)
