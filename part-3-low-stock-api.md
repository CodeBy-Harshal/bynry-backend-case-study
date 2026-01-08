# Part 3: Low-Stock Alerts API

## Endpoint

GET /api/companies/{company_id}/alerts/low-stock


## Objective

Implement an API endpoint that returns low-stock alerts for a company by:
- Checking inventory across multiple warehouses
- Applying product-type–specific low-stock thresholds
- Considering only products with recent sales activity
- Including supplier information for reordering

## Assumptions

Due to incomplete requirements, the following assumptions are made:

1. SKU is globally unique.
2. Inventory is tracked per product per warehouse.
3. Recent sales are defined as sales in the last 30 days.
4. Low-stock threshold is defined per product type.
5. Each product has one primary supplier.
6. Daily sales velocity = total sales in last 30 days / 30.
7. Products with no recent sales are ignored.
8. `days_until_stockout` is calculated only when sales velocity > 0.


## Expected Response Format

    {
      "alerts": [
        {
          "product_id": 123,
          "product_name": "Widget A",
          "sku": "WID-001",
          "warehouse_id": 456,
          "warehouse_name": "Main Warehouse",
          "current_stock": 5,
          "threshold": 20,
          "days_until_stockout": 12,
          "supplier": {
            "id": 789,
            "name": "Supplier Corp",
            "contact_email": "orders@supplier.com"
          }
        }
      ],
      "total_alerts": 1
    }
    
## Implementation (Node.js / Express)

    // GET /api/companies/:companyId/alerts/low-stock
    
    router.get('/api/companies/:companyId/alerts/low-stock', async (req, res) => {
      const { companyId } = req.params;
      const alerts = [];
    
      // Fetch all warehouses for the company
      const warehouses = await Warehouse.findAll({
        where: { company_id: companyId }
      });
    
      for (const warehouse of warehouses) {
    
        // Fetch inventory for each warehouse
        const inventories = await Inventory.findAll({
          where: { warehouse_id: warehouse.id },
          include: [Product]
        });
    
        for (const inventory of inventories) {
          const product = inventory.Product;
    
          // Fetch recent sales (last 30 days)
          const recentSales = await Sales.sum('quantity', {
            where: {
              product_id: product.id,
              warehouse_id: warehouse.id,
              sold_at: {
                [Op.gte]: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
              }
            }
          }) || 0;
    
          // Ignore products with no recent sales
          if (recentSales === 0) continue;
    
          // Fetch low-stock threshold based on product type
          const thresholdConfig = await ProductThreshold.findOne({
            where: { product_type: product.product_type }
          });
    
          const threshold = thresholdConfig?.threshold || 0;
    
          // Check low-stock condition
          if (inventory.quantity < threshold) {
            const dailyVelocity = recentSales / 30;
    
            const daysUntilStockout =
              dailyVelocity > 0
                ? Math.floor(inventory.quantity / dailyVelocity)
                : null;
    
            // Fetch supplier details
            const supplierLink = await ProductSupplier.findOne({
              where: { product_id: product.id },
              include: [Supplier]
            });
    
            const supplier = supplierLink?.Supplier;
    
            alerts.push({
              product_id: product.id,
              product_name: product.name,
              sku: product.sku,
              warehouse_id: warehouse.id,
              warehouse_name: warehouse.name,
              current_stock: inventory.quantity,
              threshold: threshold,
              days_until_stockout: daysUntilStockout,
              supplier: supplier
                ? {
                    id: supplier.id,
                    name: supplier.name,
                    contact_email: supplier.contact_email
                  }
                : null
            });
          }
        }
      }
    
      return res.status(200).json({
        alerts,
        total_alerts: alerts.length
      });
    });
    
## Edge Cases Handled
  1. Company with no warehouses → empty alerts list
  2. Products with no recent sales → ignored
  3. Zero sales velocity → avoids division by zero
  4. Missing supplier → supplier returned as null
  5. Multiple warehouses → inventory evaluated per warehouse
