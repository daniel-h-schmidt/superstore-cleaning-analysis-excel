**Data Cleaning Log**
=====================

**Data Cleaning Strategy**
--------------------------

The cleaning process followed a structured approach:

1.  Structural corrections (formatting, duplicates, text standardization)
    
2.  Calculated columns
    
3.  Handling missing and inconsistent values
    
4.  Business rule validation
    
5.  Outlier detection and classification
    
6.  Preservation of ambiguous records through flagging
    

Records violating clear business constraints were removed. Records with ambiguous validity were retained but flagged to preserve analytical flexibility.

**1\. Structural Corrections**
==============================

**Postal Code Formatting**
--------------------------

**Issue:** 

Postal codes were stored as whole numbers, resulting in missing leading zeros for codes beginning with “0”.

**Action Taken:** 

A new column was created using the following transformation to enforce 5-digit formatting:
```excel
Text.PadStart(Text.From([Postal Code]), 5, "0")
```

The original Postal Code column was removed.

**Category Standardization**
----------------------------

**Issue:**

Category values contained inconsistent naming (e.g., “Furni”, “OfficeSupply”, “Tech”, “technologies”).

**Action Taken:**

*   Values were standardized to:
    
    *   Furniture
        
    *   Office Supplies
        
    *   Technologies
        
*   All category values were normalized to consistent capitalization
    

**Text Cleaning**
-----------------

**Issue:**

Text fields contained leading/trailing whitespace and minor inconsistencies.

**Action Taken:**

All text columns were trimmed and cleaned.

**Duplicate Records**
---------------------

**Issue:**

Duplicate rows were present in the dataset.

**Validation:**

Order ID is not unique and appears once per product within an order. Therefore, duplicates were defined based on the combination of Order ID and Product ID.

**Action Taken:**

Duplicate records were removed using Order ID + Product ID as the unique key.

**Removal of Row ID Column**
----------------------------

**Issue:**

Row ID served no analytical purpose.

**Action Taken:**

Column removed.

**2\. Calculated Columns**
==========================

Calculated columns were created to support validation, normalization, and analytical consistency.

**Margin**
----------

**Definition:**
```excel
=[@Profit]/[@Sales]
```
**Purpose:**

*   Validate consistency between Profit and Sales
    
*   Identify invalid margin values (e.g., >100% or extreme negatives)
    
*   Provide a key metric for profitability analysis
    

**Unit Price**
--------------

**Definition:**
```excel
=[@Sales]/[@Quantity]
```
**Purpose:**

*   Standardize price at the unit level
    
*   Enable comparison of pricing across transactions
    
*   Support analysis of pricing patterns and anomalies
    

**Undiscounted Unit Price**
---------------------------

**Definition:**
```excel
=[@Sales]/[@Quantity]/(1-[@Discount])
```
**Purpose:**

*   Normalize pricing by removing the effect of discounts
    
*   Enable consistent comparison of base prices across transactions
    
*   Allow aggregation at the Product Name level without distortion from varying discount rates
    

**Rationale:**

Initial analysis using Unit Price required grouping by both Product Name and Discount, which limited interpretability. The Undiscounted Unit Price metric allowed for product-level aggregation while preserving comparability across transactions.

This column was primarily used for detecting pricing anomalies and identifying inconsistencies within product groupings.

**Validation Column**
---------------------

**Purpose:**

*   Consolidate all validation rules into a single classification field
    
*   Distinguish between:
    
    *   Invalid records (removed)
        
    *   Ambiguous records (flagged)
        
    *   Valid records
        

This column serves as the central mechanism for controlling data inclusion in analysis.

**3\. Missing and Inconsistent Values**
=======================================

**Region Missing or Invalid Values**
------------------------------------

**Issue:**

The _Region_ column contained blank and placeholder values (“--”).

**Validation:**

Region was determined to be functionally dependent on _State_, which contained complete data.

**Action Taken:**

*   A grouped table was created by State
    
*   Region was derived using the Max aggregation (assuming consistency within each state)
    
*   The result was merged back into the original dataset
    
*   The original Region column was replaced
    

**Missing Discount Values**
---------------------------

**Issue:**

Some records contained null Discount values.

**Validation:**

*   For Discount = 0, the lowest observed margin was -45%
    
*   The average margin of unprofitable sales is -8% and sales with margins below -18% are statistical outliers
    
*   Typical loss-leading margins range between -5% and -20%
    
*   Records with margin < -20% cannot reliably assume Discount = 0
    

