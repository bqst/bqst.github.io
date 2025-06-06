---
layout: post
title: "Medusa v2 Product Availability Module"
date: 2025-06-06 12:00:00 +0000
categories: [medusa, ecommerce]
tags: [medusa, ecommerce]
author: bqst
summary: "Guide on how to create a product availability module for Medusa v2."
---

## 🎯 Overview

This guide demonstrates how to create a custom **Availability Module** for Medusa v2 to filter products by stock status on listing pages (categories, brands, collections) while maintaining SEO-friendly product pages and order history access.

### ❌ The Problem

Medusa v2's Store API doesn't support filtering products by availability status or variant inventory quantities. Common workarounds have significant drawbacks:

- **Archiving out-of-stock products**: Breaks SEO, wishlist functionality, and order history
- **Client-side filtering**: Creates inconsistent pagination, incorrect counts, and performance issues
- **Over-fetching**: Poor performance and user experience
- **Database-level filtering**: Not supported by current API structure

### ✅ The Solution

A dedicated **Availability Module** that:

- Extends the Product module following Medusa v2 architecture
- Provides `/store/products/available` endpoint with proper filtering, sorting, and pagination
- Manages availability states: `incoming`, `in_stock`, `out_of_stock`
- Automatically updates based on inventory changes
- Includes admin dashboard integration

---

## 🏗️ Architecture Overview

```mermaid
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Product       │────│   Availability   │────│   Inventory     │
│   Module        │    │   Module         │    │   Module        │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                              │
                    ┌─────────┼─────────┐
                    │         │         │
            ┌───────▼───┐ ┌───▼────┐ ┌──▼────────┐
            │Subscribers│ │Workflow│ │API Routes │
            └───────────┘ └────────┘ └───────────┘
```

**Key Design Decisions:**

- **Product-level availability** (not variant-level) for simplified management
- **Single location/sales channel** optimization
- **One variant per product** assumption
- **Automatic status calculation** based on inventory levels

---

## 📋 Implementation Steps

### Step 1: Create Module Structure

```bash
src/modules/availability/
├── models/
│   └── availability.ts
├── service.ts
└── index.ts
```

### Step 2: Define the Availability Data Model

**File:** `src/modules/availability/models/availability.ts`

```typescript
import { model } from "@medusajs/framework/utils"

export enum AvailabilityStatus {
  INCOMING = "incoming",
  IN_STOCK = "in_stock",
  OUT_OF_STOCK = "out_of_stock",
}

const Availability = model.define("availability", {
  id: model.id().primaryKey(),
  product_id: model.text(), // Link to product
  status: model.enum(AvailabilityStatus).default(AvailabilityStatus.INCOMING),
  notes: model.text().nullable(),
})

export default Availability
```

### Step 3: Create Module Service

**File:** `src/modules/availability/service.ts`

```typescript
import { MedusaService } from "@medusajs/framework/utils"
import Availability, { AvailabilityStatus } from "./models/availability"

class AvailabilityModuleService extends MedusaService({
  Availability,
}) {
  /**
   * Update or create availability record for a product
   */
  async updateProductAvailability(
    product_id: string,
    status: AvailabilityStatus,
    notes?: string
  ) {
    const [existingAvailability] = await this.listAvailabilities({
      product_id,
    })

    if (existingAvailability) {
      return await this.updateAvailabilities({
        id: existingAvailability.id,
        status,
        notes,
      })
    } else {
      return await this.createAvailabilities({
        product_id,
        status,
        notes,
      })
    }
  }

  /**
   * Get availability for multiple products
   */
  async getProductsAvailability(product_ids: string[]) {
    return await this.listAvailabilities({
      product_id: product_ids,
    })
  }

  /**
   * Get products by availability status
   */
  async getProductsByStatus(status: AvailabilityStatus) {
    return await this.listAvailabilities({
      status,
    })
  }
}

export default AvailabilityModuleService
```

### Step 4: Register the Module

**File:** `src/modules/availability/index.ts`

```typescript
import { Module } from "@medusajs/framework/utils"
import AvailabilityModuleService from "./service"

export const AVAILABILITY_MODULE = "availability"

export default Module(AVAILABILITY_MODULE, {
  service: AvailabilityModuleService,
})
```

