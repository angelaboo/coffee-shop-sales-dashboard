# Coffee Shop Sales Analytics Dashboard

## Overview
This **Power BI dashboard** provides actionable insights into **Chatcup Coffee Chain**'s sales performance, enabling data-driven decisions to optimize inventory, staffing, and store operations. Designed for business stakeholders, it answers critical questions like:  
- **When** are peak sales hours, and how do sales trends vary hourly?  
- **What** are the top-selling products across categories (Coffee, Tea, Bakery)?  
- **Where** are the highest-performing store locations?  

Leveraging **DAX formulas** and a **star schema data model**, the dashboard dynamically tracks:  
- Total Sales, Average Transaction Value, and Total Transactions
- Hourly Sales Trends
- Product Sales Performance, with drilldowns
- Revenue Distribution Across Stores
- Month-over-Month (MoM) Growth

![Dashboard Snapshot](https://github.com/angelaboo/coffee-shop-sales-dashboard/blob/19d06d0ab7d560252ba2e6369adddf7a96664f65/dashboard/Coffee%20Shop%20Sales%20Dashboard.jpg)  

<br>

## Data Sources

Source: Dataset provided by **[Maven Analytics - Data Playground](https://mavenanalytics.io/data-playground)**.

- **[Coffee Shop Sales.xlsx](https://github.com/angelaboo/coffee-shop-sales-dashboard/blob/19d06d0ab7d560252ba2e6369adddf7a96664f65/dataset/Coffee%20Shop%20Sales.xlsx)**:  
  Transactional data from January to June 2023, including:  
  - `transaction_id`, `transaction_date`, `transaction_time`
  - `product_id`, `product_category`, `product_type`, `product_detail`, `unit_price`, `transaction_qty`  
  - `store_id`, `store_location`  

- **[Sales Data Dictionary.xlsx](https://github.com/angelaboo/coffee-shop-sales-dashboard/blob/19d06d0ab7d560252ba2e6369adddf7a96664f65/dataset/Sales%20Data%20Dictionary.xlsx)**:  
  Provides metadata, including column names and their descriptions, for understanding the dataset.

<br>

## Methodology

### Data Preparation
Before proceeding with data modeling, the dataset was imported, explored, and transformed to ensure its quality and readiness. To keep the focus on the data modeling process, the details of these preparatory steps are not included here.

### Data Modeling
#### Star Schema Design
To optimize query performance and simplify analysis, the data was structured into a **star schema**:  

- **Fact Table**: `Fact_Sales`  
  - Contains transactional metrics:  
    - `transaction_id`, `transaction_qty`, `unit_price`, `date_key`, `time_key`, `product_id`, `store_id`  

- **Dimension Tables**:  
  - `Dim_Date`: Date attributes (`year`, `quarter`, `month`, `day`, etc.).  
  - `Dim_Time`: Hourly attributes (`hour_label`, `part_of_day`, etc.).  
  - `Dim_Product`: Product attributes (`product_category`, `product_type`, `product_detail`).  
  - `Dim_Store`: Store details (`store_id`, `store_location`).  

![Star Schema](https://github.com/angelaboo/coffee-shop-sales-dashboard/blob/60e57e3296bfe00a190a81919b2372ed17a7c743/power-bi-star-schema-data-model.png)  

#### Relationships & Surrogate Keys
- **Surrogate Keys**:  
  Artificial identifiers (e.g., `date_key = 20230101` for `1/1/2023`) replace natural keys (like dates or text) to:  
  - Improve join performance.  
  - Avoid duplication errors (e.g., two stores named "Downtown").  

- **Active Relationships**:  
  - `Fact_Sales[date_key]` → `Dim_Date[date_key]`  
  - `Fact_Sales[time_key]` → `Dim_Time[time_key]`  
  - `Fact_Sales[product_id]` → `Dim_Product[product_id]`  
  - `Fact_Sales[store_id]` → `Dim_Store[store_id]`  


### DAX Formulas

#### 1. Total Sales  
- **Purpose**: Calculates total revenue by summing the product of transaction quantity and unit price.  
- **Why `SUMX`?** Handles row-level calculations to ensure accurate aggregation.  
```dax
Total Sales = 
SUMX(
    Fact_Sales, 
    Fact_Sales[transaction_qty] * Fact_Sales[unit_price]
)
```

#### 2. Average Transaction Value
- **Purpose**: Determines the average spend per transaction.price.  
- **Why `DISTINCTCOUNT`?** Determines the average spend per transaction.
```dax
Avg Transaction Value = 
DIVIDE(
    [Total Sales], 
    DISTINCTCOUNT(Fact_Sales[transaction_id]),  // Unique transactions
    BLANK()
)
```

#### 3. Total Transactions
- **Purpose**: Counts unique transactions to analyze customer activity or sales trends.
```dax
Total Transactions = 
DISTINCTCOUNT(Fact_Sales[transaction_id])  // Unique transaction IDs
```

#### 4. Top 5 Products by Total Sales
- **Purpose**: Identifies the 5 highest-selling products using dynamic ranking.
- **Key Functions**: 
    - `SUMMARIZE`: Creates a virtual table of products and their total sales.
    - `RANKX`: Ranks products by sales, with `DENSE` to handle ties.
    - `TOPN`: Filters the ranked list to return only the top 5 products.
```dax
Top 5 Products = 
VAR ProductsWithSales =
    SUMMARIZE(
        Fact_Sales,
        Dim_Product[product_detail],
        "Total Sales", [Total Sales]
    )
VAR RankedProducts =
    ADDCOLUMNS(
        ProductsWithSales,
        "Rank", RANKX(
            ProductsWithSales,
            [Total Sales],
            , 
            DESC, 
            Dense  // No gaps in ranking (e.g., 1,1,2,3)
        )
    )
RETURN
TOPN(5, RankedProducts, [Total Sales], DESC)
```

#### 5. Total Sales Variance % by Year-Month
- **Purpose**: Tracks monthly sales growth to identify trends.
- **Key Considerations**:
    - Uses `DATEADD` for time-shifting, requiring Dim_Date to be marked as a date table.
    - Skips the first month (Jan 2023) with `BLANK()` to avoid misleading values.
```
Previous Month Sales = 
CALCULATE(
    [Total Sales],
    DATEADD(Dim_Date[transaction_date], -1, MONTH)
)
```
```
MoM Sales Variance % = 
IF(
    NOT ISBLANK([Previous Month Sales]),
    DIVIDE([Total Sales] - [Previous Month Sales], [Previous Month Sales]),
    BLANK()
)
```

<br>

## Dashboard Insights  

### 1. Product Performance  
- **Top Categories**:  
  - ☕ **Coffee** (highest sales) → 🍵 **Tea** → 🥐 **Bakery**  
  - *Action*: Bundle coffee with bakery items to increase average order value.  
- **Top-Selling Products**:  
  1. **Sustainably Grown Organic Lg**  
  2. **Dark Chocolate Lg**  
  3. **Latte Rg**  
  - *Action*: Promote top sellers through combo deals (e.g., coffee + pastry discounts).  


### 2. Peak Sales Hours  
- **Morning Rush**: 🕗 **8:00 AM – 10:00 AM** drives the largest share of daily sales.  
- **Sales Distribution**:  
  - 🌅 **Morning**: Highest volume  
  - ☀️ **Afternoon**: Moderate traffic  
  - 🌙 **Evening**: Lowest sales  
  - *Action*: Increase staff during morning hours and test "afternoon happy hour" promotions.  


### 3. Store Performance  
- **Revenue Ranking**:  
  - 🏆 **Hell’s Kitchen** (highest sales) → 🏙️ **Astoria** → 🏛️ **Lower Manhattan**  
  - *All stores perform similarly, but Hell’s Kitchen leads marginally.*  
  - *Action*: Analyze Hell’s Kitchen’s customer engagement strategies (e.g., loyalty programs, in-store experience).  



### 4. Actionable Insights  
1. **Leverage Coffee Dominance**: Prioritize stock and promotions for coffee products.  
2. **Staffing Alignment**: Focus resources on the **8–10 AM rush**.  
3. **Store Optimization**: Replicate Hell’s Kitchen’s successful tactics in other stores.  
4. **Seasonal Campaigns**: Capitalize on high-growth periods (March–May).


<br>

## **Author**
**Angela Boo**  
- **GitHub**: [GitHub](https://github.com/angelaboo)  
- **Kaggle**: [Kaggle](https://www.kaggle.com/xiaotingb)  
- **LinkedIn**: [Connect with Me](https://www.linkedin.com/in/xxtt)  

Feel free to connect and explore my work!
