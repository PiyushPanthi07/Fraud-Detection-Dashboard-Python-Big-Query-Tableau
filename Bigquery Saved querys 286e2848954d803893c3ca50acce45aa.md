# Bigquery Saved querys

- **Dimension Affiliate**
    
    ```
     CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.dim_affiliate` AS
    SELECT DISTINCT
      AFID AS afid,
      AFFID AS affid,
      AID AS aid,
      BID AS bid,
      SID AS sid,
      CAST(Campaign_Id AS STRING) AS campaign_id,
      Offer_Id AS offer_id,
      Promo_Code AS promo_code
    FROM `variant-data-analytics.Sticky_Data.Sticky_PD`;
    ```
    
- **Dimension Gateway**
    
    ```
    
     CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.dim_gateway` AS
    SELECT DISTINCT
      CAST(Gateway_Id AS STRING) AS gateway_id,
      Gateway_Alias AS gateway_alias,
      Gateway_Descriptor AS gateway_descriptor,
      Processor_Id AS processor_id,
      Gateway_Customer_Service_Number AS support_number,
      SAFE_CAST(Gateway_Processing_Percent AS FLOAT64) AS processing_percent,
      SAFE_CAST(Gateway_Chargeback_Fee AS FLOAT64) AS chargeback_fee
    FROM `variant-data-analytics.Sticky_Data.Sticky_PD`
    WHERE Gateway_Id IS NOT NULL;
    ```
    
- Dimension Customer
    
    ```
    CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.dim_customer` AS
    SELECT DISTINCT
      CAST(Customer_Number AS STRING) AS customer_id,
      Bill_First AS first_name,
      Bill_Last AS last_name,
      Bill_Email AS email,
      Bill_Phone AS phone,
      Bill_Country AS country,
      Bill_State AS state,
      Bill_City AS city,
      Bill_Zip AS zip,
      Blacklisted AS blacklist_flag
    FROM `variant-data-analytics.Sticky_Data.Sticky_PD`
    WHERE Customer_Number IS NOT NULL;
    ```
    
- Dimension Geography
    
    ```
    CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.dim_geography` AS
    SELECT DISTINCT
      CONCAT(COALESCE(Bill_Country, ''), '-', COALESCE(Bill_State, ''), '-', COALESCE(Bill_City, '')) AS geo_id,
      Bill_Country AS billing_country,
      Bill_State AS billing_state,
      Bill_City AS billing_city,
      Ship_Country AS shipping_country,
      Ship_State AS shipping_state,
      Ship_City AS shipping_city,
      IP_Address AS ip_address,
      IP_Address_Lookup AS ip_lookup
    FROM `variant-data-analytics.Sticky_Data.Sticky_PD`;
    ```
    
- Dimension Product
    
    ```
    CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.dim_product` AS
    SELECT DISTINCT
      CAST(Product_Id AS STRING) AS product_id,
      Product_Name AS product_name,
      Product_Category AS product_category,
      Product_Sku_# AS product_sku,
      SAFE_CAST(Product_Price AS FLOAT64) AS price,
      Billing_Cycle AS billing_cycle,
      Is_Product_Shippable AS shippable_flag
    FROM `variant-data-analytics.Sticky_Data.Sticky_PD`
    WHERE Product_Id IS NOT NULL;
    ```
    
- Dimension- Date
    
    ```
    CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.dim_date` AS
    WITH dates AS (
      SELECT DISTINCT SAFE_CAST(Date_of_Sale AS DATE) AS date_value
      FROM `variant-data-analytics.Sticky_Data.Sticky_PD`
      WHERE Date_of_Sale IS NOT NULL
    )
    SELECT
      date_value AS date_id,
      EXTRACT(YEAR FROM date_value) AS year,
      EXTRACT(QUARTER FROM date_value) AS quarter,
      EXTRACT(MONTH FROM date_value) AS month,
      EXTRACT(DAY FROM date_value) AS day,
      FORMAT_DATE('%B', date_value) AS month_name,
      FORMAT_DATE('%A', date_value) AS weekday_name
    FROM dates;
    ```
    
