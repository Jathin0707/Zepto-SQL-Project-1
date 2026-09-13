# Zepto-SQL-Project-1
-- ============================================================
-- ZEPTO E-COMMERCE SQL PORTFOLIO PROJECT
-- Complete File: Table Creation + EDA + Data Cleaning + Business Analysis
-- Database: PostgreSQL
-- Author: Portfolio Project for Data Analyst Role
-- ============================================================

-- 1. DATABASE & TABLE CREATION
-- Drop table if exists for clean run
DROP TABLE IF EXISTS zepto;

CREATE TABLE zepto (
  sku_id SERIAL PRIMARY KEY,
  category VARCHAR(120),
  name VARCHAR(150) NOT NULL,
  mrp NUMERIC(8,2),
  discountPercent NUMERIC(5,2),
  availableQuantity INTEGER,
  discountedSellingPrice NUMERIC(8,2),
  weightInGms INTEGER,
  outOfStock BOOLEAN,
  quantity INTEGER
);

-- 2. DATA IMPORT
-- Use this in pgAdmin or psql. Update file path as per your system.
-- \copy zepto(category,name,mrp,discountPercent,availableQuantity,discountedSellingPrice,weightInGms,outOfStock,quantity) FROM 'C:/path/to/zepto_v2.csv' WITH (FORMAT csv, HEADER true, DELIMITER ',', QUOTE '"', ENCODING 'UTF8');

-- If using pgAdmin Import Tool: Right click table -> Import/Export -> Select CSV (Save CSV as CSV UTF-8 first)


-- ============================================================
-- 3. DATA EXPLORATION (EDA)
-- ============================================================

-- Q1. Total number of records
SELECT COUNT(*) AS total_records FROM zepto;

-- Q2. Sample data (first 10 rows)
SELECT * FROM zepto LIMIT 10;

-- Q3. Check for NULL values in each column
SELECT 
  COUNT(*) - COUNT(category) AS category_nulls,
  COUNT(*) - COUNT(name) AS name_nulls,
  COUNT(*) - COUNT(mrp) AS mrp_nulls,
  COUNT(*) - COUNT(discountPercent) AS discount_nulls,
  COUNT(*) - COUNT(availableQuantity) AS qty_nulls,
  COUNT(*) - COUNT(discountedSellingPrice) AS selling_price_nulls,
  COUNT(*) - COUNT(weightInGms) AS weight_nulls,
  COUNT(*) - COUNT(outOfStock) AS stock_nulls,
  COUNT(*) - COUNT(quantity) AS quantity_nulls
FROM zepto;

-- Q4. Distinct product categories
SELECT DISTINCT category FROM zepto ORDER BY category;
SELECT COUNT(DISTINCT category) AS total_categories FROM zepto;

-- Q5. In-stock vs Out-of-stock count
SELECT outOfStock, COUNT(*) AS product_count
FROM zepto
GROUP BY outOfStock;

-- Q6. Products appearing multiple times (Same product, different SKU/size)
SELECT name, COUNT(*) AS sku_count
FROM zepto
GROUP BY name
HAVING COUNT(*) > 1
ORDER BY sku_count DESC
LIMIT 10;

-- Q7. Products with MRP = 0 or Selling Price = 0 (Data quality check)
SELECT * FROM zepto WHERE mrp = 0 OR discountedSellingPrice = 0;


-- ============================================================
-- 4. DATA CLEANING
-- ============================================================

-- Q8. Remove rows where MRP or discounted price is zero (invalid data)
DELETE FROM zepto WHERE mrp = 0 OR discountedSellingPrice = 0;

-- Q9. Convert MRP and Selling Price from Paise to Rupees
UPDATE zepto SET mrp = mrp / 100.0;
UPDATE zepto SET discountedSellingPrice = discountedSellingPrice / 100.0;

-- Verification after cleaning
SELECT name, mrp, discountedSellingPrice, discountPercent 
FROM zepto LIMIT 10;


-- ============================================================
-- 5. BUSINESS INSIGHTS & ANALYSIS
-- ============================================================

-- Q10. Top 10 Best-Value Products Based on Discount Percentage
SELECT DISTINCT name, category, mrp, discountPercent, discountedSellingPrice
FROM zepto
ORDER BY discountPercent DESC
LIMIT 10;

-- Q11. High MRP Products Currently Out of Stock (Revenue Leak)
SELECT name, category, mrp, availableQuantity
FROM zepto
WHERE outOfStock = TRUE
ORDER BY mrp DESC
LIMIT 10;

-- Q12. Estimated Potential Revenue by Category
SELECT category,
       SUM(discountedSellingPrice * availableQuantity) AS potential_revenue
FROM zepto
GROUP BY category
ORDER BY potential_revenue DESC;

-- Q13. Expensive Products (MRP > 500) with Minimal Discount (< 5%)
SELECT name, category, mrp, discountPercent, discountedSellingPrice
FROM zepto
WHERE mrp > 500 AND discountPercent < 5
ORDER BY mrp DESC;

-- Q14. Top 5 Categories Offering Highest Average Discounts
SELECT category,
       ROUND(AVG(discountPercent)::numeric, 2) AS avg_discount
FROM zepto
GROUP BY category
ORDER BY avg_discount DESC
LIMIT 5;

-- Q15. Price Per Gram Analysis to Find Value-for-Money Products
SELECT name, category, discountedSellingPrice, weightInGms,
       ROUND((discountedSellingPrice / NULLIF(weightInGms, 0))::numeric, 2) AS price_per_gram
FROM zepto
WHERE weightInGms > 0
ORDER BY price_per_gram ASC
LIMIT 10;

-- Q16. Group Products by Weight Category (Low, Medium, Bulk)
SELECT name, weightInGms,
  CASE 
    WHEN weightInGms < 250 THEN 'Low Weight'
    WHEN weightInGms BETWEEN 250 AND 1000 THEN 'Medium Weight'
    ELSE 'Bulk Weight'
  END AS weight_category
FROM zepto;

-- Q17. Total Inventory Weight Per Product Category
SELECT category,
       SUM(availableQuantity * weightInGms) / 1000 AS total_inventory_weight_kg
FROM zepto
GROUP BY category
ORDER BY total_inventory_weight_kg DESC;

-- Q18. Revenue Contribution by Stock Status
SELECT outOfStock,
       SUM(discountedSellingPrice * availableQuantity) AS revenue
FROM zepto
GROUP BY outOfStock;

-- Q19. Category-wise Stock Availability
SELECT category,
       COUNT(*) AS total_skus,
       SUM(CASE WHEN outOfStock = FALSE THEN 1 ELSE 0 END) AS in_stock_skus,
       SUM(CASE WHEN outOfStock = TRUE THEN 1 ELSE 0 END) AS out_of_stock_skus
FROM zepto
GROUP BY category
ORDER BY total_skus DESC;

-- Q20. Most Expensive Product in Each Category (Using Window Function)
SELECT category, name, mrp
FROM (
  SELECT category, name, mrp,
         RANK() OVER (PARTITION BY category ORDER BY mrp DESC) as rnk
  FROM zepto
) ranked
WHERE rnk = 1;

-- END OF PROJECT
