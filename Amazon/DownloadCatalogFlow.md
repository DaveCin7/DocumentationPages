# Amazon DownloadCatalog Flow - Complete Documentation

> **📋 Document Overview**  
> The Amazon DownloadCatalog process is a comprehensive system that synchronizes product catalog data from Amazon's Selling Partner API to the DEAR inventory management system. This document outlines the complete end-to-end flow from API calls to database storage and product matching.

## Architecture Components

### Core System Components

#### 1. Entry Point
- **Class:** `AmazonECSV2`
- **Method:** `DownloadCatalog()`
- **Location:** `Sim.Amazon/AmazonECSV2.cs`

#### 2. Core Processing Engine  
- **Class:** `DearSim`
- **Event Handler:** `ecs_ProductDownloaded(ProductInfo state)`
- **Location:** `Sim.DearCore/DearSim.cs`

#### 3. Amazon API Client
- **Class:** `AmazonClientV2`
- **Location:** `Sim.Amazon/AmazonClientV2.cs`

## Complete Flow Breakdown

<details>
<summary><strong>📊 Phase 1: Initialization and Report Setup</strong></summary>

```
1. User triggers catalog download
2. AmazonECSV2.DownloadCatalog() starts
3. Progress update: "Check for accessible inventory data..."
4. Initialize client: GetClient()
5. Get or request inventory report: ObtainReportId()
```

> **🔑 Key Details**
> - **Report Type:** `GET_MERCHANT_LISTINGS_DATA`
> - **Report Contains:** SKU-to-ASIN mappings, prices, quantities, product IDs
> - **Validation:** If report is null/outdated, process fails with error message

</details>

<details>
<summary><strong>🔄 Phase 2: Amazon API Data Retrieval</strong></summary>

### 2.1 Inventory Report Processing
```
6. Parse report into MerchantListingsData objects
7. Cache listings in m_cachedListingList
8. Progress update: "Downloading data... Please wait..."
```

### 2.2 Product Catalog API Calls
```
9. client.GetProductList() processes inventory data
10. For each product in inventory:
    - Extract productAsin and familyProductAsin
    - Map SKU using AmazonHelper.GetSkuForASIN()
    - Identify parent-child relationships
    - Separate regular vs family products
```

> **💡 API Endpoints Used**
> - **Individual Products:** `/catalog/2022-04-01/items/{asin}`
> - **Bulk Products:** `/catalog/2022-04-01/items` (SearchCatalogItems)
> - **Parameters:** `["attributes", "dimensions", "identifiers", "images", "productTypes", "summaries", "relationships"]`

### 2.3 Family Product Handling
```
11. For family products:
    - Collect parent ASINs in asinToDownload list
    - Call client.GetProductsByAsinList() for parent products
    - Map parent SKUs using AmazonHelper.GetSkuForASIN()
    - Process parent-child relationships
```

> **⚠️ Family Product Logic**
> - **Child Products:** Use individual seller SKUs
> - **Parent Products:** Use parent ASIN, fallback to parent ASIN as SKU if no mapping exists
> - **Relationship Mapping:** Uses `relationships.parentAsins` and `relationships.childAsins`

</details>

<details>
<summary><strong>🔄 Phase 3: Product Data Transformation</strong></summary>

### 3.1 Data Conversion
```
12. For each product/family product:
    - Regular products: Convertor.MapProduct(product, data, priceTier)
    - Family products: Convertor.MapProductFamily(familyProduct, products, listings, priceTier)
    - Extract ProductInfo with standardized fields
```

**Field Mapping Details:**
- **Primary ID:** `sellerSKU` (preferred) or `ASIN` (fallback)
- **Product Details:** Name, description, price, dimensions, images
- **Custom Fields:** ASIN stored in CustomField5, ID type in CustomField6
- **Relationships:** Parent-child product relationships preserved

### 3.2 Event Trigger
```
13. For each ProductInfo object:
    - Trigger ProductDownloaded?.Invoke(productInfo)
    - This calls ecs_ProductDownloaded in DearSim
```

</details>

<details>
<summary><strong>🎯 Phase 4: DEAR System Integration (DearSim Processing)</strong></summary>

### 4.1 Initial Validation and Setup
```
14. ecs_ProductDownloaded(ProductInfo state) receives product data
15. Validate: SKU must not be empty (throws ECSInfoException if empty)
16. Track progress: Update counters and statistics
```

### 4.2 Product Family Processing
```
17. CreateOrUpdateEcsProductFamilyRecord(state):
    - Check if product has variations (family product)
    - Create/update SIM family record in database
    - Map EcsFamilyID to internal GUID
    - Return simFamilyId for linking
```

### 4.3 Product Matching (if not manual mapping)
```
18. If !ManualMapping && simFamilyId exists:
    - MatchEcsProductFamily(simFamilyId):
      - Search for existing DEAR product families
      - Match by SKU, name, or other criteria
      - Create mapping links if matches found
```

