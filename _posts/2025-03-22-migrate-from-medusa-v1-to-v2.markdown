---
layout: post
title: "Migrate from Medusa v1 to v2"
date: 2025-03-22 12:00:00 +0000
categories: [medusa, ecommerce]
tags: [migration, ecommerce]
author: bqst
summary: "Guide on how to migrate from Medusa v1 to v2."
---

Learn how we migrated our e-commerce store from Medusa v1 to v2 with this step-by-step guide. We share the exact SQL scripts and techniques we used to transfer products, customers, inventory, and pricing data while avoiding common pitfalls.

---

WIP: This post is a work in progress.

## Introduction: Our Journey from Medusa v1 to v2

I'm the tech lead of [Beans](https://www.beansclub.fr), an e-commerce platform specializing in B2C products with short shelf lives. When launching Beans, I needed a robust, flexible e-commerce solution that could handle our unique requirements for managing products with limited shelf life while providing excellent customer experiences.

### Why We Chose Medusa Initially

After evaluating several e-commerce frameworks, [Medusa](https://www.medusajs.com/) stood out for its developer-friendly architecture, extensibility, and headless approach. As an open-source alternative to platforms like Shopify, Medusa offered the perfect balance of customization and structure. We built our entire platform on Medusa v1, which continues to power many e-commerce businesses today. It's also based on TypeScript, which is a great fit for our team and the project.

### Understanding Medusa

For those unfamiliar, Medusa is a composable commerce engine that enables developers to create custom e-commerce experiences with ease. It provides a robust backend with APIs that can connect to any frontend, allowing for complete flexibility in how you present your store to customers. See the [Medusa docs](https://docs.medusajs.com/) for more information.

### The Arrival of Medusa v2

When [Medusa announced version 2](https://medusajs.com/blog/v2-release/), it represented a significant evolution of the platform. The upgrade introduced a more modular architecture, improved database schemas, a completely revamped inventory management system, and enhanced authentication—all designed to offer better performance and scalability.

### Why We Decided to Migrate

Several factors influenced our decision to upgrade:

- **Future-proofing our platform**: Staying up-to-date with the latest features and security updates
- **Improved documentation and community support**: v2 has much better resources for developers
- **Enhanced extension capabilities**: The new modules system makes it easier to add custom functionality
- **Access to Medusa Cloud offerings**: Taking advantage of managed services that integrate seamlessly with v2
- **Performance improvements**: Benefiting from optimizations in the core architecture

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

- Core Platform: [Medusa v1](https://docs.medusajs.com/v1/)
- Payment Processing:
  - [Stripe](https://stripe.com/) via [medusa-payment-stripe](https://github.com/medusajs/medusa/tree/v1.20.11)
  - [PayPal](https://www.paypal.com/) via [medusa-payment-paypal](https://github.com/medusajs/medusa/tree/v1.20.11)
- File Storage: [Cloudflare R2](https://www.cloudflare.com/r2/) (S3-compatible) via [medusa-file-r2](https://github.com/yinkakun/medusa-file-r2)
- Search: [Algolia](https://www.algolia.com/) via [medusa-plugin-algolia](https://github.com/adrien2p/medusa/tree/863d80396a9ec1eb535ed1194a52b790bc573406/packages/medusa-plugin-algolia)
- Customer Communications: [Customer.io](https://customer.io/) (custom integration)
- Error Monitoring: [Sentry](https://sentry.io/) via [medusa-plugin-sentry](https://github.com/adrien2p/medusa-plugins)
- Frontend: [Next.js](https://nextjs.org/) storefront

It's worth noting that our setup uses a single region, single currency, single language, and serves only one country. This simplified some aspects of the migration, but the approach outlined in this guide can be adapted for multi-region setups with additional considerations.

### Migration Goals

Our primary objectives for this migration were:

- **Maintain the same technology stack** - We wanted to continue using all the same services and plugins, just upgraded for Medusa v2 compatibility
- **Seamless customer experience** - Customers should be able to log in post-migration and find their accounts and order history intact
- **Business continuity** - All products, inventory, and operational systems should remain consistent before and after migration
- **Minimal downtime** - Complete the transition quickly to avoid disrupting our business operations

We also wanted to benefit [Medusa Cloud offerings](https://medusajs.com/pricing/), but we didn't want to migrate to it right away, so we decided to migrate to v2 first and then migrate to Medusa Cloud later.

This article focuses specifically on the database migration aspects of moving from Medusa v1 to v2, which is the most complex part of the process. While plugin configuration and frontend adjustments were also required, the data migration was the critical path that required careful planning and execution.

In the following sections, I'll walk you through the exact steps we took to migrate our product catalog, customer data, and order information while ensuring everything continued to work seamlessly with our existing integrations.

## The Migration Process

### Step 1: Initial Setup

Before starting the migration, we created a new Medusa v2 project and set up the initial project structure, based on the [Medusa v2 docs](https://docs.medusajs.com/learn/quickstart/create-a-new-project). This included:

```bash
npx create-medusa-app@latest my-medusa-store
```

This command created a new Medusa v2 project with the default configuration. We then added the necessary plugins and integrations to match our existing setup.

### Step 2 : Create a default admin user

```bash
npx medusa user -e admin@medusajs.com -p supersecret
```

### Step 3: Setup the default settings via the admin dashboard

We need to setup the default settings via the admin dashboard.

- Create a default region
- Create a default tax region
- Create a default store
- Create a default sales channel
- Create a default warehouse

### Step 4 : Setup `postgres_fdw` extension

For data migration, several solutions exist. It's possible to migrate data via Medusa's APIs, but we chose to perform a database-level migration instead.

We decided to use the `postgres_fdw` extension to access the existing database.

`postgres_fdw` is a PostgreSQL extension that allows you to access remote databases from a PostgreSQL database.

For more information, you can refer to the [official documentation](https://www.postgresql.org/docs/current/postgres-fdw.html).

From the new Medusa v2 database, run the following command to install the `postgres_fdw` extension:

```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;
```

### Step 5 : Create a foreign server & user mapping

We need to create a foreign server to access the existing database, this will be used to create the foreign tables.

```sql
-- Create a foreign server
CREATE SERVER medusa_v1_server FOREIGN DATA WRAPPER postgres_fdw OPTIONS (host 'localhost', port '5432', dbname 'medusa_v1');
```

```sql
-- Create a user mapping for the current user to the source server
CREATE USER MAPPING FOR current_user SERVER source_server 
OPTIONS (user 'postgres', password 'postgres');
```

### Step 6 : Migrate products data

With the help of IA (ChatGPT or Claude will help you), we created the following SQL scripts to migrate the data from the existing database to the new Medusa v2 database based on the schema of the new Medusa v2 database.

We also tried to preserve the same identifiers as much as possible to facilitate data relationships. (e.g., `product_category.id = product_category_foreign.id`)

#### Categories, collections and tags

##### Product category

```sql
CREATE FOREIGN TABLE product_category_foreign (
    id character varying NOT NULL,
    name text NOT NULL,
    handle text NOT NULL,
    parent_category_id character varying,
    mpath text,
    is_active boolean DEFAULT false,
    is_internal boolean DEFAULT false,
    created_at timestamp with time zone NOT NULL DEFAULT now(),
    updated_at timestamp with time zone NOT NULL DEFAULT now(),
    rank integer NOT NULL DEFAULT 0,
    description text NOT NULL DEFAULT ''::text,
    metadata jsonb
)
SERVER source_server OPTIONS (table_name 'product_category');

INSERT INTO product_category (
    id, name, handle, parent_category_id, mpath, 
    is_active, is_internal, created_at, updated_at,
    rank, description, metadata
)
SELECT 
    id, name, handle, parent_category_id, mpath,
    is_active, is_internal, created_at, updated_at,
    rank, description, metadata 
FROM product_category_foreign;
```

##### Product collection

```sql
CREATE FOREIGN TABLE product_collection_foreign (
    id character varying NOT NULL,
    title character varying NOT NULL,
    handle character varying,
    created_at timestamp with time zone NOT NULL DEFAULT now(),
    updated_at timestamp with time zone NOT NULL DEFAULT now(),
    deleted_at timestamp with time zone,
    metadata jsonb
)
SERVER source_server OPTIONS (table_name 'product_collection');

INSERT INTO product_collection (
    id, title, handle, created_at, updated_at, deleted_at, metadata
)
SELECT 
    id, title, handle, created_at, updated_at, deleted_at, metadata
FROM product_collection_foreign;
```

##### Product tag

```sql
CREATE FOREIGN TABLE product_tag_foreign (
    id character varying NOT NULL,
    value character varying NOT NULL,
    created_at timestamp with time zone NOT NULL DEFAULT now(),
    updated_at timestamp with time zone NOT NULL DEFAULT now(),
    deleted_at timestamp with time zone,
    metadata jsonb
)
SERVER source_server OPTIONS (table_name 'product_tag');

INSERT INTO product_tag (
    id, value, created_at, updated_at, deleted_at, metadata
)
SELECT 
    id, value, created_at, updated_at, deleted_at, metadata
FROM product_tag_foreign;
```

#### Products & variants

##### Product

```sql
CREATE FOREIGN TABLE product_foreign (
    id character varying NOT NULL,
    title character varying NOT NULL,
    subtitle character varying,
    description character varying,
    handle character varying,
    is_giftcard boolean NOT NULL DEFAULT false,
    thumbnail character varying,
    weight integer,
    length integer,
    height integer,
    width integer,
    hs_code character varying,
    origin_country character varying,
    mid_code character varying,
    material character varying,
    created_at timestamp with time zone NOT NULL DEFAULT now(),
    updated_at timestamp with time zone NOT NULL DEFAULT now(),
    deleted_at timestamp with time zone,
    metadata jsonb,
    collection_id character varying,
    type_id character varying,
    discountable boolean NOT NULL DEFAULT true,
    status text NOT NULL DEFAULT 'draft',
    external_id character varying
)
SERVER source_server OPTIONS (table_name 'product');

INSERT INTO product (
    id, title, subtitle, description, handle, is_giftcard, thumbnail, weight, length, height, width, hs_code, origin_country, mid_code, material, created_at, updated_at, deleted_at, metadata, collection_id, type_id, discountable, status, external_id
)
SELECT 
    id, title, subtitle, description, handle, is_giftcard, thumbnail, weight, length, height, width, hs_code, origin_country, mid_code, material, created_at, updated_at, deleted_at, metadata, collection_id, type_id, discountable, status, external_id
FROM product_foreign;
```

#### Product sales channel

Currently, we have only one sales channel.

```sql
INSERT INTO product_sales_channel (id, product_id, sales_channel_id)
SELECT 
    'prodsc_' || substr(md5(random()::text), 1, 26), -- Generate a unique ID with 'psc_' prefix
    id as product_id,
    'sc_XXX' as sales_channel_id -- TODO: add the correct sales channel id previously created from the admin dashboard
FROM product;
```

#### Product images

Currently, products in our system are limited to a single thumbnail image stored in the product table, so we don't need to populate the product_images table separately. This simplifies our migration process for the image data.


#### Product option

In our implementation, each product is configured with a single variant, utilizing only one product option labeled "default". This simplified approach streamlines our product structure while still maintaining compatibility with Medusa's variant system.

```sql
INSERT INTO product_option (id, title, product_id)
SELECT 
    'opt_' || substr(md5(random()::text), 1, 26), -- Generate unique ID with 'opt_' prefix
    'default' as title,
    id as product_id
FROM product;
```

#### Link product to category

```sql
-- Create foreign table for old product_category_product relationships
CREATE FOREIGN TABLE product_category_product_foreign (
    product_category_id text NOT NULL,
    product_id text NOT NULL
)
SERVER source_server 
OPTIONS (table_name 'product_category_product');

-- Migrate the relationships
INSERT INTO product_category_product (
    product_category_id,
    product_id
)
SELECT 
    product_category_id,
    product_id
FROM product_category_product_foreign;
```

#### Variant

**Important Note:** The `inventory_quantity` field has been removed from the product variant schema in Medusa v2. Inventory is now managed through the new inventory module, which provides more robust stock tracking capabilities.

```sql
CREATE FOREIGN TABLE product_variant_foreign (
   id character varying NOT NULL,
   title character varying NOT NULL,
   product_id character varying NOT NULL,
   sku character varying,
   barcode character varying,
   ean character varying,
   upc character varying,
   inventory_quantity integer NOT NULL,
   allow_backorder boolean NOT NULL DEFAULT false,
   manage_inventory boolean NOT NULL DEFAULT true,
   hs_code character varying,
   origin_country character varying,
   mid_code character varying,
   material character varying,
   weight integer,
   length integer,
   height integer,
   width integer,
   created_at timestamp with time zone NOT NULL DEFAULT now(),
   updated_at timestamp with time zone NOT NULL DEFAULT now(),
   deleted_at timestamp with time zone,
   metadata jsonb,
   variant_rank integer DEFAULT 0
)
SERVER source_server OPTIONS (table_name 'product_variant');

INSERT INTO product_variant (
   id,
   title,
   sku,
   barcode,
   ean,
   upc,
   allow_backorder,
   manage_inventory,
   hs_code,
   origin_country,
   mid_code,
   material,
   weight,
   length,
   height,
   width,
   metadata,
   variant_rank,
   product_id,
   created_at,
   updated_at,
   deleted_at
)
SELECT 
   id,
   title,
   sku,
   barcode,
   ean,
   upc,
   allow_backorder,
   manage_inventory,
   hs_code,
   origin_country,
   mid_code,
   material,
   weight,
   length,
   height,
   width,
   metadata,
   variant_rank,
   product_id,
   created_at,
   updated_at,
   deleted_at
FROM product_variant_foreign;
```

```sql
INSERT INTO product_option_value (id, value, option_id)
SELECT 
    'optval_' || substr(md5(random()::text), 1, 24), -- Generate unique ID with 'optval_' prefix
    'default' as value,
    id as option_id
FROM product_option;
```

```sql
INSERT INTO product_variant_option (variant_id, option_value_id)
SELECT 
    pv.id as variant_id,
    pov.id as option_value_id
FROM product_variant pv
JOIN product_option po ON po.product_id = pv.product_id
JOIN product_option_value pov ON pov.option_id = po.id;
```

#### Inventory

Create a default stock_location first → [http://localhost:9000/app/settings/locations/](http://localhost:9000/app/settings/locations/)

```sql
INSERT INTO inventory_item (
    id,
    sku,
    origin_country,
    hs_code,
    mid_code,
    material,
    weight,
    length,
    height,
    width,
    title,
    metadata
)
SELECT 
    'iitem_' || substr(md5(random()::text), 1, 24) as id,
    sku,
    origin_country,
    hs_code,
    mid_code,
    material,
    weight,
    length,
    height,
    width,
    title,
    metadata
FROM product_variant_foreign;
```

```sql
WITH inserted_items AS (
    SELECT pv.id as variant_id, ii.id as inventory_item_id
    FROM product_variant_foreign pv
    JOIN inventory_item ii ON ii.sku = pv.sku
)
INSERT INTO product_variant_inventory_item (
    id,
    variant_id,
    inventory_item_id,
    required_quantity
)
SELECT 
    'pvitem_' || substr(md5(random()::text), 1, 23),
    variant_id,
    inventory_item_id,
    1
FROM inserted_items;
```

```sql
INSERT INTO inventory_level (
    id,
    inventory_item_id,
    location_id,
    stocked_quantity,
    reserved_quantity,
    incoming_quantity
)
SELECT 
    'ilev_' || substr(md5(random()::text), 1, 25) as id,
    ii.id as inventory_item_id,
    'sloc_XXX' as location_id, -- Replace with your actual default location_id
    pv.inventory_quantity::numeric as stocked_quantity,
    0 as reserved_quantity,
    0 as incoming_quantity
FROM product_variant_foreign pv
JOIN product_variant_inventory_item pvii ON pvii.variant_id = pv.id
JOIN inventory_item ii ON ii.id = pvii.inventory_item_id;
```

#### Prices

```sql
-- First, create price sets for all variants
WITH variant_list AS (
    SELECT 
        id as variant_id,
        ROW_NUMBER() OVER () as rn
    FROM product_variant
    WHERE deleted_at IS NULL
),
inserted_price_sets AS (
    INSERT INTO price_set (id)
    SELECT 
        'pset_' || substr(md5(random()::text), 1, 24)
    FROM variant_list
    RETURNING id
),
numbered_price_sets AS (
    SELECT 
        id as price_set_id,
        ROW_NUMBER() OVER () as rn
    FROM inserted_price_sets
),
variant_price_set_mapping AS (
    SELECT 
        vl.variant_id,
        ps.price_set_id
    FROM variant_list vl
    JOIN numbered_price_sets ps ON ps.rn = vl.rn
)
-- Then link price sets to variants
INSERT INTO product_variant_price_set (
    id,
    variant_id,
    price_set_id
)
SELECT 
    'pvps_' || substr(md5(random()::text), 1, 24),
    variant_id,
    price_set_id
FROM variant_price_set_mapping;
```

```sql
-- Create foreign tables for money_amount data
CREATE FOREIGN TABLE money_amount_foreign (
    id text NOT NULL,
    currency_code text NOT NULL,
    amount integer NOT NULL,
    region_id text,
    created_at timestamp with time zone NOT NULL DEFAULT now(),
    updated_at timestamp with time zone NOT NULL DEFAULT now(),
    deleted_at timestamp with time zone,
    min_quantity integer,
    max_quantity integer,
    price_list_id text
)
SERVER source_server 
OPTIONS (table_name 'money_amount');

CREATE FOREIGN TABLE product_variant_money_amount_foreign (
    id text NOT NULL,
    money_amount_id text NOT NULL,
    variant_id text NOT NULL,
    created_at timestamp with time zone DEFAULT CURRENT_TIMESTAMP,
    updated_at timestamp with time zone DEFAULT CURRENT_TIMESTAMP,
    deleted_at timestamp with time zone
)
SERVER source_server 
OPTIONS (table_name 'product_variant_money_amount');

-- Insert into price table
INSERT INTO price (
    id,
    price_set_id,
    currency_code,
    amount,
    raw_amount,
    created_at,
    updated_at,
    deleted_at,
    min_quantity,
    max_quantity
)
SELECT 
    'price_' || substr(md5(random()::text), 1, 24),
    pvps.price_set_id,
    ma.currency_code,
    (ma.amount::numeric / 100),  -- Convert from cents to decimal
    jsonb_build_object(
        'value', (ma.amount::numeric / 100)::text,
        'precision', '20'
    ),
    ma.created_at,
    ma.updated_at,
    ma.deleted_at,
    ma.min_quantity,
    ma.max_quantity
FROM money_amount_foreign ma
JOIN product_variant_money_amount_foreign pvma ON ma.id = pvma.money_amount_id
JOIN product_variant_price_set pvps ON pvma.variant_id = pvps.variant_id
WHERE ma.deleted_at IS NULL
AND pvma.deleted_at IS NULL;

-- Clean up
DROP FOREIGN TABLE money_amount_foreign;
DROP FOREIGN TABLE product_variant_money_amount_foreign;
```

```sql
-- First, create a temporary table with the prices we want to keep
WITH prices_to_keep AS (
    SELECT DISTINCT ON (price_set_id) 
        id,
        price_set_id,
        amount
    FROM price
    ORDER BY price_set_id, amount ASC
)
-- Delete all prices except the ones we want to keep
DELETE FROM price 
WHERE id NOT IN (
    SELECT id FROM prices_to_keep
);

-- Verify the results (optional)
SELECT price_set_id, COUNT(*) as count
FROM price
GROUP BY price_set_id
HAVING COUNT(*) > 1;
```

### Step 6 : Migrate customers data

We should keep the same COOKIE_SECRET and JWT_SECRET as the ones in the old database if possible if you want to keep the same session.

#### Customer

```sql
-- Create foreign table for old customer data
CREATE FOREIGN TABLE customer_foreign (
    id text NOT NULL,
    email text NOT NULL,
    first_name text,
    last_name text,
    billing_address_id text,
    password_hash text,
    phone text,
    has_account boolean NOT NULL DEFAULT false,
    created_at timestamp with time zone NOT NULL DEFAULT now(),
    updated_at timestamp with time zone NOT NULL DEFAULT now(),
    deleted_at timestamp with time zone,
    metadata jsonb,
    confirmed_at timestamp with time zone,
    confirmation_token text
)
SERVER source_server 
OPTIONS (table_name 'customer');

-- Migrate customers
INSERT INTO customer (
    id,
    email,
    first_name,
    last_name,
    phone,
    has_account,
    metadata,
    created_at,
    updated_at,
    deleted_at
)
SELECT 
    id,
    email,
    first_name,
    last_name,
    phone,
    has_account,
    COALESCE(metadata, '{}'::jsonb),
    created_at,
    updated_at,
    deleted_at
FROM customer_foreign
WHERE deleted_at IS NULL;

-- Clean up
DROP FOREIGN TABLE customer_foreign;
```

#### Auth & provider identity

```sql
-- Insert auth identities for customers with accounts
INSERT INTO auth_identity (
    id,
    app_metadata
)
SELECT 
    'authid_' || substr(md5(random()::text), 1, 24),
    jsonb_build_object('customer_id', id)
FROM customer
WHERE has_account = true
AND deleted_at IS NULL;

-- Verify the results (optional)
SELECT 
    c.id as customer_id,
    ai.id as auth_id,
    ai.app_metadata->>'customer_id' as linked_customer_id
FROM customer c
JOIN auth_identity ai ON ai.app_metadata->>'customer_id' = c.id
WHERE c.has_account = true;
```

```sql
-- Create foreign table to access old customer data with passwords
CREATE FOREIGN TABLE customer_foreign (
    id text NOT NULL,
    email text NOT NULL,
    password_hash text,
    has_account boolean NOT NULL DEFAULT false,
    deleted_at timestamp with time zone
)
SERVER source_server 
OPTIONS (table_name 'customer');

-- Insert provider identities
INSERT INTO provider_identity (
    id,
    entity_id,
    provider,
    auth_identity_id,
    provider_metadata
)
SELECT 
    'pid_' || substr(md5(random()::text), 1, 24) as id,
    c.email as entity_id,
    'emailpass' as provider,
    ai.id as auth_identity_id,
    jsonb_build_object('password', cf.password_hash) as provider_metadata
FROM customer c
JOIN auth_identity ai ON ai.app_metadata->>'customer_id' = c.id
JOIN customer_foreign cf ON cf.email = c.email
WHERE c.has_account = true 
AND c.deleted_at IS NULL
AND cf.password_hash IS NOT NULL;

-- Clean up
DROP FOREIGN TABLE customer_foreign;

-- Verify the results (optional)
SELECT 
    pi.id,
    c.email,
    pi.provider,
    pi.auth_identity_id,
    ai.app_metadata->>'customer_id' as customer_id
FROM customer c
JOIN auth_identity ai ON ai.app_metadata->>'customer_id' = c.id
JOIN provider_identity pi ON pi.auth_identity_id = ai.id
WHERE c.has_account = true;
```

#### Address

WIP

### Step 7 : Migrate orders data

WIP

### Step 8 : Setup plugins

WIP
