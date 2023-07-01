---
layout: post
title: "Supabase : User Authentication and Data Access in NextJS / NuxtJS"
date: 2023-07-01 12:00:10 +0100
categories: [database, web development]
tags: [supabase, nextjs, nuxtjs]
author: bqst
summary: "How I use Supabase to handle user authentication and data access in my NextJS and NuxtJS projects."
---

Recently, I've been harnessing the power of [Supabase](https://supabase.com/) to scaffold my latest minimum viable products (MVPs) using [NextJS](https://nextjs.org/) or [NuxtJS](https://nuxt.com/). To say the least, I've been thoroughly impressed with both the range of features and DX it offers. Supabase is an open-source Firebase alternative. It provides an instantaneous, real-time backend service that uses Postgres as its primary database. It also offers a range of other features, including authentication, storage, and serverless functions. You can find out more about Supabase [here](https://supabase.com/docs/).

## What I need to do

I plan to set up a classic `Posts` table including a title and content that belongs to a `User`. To establish a link between the user who creates a post and the post itself, I will utilize a foreign key `user_id` in the `Posts` table. This will allow me to retrieve all posts created by a specific user. However, I also want to display the user's name and profile picture alongside each post. This is where things get tricky.

The `auth.users` table [isn't accessible via the Supabase API](https://supabase.com/docs/guides/auth/managing-user-data). To circumvent this, I initially attempted to create a **view** `public.users`, replicating `auth.users`.

```sql
create view public.users as select * from auth.users;
revoke all on public.users from anon, authenticated;
```

However, this approach failed as the **joins didn't function through the API**. 

## Solution

As a solution, I ended up creating a `public.users` **table** that contains fields which are readable. I followed this up with the creation of associated functions and triggers. This new arrangement allows my applications to have direct and secure access to the necessary user data while still maintaining the integrity of the data.

### Setting up the Database

To establish a relationship between our users and posts, we need to create corresponding tables in our public schema. We'll also create a trigger function that handles new user sign-ups. Here's the SQL code for setting this up:

```sql
-- USERS
create table public.users (
  id          uuid not null primary key, -- UUID from auth.users
  username    text,
  email       text
);
comment on table public.users is 'Profile data for each user.';
comment on column public.users.id is 'References the internal Supabase Auth user.';

-- POSTS
create table public.posts (
  id uuid default uuid_generate_v4() primary key,
  user_id uuid not null references public.users,
  title text not null,
  description text,
  created_at timestamp with time zone default now()
);

comment on table public.posts is 'Posts created by users.';

-- Inserts a row into public.profiles
create function public.handle_new_user() returns trigger language plpgsql security definer
set
  search_path = public as $$ begin
insert into
  public.users (id, username, email)
values
  (new.id, split_part(new.email, '@', 1), new.email);
return new;
end;
$$;

-- Trigger the function every time a user is created
create trigger on_auth_user_created
after
insert
  on auth.users for each row execute procedure public.handle_new_user();

```

The `users` table is designed to hold profile data for each user, with the `id` field referencing the internal Supabase Auth user. The `posts` table, on the other hand, stores the posts created by users.

The `handle_new_user()` function inserts a row into `public.users` every time a new user signs up, using the user's email id and a generated username based on the email.

The trigger `on_auth_user_created` is set up to call `handle_new_user()` every time a user is created.

With these tables and functions in place, we can handle user sign-ups and establish a relationship between users and their posts.

## Setting up the API

Now that we have the database set up, we can create the API to handle user authentication and data access. We'll be using NuxtJS for this example, but the same principles apply to NextJS. Here is my composable for the Supabase client:

```javascript
import type { Database } from 'types/Supabase'

export const usePost = () => {
  const config = useRuntimeConfig()
  const supabase = useSupabaseClient<Database>()

  const getPosts = async () => {
    try {
      return await supabase.from('posts')
        .select(`*, user:user_id(*)`)
    } catch (error) {
      console.log(`Error getting posts: ${error}`)
      return null
    }
  }

  return {
    getPosts,
  }
}
```

Now, you can get user.email and user.username from the `user` object returned by `getPosts()` :

```javascript
const { getPosts } = usePost()
const posts = await getPosts()

posts.map(post => {
  console.log(post.user.email)
  console.log(post.user.username)
})
```

To conclude, despite the initial hiccups, Supabase has proven to be an excellent choice for quick and secure scaffolding of my MVPs in NextJS and NuxtJS. The authentication and data access features it offers are commendable, making it a prime tool in my developer arsenal. I look forward to discovering and sharing more insights and best practices regarding Supabase in the future.