**File:** `medusa-config.ts` (add to modules array)

```typescript
{
  resolve: "./src/modules/availability",
},
```

### Step 5: Database Migration

```bash
# Generate migration
npx medusa db:generate availability

# Run migration
npx medusa db:migrate
```

---

## 🔄 Automated Availability Management

### Product Creation Subscriber

**File:** `src/subscribers/availability-product-created-handler.ts`

```typescript
import { SubscriberArgs, SubscriberConfig } from "@medusajs/framework"
import { AvailabilityStatus } from "../modules/availability/models/availability"
import { updateProductAvailabilityWorkflow } from "../workflows/update-product-availability"

export default async function availabilityProductCreatedHandler({
  event: { data },
  container,
}: SubscriberArgs<{ id: string }>) {
  // Initialize new products as INCOMING
  await updateProductAvailabilityWorkflow(container).run({
    input: {
      product_id: data.id,
      status: AvailabilityStatus.INCOMING,
    },
  })
}

export const config: SubscriberConfig = {
  event: "product.created",
}
```

### Inventory Level Subscriber

**File:** `src/subscribers/availability-inventory-updated-handler.ts`

```typescript
import { SubscriberArgs, SubscriberConfig } from "@medusajs/framework"
import { ContainerRegistrationKeys, InventoryEvents } from "@medusajs/framework/utils"
import { updateProductAvailabilityWorkflow } from "../workflows/update-product-availability"

export default async function availabilityInventoryUpdatedHandler({
  event: { data },
  container,
}: SubscriberArgs<{ id: string }>) {
  console.log("Inventory updated, recalculating availability:", data)

  const query = container.resolve(ContainerRegistrationKeys.QUERY)

  // Get affected product IDs from inventory level changes
  const { data: inventoryLevels } = await query.graph({
    entity: "inventory_level",
    fields: ["id", "inventory_item.variants.*"],
    filters: { id: { $eq: data.id } },
  })

  const productIds = inventoryLevels
    .map((level) => level?.inventory_item?.variants)
    .filter(Boolean)
    .flat()
    .map((variant) => variant?.product_id)
    .filter(Boolean) as string[]

  // Update availability for each affected product
  for (const productId of [...new Set(productIds)]) {
    await updateProductAvailabilityWorkflow(container).run({
      input: { product_id: productId },
    })
  }
}

export const config: SubscriberConfig = {
  event: [
    InventoryEvents.INVENTORY_LEVEL_CREATED,
    InventoryEvents.INVENTORY_LEVEL_UPDATED,
    InventoryEvents.INVENTORY_LEVEL_DELETED,
  ],
}
```

---

## ⚙️ Availability Calculation Workflow

**File:** `src/workflows/update-product-availability.ts`