- Fact table
    
    ```
    CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.fact_transactions` AS
    SELECT
        -- Identifiers
        CAST(Order_Id AS STRING) AS order_id,
        CAST(Customer_Number AS STRING) AS customer_id,
        CAST(Campaign_Id AS STRING) AS campaign_id,
        CAST(AFID AS STRING) AS afid,
        CAST(AFFID AS STRING) AS affid,
        CAST(AID AS STRING) AS aid,
        CAST(BID AS STRING) AS bid,
        CAST(SID AS STRING) AS sid,
    
        -- Dates
        SAFE_CAST(Date_of_Sale AS DATE) AS order_date,
        SAFE_CAST(Refund_Date AS DATE) AS refund_date,
        SAFE_CAST(Chargeback_Date AS DATE) AS chargeback_date,
        SAFE_CAST(Fraud_Date AS DATE) AS fraud_date,
        SAFE_CAST(Void_Date AS DATE) AS void_date,
        SAFE_CAST(Recurring_Date AS DATE) AS recurring_date,
    
        -- Payment & Gateway
        Payment AS payment_method,
        CAST(Credit_Card_Number AS STRING) AS card_number_masked,
        Prepaid_Match AS prepaid_flag,
        CAST(Gateway_Id AS STRING) AS gateway_id,
        Gateway_Alias AS gateway_alias,
        Gateway_Descriptor AS gateway_descriptor,
    
        -- Transaction Status Flags
        CASE WHEN Is_Fraud = 'YES' THEN TRUE ELSE FALSE END AS fraud_flag,
        CASE WHEN Is_Chargeback = 'YES' THEN TRUE ELSE FALSE END AS chargeback_flag,
        CASE WHEN Is_Refund = 'YES' THEN TRUE ELSE FALSE END AS refund_flag,
        CASE WHEN Is_Cancel = 'YES' THEN TRUE ELSE FALSE END AS cancel_flag,
        CASE WHEN Is_Void = 'YES' THEN TRUE ELSE FALSE END AS void_flag,
        CASE WHEN Blacklisted = 'YES' THEN TRUE ELSE FALSE END AS blacklist_flag,
    
        -- Financials
        SAFE_CAST(Order_Total AS FLOAT64) AS order_total,
        SAFE_CAST(Refund_Amount AS FLOAT64) AS refund_amount,
        SAFE_CAST(Void_Amount AS FLOAT64) AS void_amount,
        SAFE_CAST(Quantity AS INT64) AS quantity,
        SAFE_CAST(Product_Price AS FLOAT64) AS product_price,
        SAFE_CAST(Sub_Total AS FLOAT64) AS sub_total,
        SAFE_CAST(Sales_Tax_Percent AS FLOAT64) AS sales_tax_percent,
        SAFE_CAST(Gateway_Processing_Percent AS FLOAT64) AS gateway_processing_percent,
        SAFE_CAST(Gateway_Chargeback_Fee AS FLOAT64) AS gateway_chargeback_fee,
    
        -- Product Details
        CAST(Product_Id AS STRING) AS product_id,
        Product_Name AS product_name,
        Product_Category AS product_category,
        `Product_Sku_#` AS product_sku,   -- ✅ escaped with backticks
        Billing_Cycle AS billing_cycle,
    
        -- Customer/Location
        Bill_Email AS customer_email,
        Ship_Email AS shipping_email,
        Bill_Phone AS billing_phone,
        Ship_Phone AS shipping_phone,
        Bill_Country AS billing_country,
        Ship_Country AS shipping_country,
        IP_Address AS ip_address,
        IP_Address_Lookup AS ip_geo,
    
        -- Declines / Reasons
        Decline_Reason AS decline_reason,
        Original_Decline_Reason AS original_decline_reason
    FROM `variant-data-analytics.Sticky_Data.Sticky_PD`;
    ```
    
- dim- geo 1
    
    ```
    CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.dim_geography1` AS
    SELECT DISTINCT
      CONCAT(COALESCE(Bill_Country, ''), '-', 
             COALESCE(NULLIF(Bill_State, ''), Ship_State, ''), '-', 
             COALESCE(Bill_City, '')) AS geo_id,
      Bill_Country AS billing_country,
      COALESCE(NULLIF(Bill_State, ''), Ship_State) AS billing_state,
      Bill_City AS billing_city,
      Ship_Country AS shipping_country,
      Ship_State AS shipping_state,
      Ship_City AS shipping_city,
      IP_Address AS ip_address,
      IP_Address_Lookup AS ip_lookup
    FROM `variant-data-analytics.Sticky_Data.Sticky_PD`;
    ```
    
- fraud by cities
    
    ```
    CREATE OR REPLACE TABLE `variant-data-analytics.Sticky_Data.fraud_by_city` AS
    SELECT
      billing_country,
      billing_city,
      COUNT(order_id) AS total_orders,
      SUM(CASE WHEN fraud_flag = TRUE THEN 1 ELSE 0 END) AS fraud_orders,
      SAFE_DIVIDE(
        SUM(CASE WHEN fraud_flag = TRUE THEN 1 ELSE 0 END),
        COUNT(order_id)
      ) AS fraud_rate
    FROM `variant-data-analytics.Sticky_Data.fact_transactions`
    GROUP BY billing_country, billing_city;
    ```