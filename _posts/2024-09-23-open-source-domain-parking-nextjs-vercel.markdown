---
layout: post
title: "I Open-Sourced My Domain Parking App in Next.js—Already Sold a Domain!"
date: 2024-09-23 12:00:00 +0000
categories: [web development, dns]
tags: [nextjs, vercel]
author: bqst
summary: "I open-sourced my Next.js domain parking app, allowing users to easily create landing pages for domains and connect with potential buyers—already sold one domain using it!"
---

I’ve just open-sourced my project for parking domain names using **Next.js**. This app makes it simple to create a landing page for any domain and connect with potential buyers interested in acquiring your domain name. The best part? One domain is already sold using this setup, so I know firsthand that it works!

## Why Build a Domain Parking App?

As someone managing multiple domain names, I often found myself searching for an easy, cost-effective way to display a for-sale page for each of my domains. I wanted visitors to land on a professional-looking page, learn about the domain, and reach out to express their interest in buying it. That’s why I built a template in **Next.js**, designed to be deployed on **Vercel** (a free and powerful hosting platform for Next.js apps).

## How It Works

Here’s a breakdown of how this domain parking system works:

- **Next.js for Frontend**: I created a simple, clean, and responsive template using [Next.js](https://nextjs.org/) and [TailwindCSS](https://tailwindcss.com/). It provides a landing page for each domain where visitors can learn more and inquire about purchasing the domain.

- **Resend Integration**: To facilitate communication, I integrated the form with [Resend](https://resend.com/). Visitors can leave their details via the contact form, which then triggers an email notification directly to me. This eliminates any complicated backend setup for handling inquiries.

![nextjs-vercel-parking]({{ url }}/assets/nextjs-vercel-parking.webp)

- **Vercel for Deployment**: [Vercel’s free tier](https://vercel.com) allows me to deploy the project quickly and configure it with multiple domains. Using **Vercel Projects**, I can park several domains under one project, each pointing to its own landing page. This means managing domains is centralized and hassle-free!

## The First Success

Shortly after setting up the app and parking a few domains, I received an inquiry through the contact form and sold one of my domains! This validated the approach I took and reinforced my decision to open-source this solution to help others with similar needs.

## Why Open Source?

By open-sourcing the project, I hope to help others who may be sitting on valuable domain names but haven’t yet found an efficient way to market them. Whether you have a single domain or a portfolio of them, this app can help you create beautiful landing pages and connect with potential buyers with minimal effort.

You can check out the code and contribute to the project on GitHub [here](https://github.com/bqst/nextjs-domain-parking-template). I’ve included instructions on how to set up your own domain parking app in minutes using **Next.js** and **Vercel**.

## Conclusion

Parking domain names doesn’t need to be complicated or expensive. With this open-source **Next.js** project, you can quickly set up a sleek landing page for any domain and start receiving offers. Plus, thanks to **Vercel’s free tier**, there’s no hosting cost involved!

Feel free to try it out, fork the project, and contribute! And who knows—maybe you’ll sell your next domain even faster than I did.

**Get started here**: [nextjs-domain-parking-template](https://github.com/bqst/nextjs-domain-parking-template).