```typescript
import {
  ContainerRegistrationKeys,
  getVariantAvailability,
  VariantAvailabilityResult,
} from "@medusajs/framework/utils"
import {
  createStep,
  createWorkflow,
  StepResponse,
  WorkflowResponse,
} from "@medusajs/framework/workflows-sdk"
import { AVAILABILITY_MODULE } from "../modules/availability"
import { AvailabilityStatus } from "../modules/availability/models/availability"
import AvailabilityModuleService from "../modules/availability/service"

type UpdateAvailabilityInput = {
  product_id: string
  status?: AvailabilityStatus // Optional: force specific status
  notes?: string
}

const calculateAvailabilityStep = createStep(
  "calculate-product-availability-step",
  async ({ product_id, status: forcedStatus }: UpdateAvailabilityInput, { container }) => {
    // Use forced status if provided
    if (forcedStatus) {
      return new StepResponse({
        product_id,
        status: forcedStatus,
        calculated: false,
      })
    }

    const query = container.resolve(ContainerRegistrationKeys.QUERY)

    // Get product with variants and inventory data
    const { data: products } = await query.graph({
      entity: "product",
      fields: ["id", "status", "variants.id", "variants.manage_inventory"],
      filters: { id: product_id },
    })

    if (!products[0]) {
      throw new Error(`Product ${product_id} not found`)
    }

    const product = products[0]
    let status = AvailabilityStatus.OUT_OF_STOCK

    if (!product.variants || product.variants.length === 0) {
      // No variants = incoming product
      status = AvailabilityStatus.INCOMING
    } else {
      // Check variant availability across all sales channels
      const salesChannels = await query.graph({
        entity: "sales_channel",
        fields: ["id"],
      })

      let hasAvailableVariant = false

      for (const salesChannel of salesChannels.data) {
        const variantAvailability = await getVariantAvailability(query, {
          variant_ids: product.variants.map((v) => v.id),
          sales_channel_id: salesChannel.id,
        })

        // Check if any variant has stock in this channel
        const channelHasStock = Object.values(variantAvailability).some(
          (availability) => availability.availability > 0
        )

        if (channelHasStock) {
          hasAvailableVariant = true
          break
        }
      }

      status = hasAvailableVariant
        ? AvailabilityStatus.IN_STOCK
        : AvailabilityStatus.OUT_OF_STOCK
    }

    return new StepResponse({
      product_id,
      status,
      calculated: true,
    })
  }
)

const updateAvailabilityStep = createStep(
  "update-product-availability-step",
  async (
    {
      product_id,
      status,
      notes,
    }: {
      product_id: string
      status: AvailabilityStatus
      notes?: string
    },
    { container }
  ) => {
    const availabilityService: AvailabilityModuleService =
      container.resolve(AVAILABILITY_MODULE)

    await availabilityService.updateProductAvailability(product_id, status, notes)

    return new StepResponse({ product_id, status, updated: true })
  }
)

export const updateProductAvailabilityWorkflow = createWorkflow(
  "update-product-availability-workflow",
  (input: UpdateAvailabilityInput) => {
    const availability = calculateAvailabilityStep(input)
    const result = updateAvailabilityStep({
      ...availability,
      notes: input.notes,
    })

    return new WorkflowResponse(result)
  }
)
```

---

## 🌐 Store API Endpoint

**File:** `src/api/store/products/available/route.ts`

```typescript
import {
  ContainerRegistrationKeys,
  ProductStatus,
} from "@medusajs/framework/utils"
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"
import { AVAILABILITY_MODULE } from "../../../../modules/availability"
import AvailabilityModuleService from "../../../../modules/availability/service"
import { AvailabilityStatus } from "../../../../modules/availability/models/availability"

export async function GET(req: MedusaRequest, res: MedusaResponse) {
  const query = req.scope.resolve(ContainerRegistrationKeys.QUERY)
  const availabilityService: AvailabilityModuleService =
    req.scope.resolve(AVAILABILITY_MODULE)

  const {
    limit = 20,
    offset = 0,
    status = AvailabilityStatus.IN_STOCK,
    category_id,
    collection_id,
    brand_id,
  } = req.query

  // Get available product IDs
  const availabilityRecords = await availabilityService.listAvailabilities(
    {
      status: status as AvailabilityStatus,
    },
    {
      take: parseInt(limit as string),
      skip: parseInt(offset as string),
      select: ["product_id"],
    }
  )

  const productIds = availabilityRecords.map((record) => record.product_id)

  if (productIds.length === 0) {
    return res.json({
      products: [],
      count: 0,
      offset: parseInt(offset as string),
      limit: parseInt(limit as string),
    })
  }

  // Build product filters
  const filters: any = {
    id: productIds,
    status: { $eq: ProductStatus.PUBLISHED },
  }

  // Add optional filters
  if (category_id) {
    filters.categories = { id: category_id }
  }
  if (collection_id) {
    filters.collections = { id: collection_id }
  }
  // Add brand filter if using brand module

  // Get products with full data
  const { data: products } = await query.graph({
    entity: "product",
    fields: [
      "id",
      "title",
      "handle",
      "thumbnail",
      "description",
      "variants.id",
      "variants.title",
      "variants.calculated_price.*",
      "variants.inventory_quantity",
      "categories.id",
      "categories.name",
      "collections.id",
      "collections.title",
    ],
    filters,
  })

  res.json({
    products: products || [],
    count: availabilityRecords.length,
    offset: parseInt(offset as string),
    limit: parseInt(limit as string),
  })
}
```