**Action Taken:**

*   Records where Discount is null and margin < -20% were flagged as **Discount Unknown** and excluded from calculations
    
*   Remaining records with Discount = null were updated to Discount = 0
    

**Missing Profit Values**
-------------------------

**Issue:**

Some records contained null Profit values.

**Validation:**

Profit cannot be derived due to the absence of cost data.

**Conclusion:**

These records are incomplete but not invalid.

**Action Taken:**

Records with missing Profit were flagged as **Profit Unknown** and excluded from calculations.

**4\. Business Rule Validation**
================================

**Invalid Low-Value Sales**
---------------------------

**Issue:**

Approximately 5,000 records had Sales < 1, often equal to 0.01.

**Validation:**

Such values are economically implausible across all product categories and likely represent corrupted or synthetic data.

**Action Taken:**

All records where Sales < 1 were classified as invalid and removed.

**Invalid Margin Values (>100%)**
---------------------------------

**Issue:**

Some records had margin values greater than 100%.

**Validation:**

Margin represents a percentage of revenue and cannot exceed 100%.

**Action Taken:**

All records with margin > 100% were removed.

**Invalid Extreme Negative Margins**
------------------------------------

**Issue:**

Some records contained extremely negative margins (e.g., -16000%).

**Validation:**

Analysis of the margin distribution showed a clear separation between plausible losses and extreme outliers. Values below -300% were deemed invalid.

**Action Taken:**

All records with margin < -300% were removed.

**5\. Outlier Detection and Classification**
============================================

**Extreme Sales Outliers**
--------------------------

**Issue:**

A small number of records contained unusually high sales values.

**Detection Method:**

A scatter plot revealed a sharp drop-off in frequency above approximately 5,000.

**Validation:**

Manual review identified economically implausible transactions, particularly where high sales values were associated with low-cost items.

**Conclusion:**

Invalid records clustered above approximately 25,000 in sales. While this threshold was not statistically derived, it represented a clear boundary separating valid and invalid observations based on manual validation.

**Action Taken:**

All records with Sales > 25,000 were removed.

**High Variation in Unit Price Within Products**
------------------------------------------------

**Issue:**

Undiscounted Unit Price (Sales / Quantity / (1 - Discount)) showed extreme variation within the same Product Name.

**Detection Method:**

A pivot table was created with Product Name as rows and summary statistics (min, max, median, average). Additional metrics included:

*   Range (Max - Min)
    
*   Max/Min ratio
    

A product-relative approach was used to detect outliers:

*   Unit Price > 10 × Median
    
*   Unit Price < 0.1 × Median
    

**Conclusion:**

*   Product Name does not uniquely define a product
    
*   Variation likely reflects differences in packaging, vendor, or configuration
    
*   Unit price distributions are highly right-skewed; median is a more reliable measure than mean
    
*   Higher discounts are associated with higher pre-discount prices, suggesting potential price inflation prior to discounting
    

**Action Taken:**

Outliers were flagged but retained, as variation reflects data limitations rather than clear errors.

**6\. Flagging Strategy**
=========================

**Validation Column Logic**
---------------------------
```excel
=IFS(
    [@Sales]<1,"Invalid",
    [@Margin]>1,"Invalid",
    [@Margin]<-3,"Invalid",
    [@Sales]>25000,"Invalid",
    [@Profit]="","Profit Unknown",
    AND(
        [@Margin]<-0.2,
        [@Discount]=""
    ),"Discount Unknown",
    AND(
        [@Margin]>=-0.2,
        [@Discount]=""
    ),"Update null to 0",
    OR(
        [@[Undiscounted Unit Price]]>XLOOKUP(
            [@[Product Name]],
            unit_price_var[Product Name],
            unit_price_var[Undiscounted Median],
            0,
            0,
            1
        )*10,
        [@[Undiscounted Unit Price]]<XLOOKUP(
            [@[Product Name]],
            unit_price_var[Product Name],
            unit_price_var[Undiscounted Median],
            0,
            0,
            1
        )/10
    ), "Undiscounted Unit Price Outlier",
    TRUE,"Valid"
)
```

**Interpretation:**

*   Records labeled **Invalid** were removed
    
*   Other flagged records were retained but excluded from calculations where appropriate
    

**Additional Validation Checks**
================================

*   **Order Date vs Ship Date:** All durations fall between 0 and 7 days
    
*   **Order ID Consistency:** Each Order ID corresponds to a single Order Date
    
*   **Quantity Values:** All values fall between 1 and 14