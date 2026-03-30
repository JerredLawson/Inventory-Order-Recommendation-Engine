WITH 
  
  /* Required user defined parameters. */
  params AS (
    SELECT
      DATE '2025-12-31' AS period_start,
      DATE '2026-02-03' AS period_end,
      DATE '2026-02-09' AS delivery_date,
      DATE '2026-02-23' AS next_delivery_date
  ),

  /* Adds reporting window and delivery timing metrics. */
  period_metrics AS (
    SELECT
      period_start,
      period_end,
      DATE_DIFF(period_end, period_start, DAY) + 1 AS period_days,
      delivery_date,
      next_delivery_date,
      DATE_DIFF(delivery_date, period_end, DAY) AS lead_time,
      DATE_DIFF(next_delivery_date, period_end, DAY) AS days_to_next_delivery,
      DATE_DIFF(next_delivery_date, delivery_date, DAY) AS days_between_deliveries
    FROM params
  ),

  /* Flags whether the reporting window is long enough for velocity-based planning. */
  period_metric_flags AS (
    SELECT
      period_days,
      CASE
        WHEN period_days < 14 
        THEN 'Window too short for planning (<14 days)'
        ELSE 'Valid planning window (>=14 days)'
      END AS planning_status
    FROM period_metrics
  ),

  /* beg_qty is the quantity on hand 
  for each product on the period_start date. */  
  beg_inventory AS (
    SELECT
      i.product_id,
      i.qty_on_hand AS beg_qty
    FROM `golden-passkey-334003.GiftShop.tblInventorySnapshot` i
    CROSS JOIN period_metrics pm
    WHERE i.snapshot_date = pm.period_start
  ),

  /* Aggregates all purchases received for each product
  during the reporting period. */
  purchases AS (
    SELECT
      p.product_id,
      SUM(p.total_qty_purchased) AS purchased
    FROM `golden-passkey-334003.GiftShop.tblPurchases` p
    CROSS JOIN period_metrics pm
    WHERE p.received_date BETWEEN pm.period_start AND pm.period_end
    GROUP BY p.product_id
  ),

  /* Aggregates all sales for each product 
  during the reporting period. */
  sales AS (
    SELECT
      s.product_id,
      SUM(s.quantity) AS sold
    FROM `golden-passkey-334003.GiftShop.tblSales` s
    CROSS JOIN period_metrics pm
    WHERE s.sale_date BETWEEN pm.period_start AND pm.period_end
    GROUP BY s.product_id
  ),

  /* Joins tblProduct with beg_inventory, purchases, and sales. */
  base AS (
    SELECT
      p.product_id,
      p.category,
      p.product_name,
      p.purchase_unit_size,
      COALESCE(b.beg_qty, 0) AS beg_qty,
      COALESCE(pr.purchased, 0) AS purchased,
      COALESCE(s.sold, 0) AS sold
    FROM `golden-passkey-334003.GiftShop.tblProduct` p
    LEFT JOIN beg_inventory b
      ON p.product_id = b.product_id
    LEFT JOIN purchases pr
      ON p.product_id = pr.product_id
    LEFT JOIN sales s
      ON p.product_id = s.product_id
  ),

  /* Extends base to include sales metrics. */
  sales_metrics AS (
    SELECT
      b.product_id,
      b.category,
      b.product_name,
      b.purchase_unit_size,
      b.beg_qty,
      b.purchased,
      b.sold,
      b.beg_qty + b.purchased - b.sold AS end_qty,
      
      b.sold / pm.period_days AS daily_veloc,
      (b.sold / pm.period_days) * 7 AS weekly_veloc

    FROM base b
    CROSS JOIN period_metrics pm
  ),

  current_days_of_inventory AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,

      CAST(
        FLOOR(
          SAFE_DIVIDE(
            end_qty,
            daily_veloc
          )
        ) AS INT64
      ) AS cur_days_of_inv

    FROM sales_metrics
  ),

  current_depletion_date_projection AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,

      DATE_ADD(pm.period_end, INTERVAL cdoi.cur_days_of_inv DAY) AS cur_dep_date

    FROM current_days_of_inventory cdoi
    CROSS JOIN period_metrics pm
  ),

  /* Calculates cumulative percent of sales for each product. */
  percent_of_sales AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,

      SAFE_DIVIDE(
        sold,
        SUM(sold) OVER ()
      ) AS pct_of_sales

    FROM current_depletion_date_projection 
  ),

  /* Calculates cumulative percent of sales used for Pareto analysis. */
  cumulative_percent_of_sales AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,
      pct_of_sales,

      SUM(pct_of_sales) OVER (ORDER BY sold DESC, product_id) AS cum_pct_of_sales

    FROM percent_of_sales
  ),

  /* Uses the Pareto distribution to assign each product an ABC classification. */
  abc_classification AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,
      pct_of_sales,
      cum_pct_of_sales,

      CASE
        WHEN cum_pct_of_sales <= 0.80 THEN 'A'
        WHEN cum_pct_of_sales <= 0.95 THEN 'B'
        ELSE 'C'
      END AS abc_class

    FROM cumulative_percent_of_sales
  ),

  /* Creates target days of inventory for each ABC classification. */
  inventory_targets AS (
    SELECT
      ac.product_id,
      ac.category,
      ac.product_name,
      ac.purchase_unit_size,
      ac.beg_qty,
      ac.purchased,
      ac.sold,
      ac.end_qty,
      ac.daily_veloc,
      ac.weekly_veloc,
      ac.cur_days_of_inv,
      ac.cur_dep_date,
      ac.pct_of_sales,
      ac.cum_pct_of_sales,
      ac.abc_class,

      CASE
        WHEN ac.abc_class = 'A' THEN 'A – Top Sellers: 2 weeks of extra inventory'
        WHEN ac.abc_class = 'B' THEN 'B – Mid Sellers: 1 week of extra inventory'
        WHEN ac.abc_class = 'C' THEN 'C – Low Sellers: no extra inventory'
      END AS abc_desc,

      CASE
        WHEN ac.abc_class = 'A' THEN pm.days_between_deliveries + 14
        WHEN ac.abc_class = 'B' THEN pm.days_between_deliveries + 7
        WHEN ac.abc_class = 'C' THEN pm.days_between_deliveries
      END AS targ_days_of_inv_after_deliv 

    FROM abc_classification ac
    CROSS JOIN period_metrics pm
  ),

  inventory_at_delivery AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,
      pct_of_sales,
      cum_pct_of_sales,
      abc_class,
      abc_desc,
      targ_days_of_inv_after_deliv,

      GREATEST(end_qty - CAST(CEIL(daily_veloc * pm.lead_time) AS INT64), 0) AS inv_at_deliv

    FROM inventory_targets
    CROSS JOIN period_metrics pm
  ),

  target_inventory_after_delivery AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,
      pct_of_sales,
      cum_pct_of_sales,
      abc_class,
      abc_desc,
      targ_days_of_inv_after_deliv,
      inv_at_deliv,

      CAST(CEIL(targ_days_of_inv_after_deliv * daily_veloc) AS INT64) AS targ_inv_after_deliv

    FROM inventory_at_delivery
  ),

  units_needed_to_order AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,
      pct_of_sales,
      cum_pct_of_sales,
      abc_class,
      abc_desc,
      targ_days_of_inv_after_deliv,
      inv_at_deliv,
      targ_inv_after_deliv,

      GREATEST(targ_inv_after_deliv - inv_at_deliv, 0) AS units_to_order 

    FROM target_inventory_after_delivery
  ),

  /* Converts target units of inventory into purchase units.
  For example, if a product is ordered in cases of 28 and the target is 30 units,
  then 2 cases must be ordered. */
  purchase_recommendations AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,
      pct_of_sales,
      cum_pct_of_sales,
      abc_class,
      abc_desc,
      targ_days_of_inv_after_deliv,
      inv_at_deliv,
      targ_inv_after_deliv,
      units_to_order,

      CASE
        WHEN purchase_unit_size IS NULL OR purchase_unit_size = 0 THEN NULL
        ELSE CAST(
          CEILING(
            SAFE_DIVIDE(
              GREATEST(units_to_order, 0),
              purchase_unit_size
            )
          ) AS INT64
        )
      END AS purch_units_to_order

    FROM units_needed_to_order
  ),

  /* Projects quantity on hand after inventory is delivered. */
  projected_inventory AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,
      pct_of_sales,
      cum_pct_of_sales,
      abc_class,
      abc_desc,
      targ_days_of_inv_after_deliv,
      inv_at_deliv,
      targ_inv_after_deliv,
      units_to_order,
      purch_units_to_order,
      
      inv_at_deliv + (purch_units_to_order * purchase_unit_size)
        AS proj_qty_on_hand_after_deliv

    FROM purchase_recommendations
  ),

  /* Converts projected quantity on hand into projected days of inventory. */
  depletion_projections AS (
    SELECT
      product_id,
      category,
      product_name,
      purchase_unit_size,
      beg_qty,
      purchased,
      sold,
      end_qty,
      daily_veloc,
      weekly_veloc,
      cur_days_of_inv,
      cur_dep_date,
      pct_of_sales,
      cum_pct_of_sales,
      abc_class,
      abc_desc,
      targ_days_of_inv_after_deliv,
      inv_at_deliv,
      targ_inv_after_deliv,
      units_to_order,
      purch_units_to_order,
      proj_qty_on_hand_after_deliv,

      CAST(
        FLOOR(
          SAFE_DIVIDE(
            proj_qty_on_hand_after_deliv,
            daily_veloc
          )
        ) AS INT64
      ) AS proj_days_until_dep_after_deliv 

    FROM projected_inventory
  ),

  /* Uses projected days of inventory to calculate the projected depletion date. */
  depletion_date_projection AS (
    SELECT
      dp.product_id,
      dp.category,
      dp.product_name,
      dp.purchase_unit_size,
      dp.beg_qty,
      dp.purchased,
      dp.sold,
      dp.end_qty,
      dp.daily_veloc,
      dp.weekly_veloc,
      dp.cur_days_of_inv,
      dp.cur_dep_date,
      dp.pct_of_sales,
      dp.cum_pct_of_sales,
      dp.abc_class,
      dp.abc_desc,
      dp.targ_days_of_inv_after_deliv,
      dp.inv_at_deliv,
      dp.targ_inv_after_deliv,
      dp.units_to_order,
      dp.purch_units_to_order,
      dp.proj_qty_on_hand_after_deliv,
      dp.proj_days_until_dep_after_deliv,

      CASE
        WHEN dp.purch_units_to_order = 0 THEN dp.cur_dep_date
        ELSE DATE_ADD(pm.delivery_date, INTERVAL dp.proj_days_until_dep_after_deliv DAY)
      END AS proj_dep_date

    FROM depletion_projections dp
    CROSS JOIN period_metrics pm
  )

/* Formats and returns final output values. */ 
SELECT
  product_id,
  category,
  product_name,
  purchase_unit_size,
  beg_qty,
  purchased,
  sold,
  end_qty,
  ROUND(daily_veloc, 2) AS daily_veloc, 
  ROUND(weekly_veloc, 2) AS weekly_veloc, 
  cur_days_of_inv,
  cur_dep_date,
  ROUND(pct_of_sales, 4) AS pct_of_sales,
  ROUND(cum_pct_of_sales, 4) AS cum_pct_of_sales,
  abc_class,
  abc_desc,
  targ_days_of_inv_after_deliv,
  inv_at_deliv,
  targ_inv_after_deliv,
  units_to_order,
  purch_units_to_order, 
  proj_qty_on_hand_after_deliv, 
  proj_days_until_dep_after_deliv, 
  proj_dep_date, 
  pmf.planning_status
FROM depletion_date_projection
CROSS JOIN period_metric_flags pmf
ORDER BY sold DESC, product_id;