**Usage Examples:**

```bash
# Get available products (in stock)
GET /store/products/available

# Get incoming products
GET /store/products/available?status=incoming

# Get available products in specific category
GET /store/products/available?category_id=cat_123

# Pagination
GET /store/products/available?limit=10&offset=20
```

---

## 🔧 Admin Dashboard Integration

### Admin API Route

**File:** `src/api/admin/products/[id]/availability/route.ts`

```typescript
import { MedusaRequest, MedusaResponse } from "@medusajs/framework/http"
import AvailabilityModuleService from "../../../../../modules/availability/service"
import { AVAILABILITY_MODULE } from "../../../../../modules/availability"
import { AvailabilityStatus } from "../../../../../modules/availability/models/availability"

export async function GET(req: MedusaRequest, res: MedusaResponse) {
  const { id } = req.params
  const query = req.scope.resolve("query")

  const {
    data: [availability],
  } = await query.graph({
    entity: "availability",
    fields: ["*"],
    filters: {
      product_id: id,
    },
  })

  res.json({
    availability,
  })
}

export async function POST(
  req: MedusaRequest<{
    status: AvailabilityStatus
    notes?: string
  }>,
  res: MedusaResponse
) {
  const { id: product_id } = req.params
  const { status, notes } = req.body

  const availabilityService: AvailabilityModuleService =
    req.scope.resolve(AVAILABILITY_MODULE)

  const availability = await availabilityService.updateProductAvailability(
    product_id,
    status,
    notes
  )

  res.json({
    availability,
  })
}
```

### Admin Widget Component

**File:** `src/admin/widgets/product-availability.tsx`

```typescript
import { AdminProduct, DetailWidgetProps } from "@medusajs/framework/types"
import { defineWidgetConfig } from "@medusajs/admin-sdk"
import { Container, Heading, Skeleton, StatusBadge, toast } from "@medusajs/ui"
import { ActionMenu } from "../components/common/action-menu"
import { Check, Loader, XMark } from "@medusajs/icons"
import {
  useProductAvailability,
  useUpdateProductAvailability,
} from "../hooks/api/availabilities"
import { useNavigate } from "react-router-dom"
import { useEffect, useState } from "react"

const getStatusColor = (status: string) => {
  switch (status) {
    case "incoming": return "orange"
    case "in_stock": return "green"
    case "out_of_stock": return "red"
    default: return "grey"
  }
}

const getStatusLabel = (status: string) => {
  switch (status) {
    case "incoming": return "Incoming"
    case "in_stock": return "In Stock"
    case "out_of_stock": return "Out of Stock"
    default: return "Unknown"
  }
}

const ProductAvailability = ({ data: product }: DetailWidgetProps<AdminProduct>) => {
  const navigate = useNavigate()
  const { availability, isLoading } = useProductAvailability(product.id)
  const { mutateAsync } = useUpdateProductAvailability(product.id)

  const [status, setStatus] = useState<string>(availability?.status || "")

  useEffect(() => {
    setStatus(availability?.status || "")
  }, [availability])

  const handleSetAvailability = async (newStatus: "incoming" | "in_stock" | "out_of_stock") => {
    try {
      await mutateAsync({ status: newStatus })
      setStatus(newStatus)
      toast.success("Availability updated successfully")
      navigate(`/products/${product.id}`, { replace: true })
    } catch (error) {
      toast.error("Failed to update availability")
    }
  }

  return (
    <Container className="divide-y p-0">
      <div className="flex items-center justify-between px-6 py-4">
        <Heading level="h2">Availability</Heading>
        <div className="flex items-center gap-x-4">
          {isLoading ? (
            <Skeleton className="h-4 w-24" />
          ) : (
            <StatusBadge color={getStatusColor(status)}>
              {getStatusLabel(status)}
            </StatusBadge>
          )}
          <ActionMenu
            groups={[
              {
                actions: [
                  {
                    label: "Set Incoming",
                    onClick: () => handleSetAvailability("incoming"),
                    icon: <Loader className="text-orange-500" />,
                  },
                  {
                    label: "Set In Stock",
                    onClick: () => handleSetAvailability("in_stock"),
                    icon: <Check className="text-green-500" />,
                  },
                  {
                    label: "Set Out of Stock",
                    onClick: () => handleSetAvailability("out_of_stock"),
                    icon: <XMark className="text-red-500" />,
                  },
                ],
              },
            ]}
          />
        </div>
      </div>
    </Container>
  )
}

export const config = defineWidgetConfig({
  zone: "product.details.side.before",
})

export default ProductAvailability
```

