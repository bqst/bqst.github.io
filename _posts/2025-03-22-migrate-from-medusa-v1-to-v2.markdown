---
layout: post
title: "Migrate from Medusa v1 to v2"
date: 2025-03-22 12:00:00 +0000
categories: [medusa, ecommerce]
tags: [migration, ecommerce]
author: bqst
summary: "Guide on how to migrate from Medusa v1 to v2."
---

Learn how we successfully migrated our e-commerce store from Medusa v1 to v2 with this step-by-step guide. We share the exact SQL scripts and techniques we used to transfer products, customers, inventory, and pricing data while avoiding common pitfalls. Whether you're planning your migration or troubleshooting an existing one, this practical walkthrough will help you navigate the process with confidence.

---

## Introduction: Our Journey from Medusa v1 to v2

I'm the tech lead of Beans, an e-commerce platform specializing in B2C products with short shelf lives. When launching Beans, I needed a robust, flexible e-commerce solution that could handle our unique requirements for managing products with limited shelf life while providing excellent customer experiences.

### Why We Chose Medusa Initially

After evaluating several e-commerce frameworks, Medusa stood out for its developer-friendly architecture, extensibility, and headless approach. As an open-source alternative to platforms like Shopify, Medusa offered the perfect balance of customization and structure. We built our entire platform on Medusa v1, which continues to power many e-commerce businesses today. It's also based on TypeScript, which is a great fit for our team and the project.

### Understanding Medusa

For those unfamiliar, Medusa is a composable commerce engine that enables developers to create custom e-commerce experiences with ease. It provides a robust backend with APIs that can connect to any frontend, allowing for complete flexibility in how you present your store to customers.

### The Arrival of Medusa v2

When Medusa announced version 2, it represented a significant evolution of the platform. The upgrade introduced a more modular architecture, improved database schemas, a completely revamped inventory management system, and enhanced authentication—all designed to offer better performance and scalability.

### Why We Decided to Migrate

Several factors influenced our decision to upgrade:

- Future-proofing our platform: Staying up-to-date with the latest features and security updates
- Improved documentation and community support: v2 has much better resources for developers
- Enhanced extension capabilities: The new modules system makes it easier to add custom functionality
- Access to Medusa Cloud offerings: Taking advantage of managed services that integrate seamlessly with v2
- Performance improvements: Benefiting from optimizations in the core architecture

### A Fresh Start with Best Practices

Rather than attempting to upgrade our existing codebase in-place, we decided to start with a fresh Medusa v2 installation and migrate our data. This approach allowed us to:

- Implement best practices from the beginning
- Clean up technical debt that had accumulated
- Rethink certain architectural decisions
- Set up improved development workflows
- Establish better coding standards and patterns

This decision to start fresh turned what could have been a complex upgrade into an opportunity for substantial improvement—not just in the platform itself but in our development practices.

In the following sections, I'll walk you through our exact migration process, sharing the SQL scripts and techniques we used to transfer everything from our product catalog to customer data, all while maintaining business continuity.

## Our Medusa v1 Stack

Before diving into the migration process, it's important to understand the technology stack we were working with. This context is crucial as it shapes the migration approach and highlights specific considerations for each component.

### Current Stack (Hosted on Render.com)

Our Medusa v1 implementation included several integrations to provide a complete e-commerce experience:

- Core Platform: Medusa v1
- Error Monitoring: Sentry via medusa-plugin-sentry
- Payment Processing:
  - Stripe via medusa-payment-stripe
  - PayPal via medusa-payment-paypal
- File Storage: Cloudflare R2 (S3-compatible) via medusa-file-r2
- Search: Algolia via medusa-plugin-algolia
- Customer Communications: Customer.io (custom integration)
- Frontend: Next.js storefront

It's worth noting that our setup uses a single region, single currency, single language, and serves only one country. This simplified some aspects of the migration, but the approach outlined in this guide can be adapted for multi-region setups with additional considerations.

### Migration Goals

Our primary objectives for this migration were:

- Maintain the same technology stack - We wanted to continue using all the same services and plugins, just upgraded for Medusa v2 compatibility
- Seamless customer experience - Customers should be able to log in post-migration and find their accounts and order history intact
- Business continuity - All products, inventory, and operational systems should remain consistent before and after migration
- Minimal downtime - Complete the transition quickly to avoid disrupting our business operations

This article focuses specifically on the database migration aspects of moving from Medusa v1 to v2, which is the most complex part of the process. While plugin configuration and frontend adjustments were also required, the data migration was the critical path that required careful planning and execution.

In the following sections, I'll walk you through the exact steps we took to migrate our product catalog, customer data, and order information while ensuring everything continued to work seamlessly with our existing integrations.
