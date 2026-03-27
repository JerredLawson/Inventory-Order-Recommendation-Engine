# Automated ABC Inventory Recommendation Engine (Tableau + Python)
- Transformed manual ordering into an automated process for a casino gift shop.
- Reduced ordering time from 2 hours to 15 minutes.
- Lowered stockouts on top-selling items and reduced spoilage on slow-movers.

# Project Outputs
**Pareto Sales Distribution** ([View Interactive Tableau Dashboard](https://public.tableau.com/app/profile/jerred.lawson/viz/RetailStoreViz/DASHABCInventoryClassification))
- Products are classified into A, B, and C categories based on the 80/15/5 rule.
- Class A items (Top 80% of sales) carry 2 weeks of buffer stock.
- Class B items (Next 15% of sales) carry 1 week of buffer stock.
- Class C items (Bottom 5% of sales) are flagged for review.<br><br>
  <a href="https://public.tableau.com/app/profile/jerred.lawson/viz/RetailStoreViz/DASHABCInventoryClassification" target="_blank">
    <img src="images/Pareto_Dist.png" width="60%">
  </a><br><br>

**Product Order Recommendations:**
 - Order quantities and projections based on historical sales velocity.<br><br>
   <img src="images/Report_Products_to_Order.png" width="60%"><br>
  
# View the Project
 - 📄 Full Project Write-Up (recommended): [link to your PDF]
 - 📓 Jupyter Notebook: [Jupyter Notebook (GitHub)](https://github.com/JerredLawson/Inventory-Order-Recommendation-Engine/blob/main/notebooks/inventory_order_engine.ipynb)
 - 📊 Data Sources: /data

# Tech Stack
 - Python (pandas, numpy)
 - SQL (BigQuery)
 - Excel (data modeling, validation)
 - Tableau (visualization)