### API Hooks

**File:** `src/admin/hooks/api/availabilities.ts`

```typescript
import { FetchError } from "@medusajs/js-sdk"
import {
  QueryKey,
  useMutation,
  UseMutationOptions,
  useQuery,
  UseQueryOptions,
} from "@tanstack/react-query"
import { sdk } from "../../lib/sdk"
import { queryClient } from "../../lib/query-client"
import { queryKeysFactory } from "../../lib/query-key-factory"

const AVAILABILITIES_QUERY_KEY = "availabilities" as const
export const availabilitiesQueryKeys = queryKeysFactory(AVAILABILITIES_QUERY_KEY)

type ProductAvailabilityResponse = {
  availability: {
    id: string
    product_id: string
    status: string
    notes?: string
    created_at: string
    updated_at: string
  }
}

type UpdateProductAvailabilityRequest = {
  status: "incoming" | "in_stock" | "out_of_stock"
  notes?: string
}

export const useProductAvailability = (
  productId: string,
  options?: Omit<
    UseQueryOptions<ProductAvailabilityResponse, FetchError, ProductAvailabilityResponse, QueryKey>,
    "queryFn" | "queryKey"
  >
) => {
  const { data, ...rest } = useQuery({
    queryFn: () =>
      sdk.client.fetch<ProductAvailabilityResponse>(
        `/admin/products/${productId}/availability`
      ),
    queryKey: availabilitiesQueryKeys.detail(productId),
    ...options,
  })

  return { ...data, ...rest }
}

export const useUpdateProductAvailability = (
  productId: string,
  options?: UseMutationOptions<
    ProductAvailabilityResponse,
    FetchError,
    UpdateProductAvailabilityRequest
  >
) => {
  return useMutation({
    mutationFn: (payload: UpdateProductAvailabilityRequest) =>
      sdk.client.fetch<ProductAvailabilityResponse>(
        `/admin/products/${productId}/availability`,
        {
          method: "POST",
          body: payload,
        }
      ),
    onSuccess: (data, variables, context) => {
      queryClient.invalidateQueries({
        queryKey: availabilitiesQueryKeys.detail(productId),
      })
      options?.onSuccess?.(data, variables, context)
    },
    ...options,
  })
}
```

---

## 🚀 Initialization Script

**File:** `src/scripts/initialize-availability.ts`

```typescript
import { ContainerRegistrationKeys } from "@medusajs/framework/utils"
import { updateProductAvailabilityWorkflow } from "../workflows/update-product-availability"
import { ExecArgs } from "@medusajs/framework/types"

export default async function initializeAvailability({ container }: ExecArgs) {
  const query = container.resolve(ContainerRegistrationKeys.QUERY)

  console.log("🚀 Initializing availability for existing products...")

  let offset = 0
  const batchSize = 50
  let totalProcessed = 0

  while (true) {
    // Get products in batches
    const { data: products } = await query.graph({
      entity: "product",
      fields: ["id", "title"],
      pagination: { take: batchSize, skip: offset },
    })

    if (!products || products.length === 0) break

    // Process each product
    for (const product of products) {
      try {
        await updateProductAvailabilityWorkflow(container).run({
          input: { product_id: product.id },
        })
        console.log(`✅ Initialized availability for: ${product.title}`)
        totalProcessed++
      } catch (error) {
        console.error(`❌ Failed for product ${product.id}:`, error)
      }
    }

    offset += batchSize
    console.log(`📊 Processed ${offset} products...`)
  }

  console.log(`🎉 Availability initialization complete! Total: ${totalProcessed} products`)
}
```

**Run the script:**

```bash
npx medusa exec src/scripts/initialize-availability.ts
```

---

## 📖 Frontend Integration

### Next.js Usage Example

