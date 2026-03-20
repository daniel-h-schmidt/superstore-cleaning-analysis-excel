# Data Cleaning Log
Records violating hard business constraints were removed. Records with ambiguous validity were retained but flagged to preserve analytical flexibility.

### Issue: Postal Code - formated as whole number, missing leading zeros
The raw dataset contains the postal codes as whole numbers and is already missing the leading "0" on any postal codes that begin with "0". A new column was created and the formula
```excel
Text.PadStart(Text.From([Postal Code]), 5, "0")
```
was used to replace the missing "0". The original Postal Code column was deleted.

### Issue: Region - contains blank and "--" values
Upon closer inspection it was apperant that Region was derived from State and since the State column is complete, records missing the Region value were filled in to match records that included Region based on the State column. To replace the blank and "--" values with the correct region, the query was duplicated and the new query was grouped by State and a new region column was created using the Max operation on the old Region column. This query was then merged with the original query and the grouped columns were expanded. The old Region column containing blank and "--" values was deleted.

### Issue: Category - contains inconsistant nameing
Names were "Furni", "OfficeSupply", "Tech", "technologies". These were renamed to "Furniture", "Office Supplies", and "Technologies" respectively. Then all words were caplitalized so "technologies" became "Technologies".

### Text Columns - trimmed and cleaned
All columns with the text data type were trimmed and cleaned.

### Issue: Duplicate Rows
The dataset includes duplicate rows. These were removed based on the Order ID and Product ID columns becuase Order ID is not unique and appears once for each product purchased.

### Row ID Column Removed
### Invalid Low-Value Sales Records
**Issue:**

The dataset contained approximately 5,000 records where Sales < 1, most commonly with a value of 0.01. These records spanned a wide range of product categories, including high-value items such as printers and vacuums.

**Validation:**

These values were evaluated against basic business logic and determined to be economically implausible. Even low-cost retail items are unlikely to be sold at $0.01, particularly in the quantities observed.

**Conclusion:**

These records were classified as invalid and likely represent corrupted or synthetic data introduced as part of the data quality challenge.

**Action Taken:**

All rows where Sales < 1 were flagged as invalid and removed from the cleaned dataset prior to analysis.

### Invalid Margin Records
**Issue:**

Some records have a margin > 100%.

**Validation:**

Because margin is a percentage of revenue, any value over 100% is invalid.

**Action Taken:**

All records where the margin was greater than 100% were removed from the dataset prior to analysis.

### Invalid High Loss Sales

**Issue:**

Some values in the margin column are negative (e.g. -16000%).

**Validation:**

After looking at the distribution of the margin values below 0 it was determined that values below -300% are invalid (the count of values below -300% is 0 unitl less than -1000% showing a gap between the plausible losses and the corrupt or synthetic data).

**Action Taken:**

Rows where margin is less than -300% were marked as invalid and removed from the dataset prior to analysis.

### Extreme Sales Outliers
**Issue:**

The dataset contained a small number of unusually high sales values that appeared inconsistent with the rest of the data.

**Detection Method:**

A scatter plot of sales values revealed a noticeable drop-off in frequency above approximately 5,000, suggesting a potential group of outliers.

**Validation:**

All records above this range were manually reviewed. A subset of these records contained economically implausible transactions, where high sales values were associated with low-cost items (e.g., binders).

**Conclusion:**

These invalid records were found to cluster among the highest sales values, specifically those exceeding approximately 25,000. While this threshold was not statistically derived, it represented a clear boundary separating valid and invalid observations based on manual validation.

**Action Taken:**

All records with Sales > 25,000 were removed from the cleaned dataset.

## Discounts = *null*
**Issue:**

Some rows have a discount value of null. It can be assumed that null = 0 in some cases, however, in cases with highly negative margins (below -50%) that cannot be asssumed.

**Detection Method:**

For rows where Discount = 0 the lowest known margin is -45%. The average loss on a product where the discount = 0 and the margin is negative is -8% and anything below -18% is an outlier. Typical loss leaders are between -5% and -20% so we cannot assume Discount = 0 for records where margin < -20%.

**Validation:**

All records where Discount = "" and profit margin below -20% cannot be assumed to have Discount = 0 becuase it would defy business and statistical logic and were therefore flagged.

**Action Taken:** 

The flagged records are excluded from calculations but remain in the dataset becuase they are not erronius, they are incomplete. The remaining records where Margin >= -20% and Discount = null were updated to discount = 0

## High Variation in Sales Between Orders of the Same Product

**Issue:** 

Undiscounted Unit price is a calculated column: (Sales / Quantity / (1 - Discount)). Upon inspection of this column it became apparent that some products had extremely wide ranges in price (e.g. Staples Min price: 0.14 Max price: 325) with most prices being close together and some outliers creating the high variation between orders (e.g. median price of Staples: 11.35).

**Detection Method:**

A pivot table was created with Product Name as Rows and the Count, Max, Min, Median, and Average of Undiscounted Unit Price as Values. Range (Max - Min) and Max/Min (Max / Min) were added as calculated columns. The Max/Min column calculates how many times greater the Max value is from the Min value (e.g. Max/Min Staples: 2223). Outlier detection was explored using both global and product-level methods. Due to high variability across product categories, a product-relative median ratio approach was preferred for identifying unusually priced transactions. Records where Undiscounted Unit Price > 10 * Median and < 0.1 * Median were flagged as outliers

**Conclusion:**

Product Name does not uniquely identify a product. Significant variation in unit price suggests aggregation of multiple product configurations (e.g., packaging, vendor, or version differences).

Due to extreme right-skew in unit price distributions, median price is used as a more robust measure of central tendency.

It is also notable that the variation is correlated with discount percent. Higher discounts are associated with higher pre-discount prices, suggesting potential price inflation prior to discounting.

**Action Taken:**

Flagged records remain in the dataset. The main purpose of the flag is to be aware that flagged sales are likely a significanly different product under the same name.

## Proft = *null*

**Issue:**

Some records have a missing profit value. Because there is no Cost column or equivelent Profit cannot be calculated.

**Conclusion:**

The records that are missing Profit data are not invalid, but they are missing key business information. Calculating something like Average Profit, including this data would make Average Proft seem much smaller than it is.

**Action Taken:**

Records where profit = *null* were flagged and excluded from calculations.

# Additional Checks That Came Up Clean

**Is the duration from Order Date to Ship Date logical?** The duration was calculated and the maximum value was 5 days and the minuimum value was 0 days (same day shipping).

**Is there more than one distinct Order Date per Order ID?** No. Each Order ID is associated with only one distinct Order Date.

**Are there extreme or "0" values in the Quantity column?** No, quantity contains values from 1 to 14.

## Validation Column Formula

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
    TRUE,"Valid")
```
All records where Validity = "Invalid" are deleted as they defy business and/or statistical logic. Other records are given unique flags and excluded from calculations as including them could distort results.