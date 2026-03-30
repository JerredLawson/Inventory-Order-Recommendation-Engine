# Inventory Order Recommendation Engine <br>(Tableau + Python)
## What does it do? 
- Replaces manual, subjective ordering decisions with a rule-based engine
- Uses Pareto (80/15/5) classification to segment products
- Recommends what to order and how much for each product
- Projects inventory levels over time to support ordering recommendations
- Generates a [report](https://jerredlawson.github.io/Inventory-Order-Recommendation-Engine/Inventory%20Order%20Recommendation%20Report.pdf)

## Why does it matter?
When implemented in a casino gift shop, the system:
- Reduced ordering time from ~2 hours to ~15 minutes
- Reduced stockouts on top-selling products
- Reduced overstock and spoilage on slow-moving products
- Standardized ordering recommendations across products

## Project outputs:
### **Inventory Order Recommendation Report** ([View PDF](https://jerredlawson.github.io/Inventory-Order-Recommendation-Engine/Inventory%20Order%20Recommendation%20Report.pdf))
 - Order quantities and projections based on historical sales velocity.
   <p><a href="https://jerredlawson.github.io/Inventory-Order-Recommendation-Engine/Inventory Order Recommendation Report.pdf">
    <img src="images/Inventory_Order_Recommendation_Report.png" alt="Inventory Order Recommendation Report" width="70%"></a>

### **Pareto Distribution** ([View Interactive Tableau Dashboard](https://public.tableau.com/app/profile/jerred.lawson/viz/RetailStoreViz/DASHABCInventoryClassification))<p>
Products are classified into A/B/C categories based on share of units sold.
- **Class A (Top 80%):** carry 2 weeks of buffer stock.
- **Class B (Next 15%):** carry 1 week of buffer stock.
- **Class C (Remaining 5%):** no additional buffer; flagged for review.
  <p><a href="https://public.tableau.com/app/profile/jerred.lawson/viz/RetailStoreViz/DASHABCInventoryClassification">
    <img src="images/Pareto_Dist.png" alt="Tableau Pareto Distribution" width="70%"></a>

## View the Project
 - [Python Engine]([url](https://github.com/JerredLawson/Inventory-Order-Recommendation-Engine/blob/main/python/inventory_order_engine.ipynb)) (Jupyter Notebook)
 - [Relational Data Model]([url](https://github.com/JerredLawson/Inventory-Order-Recommendation-Engine/blob/main/sql/Inventory_Order_Recommendation_Engine.sql)) (SQL / BigQuery)
 - [Excel Prototype]([url](https://github.com/JerredLawson/Inventory-Order-Recommendation-Engine/blob/main/excel/Inventory_Order_Recommendation_Engine.xlsx)) (Initial Model)