```typescript
// hooks/useAvailableProducts.ts
import { useState, useEffect } from 'react'

export const useAvailableProducts = (filters = {}) => {
  const [products, setProducts] = useState([])
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState(null)

  useEffect(() => {
    const fetchProducts = async () => {
      try {
        const params = new URLSearchParams({
          limit: '20',
          offset: '0',
          ...filters
        })
        
        const response = await fetch(`/store/products/available?${params}`)
        const data = await response.json()
        
        setProducts(data.products)
      } catch (err) {
        setError(err)
      } finally {
        setLoading(false)
      }
    }

    fetchProducts()
  }, [filters])

  return { products, loading, error }
}

// components/ProductListing.tsx
export const ProductListing = ({ categoryId }) => {
  const { products, loading, error } = useAvailableProducts({
    category_id: categoryId,
    status: 'in_stock'
  })

  if (loading) return <div>Loading available products...</div>
  if (error) return <div>Error loading products</div>

  return (
    <div className="grid grid-cols-1 md:grid-cols-3 lg:grid-cols-4 gap-6">
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  )
}
```

---

## 🔍 Testing & Validation

### Test the Module

```bash
# 1. Create a test product
curl -X POST http://localhost:9000/admin/products \
  -H "Authorization: Bearer {admin_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Test Product",
    "handle": "test-product"
  }'

# 2. Check availability endpoint
curl http://localhost:9000/store/products/available

# 3. Update product availability
curl -X POST http://localhost:9000/admin/products/{product_id}/availability \
  -H "Authorization: Bearer {admin_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "in_stock",
    "notes": "Manual override"
  }'
```

### Monitoring & Debugging

Add logging to track availability updates:

```typescript
// In your workflow or subscribers
console.log(`Availability updated: ${product_id} -> ${status}`)
```

---

## 🎯 Benefits & Use Cases

### ✅ Benefits

- **Performance**: Efficient database queries with proper indexing
- **SEO**: Product pages remain accessible for search engines
- **User Experience**: Consistent pagination and accurate counts
- **Flexibility**: Manual overrides and automatic calculations
- **Scalability**: Handles large product catalogs efficiently

### 🛠️ Use Cases

- **E-commerce listings**: Show only available products on category pages
- **Inventory management**: Track product availability states
- **Marketing campaigns**: Filter products for promotional listings
- **Admin operations**: Manual availability control
- **API integrations**: Provide availability data to third-party systems

---

## 🔧 Customization Options

### Extended Status Types

```typescript
export enum AvailabilityStatus {
  INCOMING = "incoming",
  IN_STOCK = "in_stock",
  LOW_STOCK = "low_stock",    // Add threshold-based status
  OUT_OF_STOCK = "out_of_stock",
  DISCONTINUED = "discontinued", // Permanent unavailability
}
```

### Variant-Level Availability

Modify the model to link to variants instead of products:

```typescript
const Availability = model.define("availability", {
  id: model.id().primaryKey(),
  variant_id: model.text(), // Changed from product_id
  status: model.enum(AvailabilityStatus),
  threshold: model.number().default(0), // Stock threshold
})
```

### Multi-Location Support

Add location-specific availability:

```typescript
const Availability = model.define("availability", {
  id: model.id().primaryKey(),
  product_id: model.text(),
  location_id: model.text(),
  status: model.enum(AvailabilityStatus),
})
```

---

## 📚 Additional Resources

- [Medusa v2 Modules Documentation](https://docs.medusajs.com/learn/fundamentals/modules)
- [Module Links & Relationships](https://docs.medusajs.com/learn/fundamentals/module-links)
- [Workflows Documentation](https://docs.medusajs.com/learn/fundamentals/workflows)
- [Subscribers & Events](https://docs.medusajs.com/learn/fundamentals/events-and-subscribers)

---

## 🏁 Conclusion

This Availability Module provides a robust, scalable solution for managing product availability in Medusa v2. It follows best practices for module architecture while providing the flexibility needed for complex e-commerce scenarios.

The implementation automatically maintains availability status based on inventory changes while allowing manual overrides when needed. The admin dashboard integration makes it easy for store operators to manage availability, and the Store API endpoint provides efficient filtering for frontend applications.