### 4.4 Individual Product Processing
```
19. CreateOrUpdateEcsProductRecord(state, simFamilyId):
    - Process main product and all variations
    - For each product/variation:
      a. Create/update vEcsProduct record
      b. Store ECS-specific data (SKU, ASIN, etc.)
      c. Link to family if applicable
      d. Track in _downloadedCatalog list
```

### 4.5 Product Matching and Creation
```
20. For each ecsProduct in results:
    - If ProductID exists: Update existing DEAR product (if UseAsMasterSource)
    - If ProductID is null: Attempt MatchEcsProduct()
      a. Search DEAR products by SKU/name/criteria
      b. Create mapping if match found
      c. If no match and AutoCreateNewProductAllowed: CreateDearProduct()
```

</details>

<details>
<summary><strong>💾 Phase 5: Data Persistence and Tracking</strong></summary>

### 5.1 Database Operations
```
21. All product data stored in tenant database:
    - vEcsProduct records (individual products)
    - vEcsProductFamily records (family groupings)
    - ProductMappingLink (ECS to DEAR mappings)
    - ProductFamilyMappingLink (family mappings)
```

### 5.2 Progress Tracking
```
22. Maintain download statistics:
    - _downloadedCatalog: List of all processed products
    - _downloadedCatalogFamiliesIDs: List of family IDs
    - ProductsDownloaded: Running count
    - ProductsSkipped: Skipped items count
```

### 5.3 Telemetry and Logging
```
23. Log operations for monitoring:
    - UpdateDownloadedProductLinkToTelemetry()
    - UpdateDownloadedProductFamilyLinkToTelemetry()
    - Individual operation logging via AddLogEntry()
```

</details>

<details>
<summary><strong>🧹 Phase 6: Post-Processing and Cleanup</strong></summary>

### 6.1 Bulk Operations (when catalog download completes)
```
24. After all products processed:
    - Compare _downloadedCatalog vs existing mapped products
    - Identify products no longer in ECS
    - BulkUnlisting(): Mark removed products as "Cancelled"
    - Update product statuses and mappings
```

### 6.2 Final Reporting
```
25. Generate completion statistics:
    - Total products downloaded
    - Products created vs updated
    - Families processed
    - Mapping successes and failures
    - Error summary
```

</details>

## Key Data Structures

### ProductInfo (Amazon → DEAR Transfer Object)
```csharp
- ProductID: Primary identifier
- SKU: Seller SKU (preferred) or ASIN (fallback)
- EcsFamilyID: Parent product identifier for variations
- Name, Description, Price: Product details
- Images: Product image URLs
- Variations: List<VariationInfo> for family products
- CustomField5: ASIN storage
- CustomField6: ID type (ASIN/ISBN/UPC/EAN)
```

### MerchantListingsData (Amazon Report Data)
```csharp
- Sku: Seller SKU (field index 4)
- Asin1: Primary ASIN (field index 17)
- Price, Quantity: Inventory data
- IdType: Identifier type (1=ASIN, 2=ISBN, 3=UPC, 4=EAN)
```

### vEcsProduct (DEAR Database Record)
```csharp
- ECSProductID: Amazon product identifier
- ECSVariantID: Variation identifier
- ProductID: DEAR product GUID (when mapped)
- SIMFamilyID: Internal family grouping
- Status: Active/Cancelled/etc.
```

## Error Handling and Resilience

### System Resilience Features

#### 1. API Rate Limiting
- Built-in retry logic with exponential backoff
- Throttling detection and sleep intervals  
- Batch processing (20 items max per request)

#### 2. Data Validation
- SKU required validation
- ASIN format validation
- Marketplace availability checks

#### 3. Database Transactions
- Atomic operations for product creation/updates
- Rollback capability on failures
- Duplicate prevention logic

#### 4. Logging and Monitoring
- Comprehensive operation logging
- Telemetry data for performance monitoring
- Error tracking with stack traces

## Configuration Options

### Mapping Behavior Configuration

| Setting | Description | Impact |
|---------|-------------|---------|
| `ManualMapping` | Disable automatic product matching | Requires manual intervention for all mappings |
| `UseAsMasterSource` | Allow ECS to update DEAR products | Amazon data overwrites DEAR product details |
| `AutoCreateNewProductAllowed` | Auto-create new DEAR products | New products created automatically for unmatched Amazon items |

### Processing Control Configuration  

| Setting | Description | Default |
|---------|-------------|---------|
| `PrimaryPriceTier` | Price tier for product pricing | 1 |
| Batch size limits | API request batching | 20 items |
| Report refresh intervals | How often to refresh inventory reports | Configurable |

## Performance Characteristics

> **⚡ System Performance**
> 
> **Scalability**
> - Processes thousands of products efficiently
> - Batch API calls minimize request overhead
> - Database bulk operations for large catalogs
> 
> **Reliability**  
> - Checkpoint-based processing
> - Resume capability after failures
> - Data consistency validation

This comprehensive flow ensures robust, scalable synchronization of Amazon product catalogs with the DEAR inventory management system while maintaining data integrity and providing extensive monitoring capabilities.
