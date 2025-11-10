

# 📝 StockFlow B2B Inventory Management System Assessment

This document contains the solutions, database design, and API implementation for the three parts of the StockFlow take-home exercise, along with explicit reasoning and documented assumptions.

-----

## 🛠️ Part 1: Code Review & Debugging

The following analysis addresses the issues in the provided Python/Flask `create_product` endpoint.

### 🛑 Identified Issues and Impact

| Issue Type | Problem | Impact in Production |
| :--- | :--- | :--- |
| **Technical/Safety** | **Lack of Input Validation/Error Handling:** Code blindly accesses `data['key']`. | A missing required field (e.g., `'name'`) results in a **`KeyError`**, causing a fatal **$\text{500 Internal Server Error}$** instead of a clean $\text{400 Bad Request}$ response. |
| **Business Logic/Constraint** | **No SKU Uniqueness Check:** The code doesn't verify if the `sku` already exists. | If a unique constraint is set in the database, the transaction fails with an **Integrity Error**. If not, it creates **duplicate SKUs**, leading to corrupted inventory tracking. |
| **Business Logic/Atomicity** | **Two Separate `db.session.commit()` calls.** | If the first commit succeeds but the second fails (e.g., due to an invalid `warehouse_id`), the **Product is created** but the **Inventory record is missing**, resulting in an inconsistent data state. |
| **Business Logic/Data Integrity** | **Implicit $\text{1:1}$ Mapping:** `Product` is initialized with `warehouse_id`. | The product is incorrectly tied to a single warehouse, contradicting the requirement that **"Products can exist in multiple warehouses."** `warehouse_id` belongs only in the $\text{Inventory}$ table. |

### ✅ Corrected Endpoint Implementation (Python/Flask)

```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json
    required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']

    # Input Validation: Check for all required fields.
    if not all(field in data for field in required_fields):
        missing = [f for f in required_fields if f not in data]
        return {"error": f"Missing required fields: {', '.join(missing)}"}, 400

    sku = data['sku']
    
    # Business Logic Validation: Check for SKU Uniqueness.
    if Product.query.filter_by(sku=sku).first():
        return {"error": f"Product with SKU '{sku}' already exists."}, 409 # 409 Conflict

    # Use a single transaction for atomicity.
    try:
        # Create new product (warehouse_id is NOT included in Product)
        product = Product(
            name=data['name'],
            sku=sku,
            price=data['price'],
        )
        db.session.add(product)

        # Create initial Inventory count in the specified warehouse
        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )
        db.session.add(inventory)

        # Single Commit: Ensures Product and Inventory are created together successfully.
        db.session.commit()
        
        return {"message": "Product created successfully", "product_id": product.id}, 201

    except Exception as e:
        # Rollback on Failure: Prevents inconsistent data state.
        db.session.rollback()
        return {"error": "An unexpected error occurred during product creation."}, 500
```

-----

## 🏗️ Part 2: Database Design

The schema design adheres to the requirements for multi-warehouse tracking, supplier management, inventory history, and product bundles.

### 📊 Design Schema (SQL DDL Notation)

```sql
-- CORE BUSINESS ENTITIES
CREATE TABLE Companies (
    id INT PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE Warehouses (
    id INT PRIMARY KEY,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    FOREIGN KEY (company_id) REFERENCES Companies(id),
    UNIQUE (company_id, name)
);

---

-- PRODUCTS & SUPPLIERS
CREATE TABLE Suppliers (
    id INT PRIMARY KEY,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    contact_email VARCHAR(255),
    FOREIGN KEY (company_id) REFERENCES Companies(id)
);

CREATE TABLE Products (
    id INT PRIMARY KEY,
    company_id INT NOT NULL,
    name VARCHAR(255) NOT NULL,
    sku VARCHAR(100) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    product_type VARCHAR(50) NOT NULL, -- e.g., 'Standard', 'Bundle'
    low_stock_threshold INT DEFAULT 10,
    FOREIGN KEY (company_id) REFERENCES Companies(id),
    UNIQUE (sku) -- SKUs MUST be unique across the platform
);

-- M:N relationship: Tracks which supplier provides which product
CREATE TABLE Product_Suppliers (
    product_id INT NOT NULL,
    supplier_id INT NOT NULL,
    is_primary_supplier BOOLEAN DEFAULT FALSE, -- Flag for easy reordering logic
    PRIMARY KEY (product_id, supplier_id),
    FOREIGN KEY (product_id) REFERENCES Products(id),
    FOREIGN KEY (supplier_id) REFERENCES Suppliers(id)
);

---

-- INVENTORY & TRACKING
CREATE TABLE Inventory (
    id BIGINT PRIMARY KEY,
    product_id INT NOT NULL,
    warehouse_id INT NOT NULL,
    quantity INT NOT NULL,
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (product_id) REFERENCES Products(id),
    FOREIGN KEY (warehouse_id) REFERENCES Warehouses(id),
    UNIQUE (product_id, warehouse_id) -- Ensures a single stock record per product/warehouse
);

-- Historical/Audit Trail for inventory changes
CREATE TABLE Inventory_History (
    id BIGINT PRIMARY KEY,
    inventory_id BIGINT NOT NULL,
    transaction_type VARCHAR(50) NOT NULL, -- e.g., 'RECEIVE', 'SALE', 'ADJUSTMENT'
    quantity_change INT NOT NULL,
    new_quantity INT NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    reference_doc_id VARCHAR(100), -- Links to an Order ID or PO ID
    FOREIGN KEY (inventory_id) REFERENCES Inventory(id)
);

---

-- PRODUCT BUNDLES (Self-referencing structure)
CREATE TABLE Product_Bundles (
    bundle_product_id INT NOT NULL, 
    component_product_id INT NOT NULL, 
    quantity_required INT NOT NULL DEFAULT 1, 
    PRIMARY KEY (bundle_product_id, component_product_id),
    FOREIGN KEY (bundle_product_id) REFERENCES Products(id),
    FOREIGN KEY (component_product_id) REFERENCES Products(id)
);
```

### 💡 Justification of Decisions

  * **`Inventory` Table Constraint:** The use of **`UNIQUE (product_id, warehouse_id)`** is the fundamental design choice that satisfies the multi-warehouse requirement while ensuring data integrity.
  * **`Products.sku` Constraint:** The **`UNIQUE (sku)`** constraint is non-negotiable, directly enforcing the business rule for platform-wide SKU uniqueness.
  * **`Inventory_History`:** This table is essential for the "Track when inventory levels change" requirement. It serves as an immutable ledger, recording the **`transaction_type`** and **`quantity_change`** for auditing and calculating historical velocity.
  * **`Product_Suppliers` (`is_primary_supplier`):** The M:N table allows for sourcing flexibility, and the boolean flag simplifies the logic needed to quickly find the correct supplier for the low-stock alert.

### ❓ Identified Gaps and Missing Requirements

The following questions must be answered by the product team:

1.  **Low Stock Threshold Logic:** Does the **`low_stock_threshold`** need to be set **per warehouse** or should it be **dynamic** (e.g., calculated based on the last $7/30$ days of sales velocity, as assumed in Part 3)?
2.  **Order Management:** Are **Sales Orders** and **Purchase Orders** required? Without dedicated tables, accurate tracking of $\text{expected}$ stock and linking $\text{Inventory\_History}$ records to financial events is difficult.
3.  **Users and Permissions:** What is the structure for **Users, Roles, and Access Control**? Who can access which company's data and which warehouse's inventory?
4.  **Cost Tracking:** Do we need to track **Supplier Cost** or **Warehouse-Specific Pricing** (e.g., Average Weighted Cost)?

-----

## 💻 Part 3: API Implementation

The following Node.js/Express implementation provides the low-stock alert endpoint, incorporating complex business rules.

### ✍️ API Implementation: `GET /api/companies/{company_id}/alerts/low-stock`

#### Documented Assumptions

1.  **Database Schema:** The design from Part 2 is used.
2.  **Recent Sales Activity:** Defined as any `transaction_type = 'SALE'` record in $\text{Inventory\_History}$ within the **last 30 days**.
3.  **Days Until Stockout:** Calculated based on the simple average daily sales velocity over the $\text{30 days}$ window.
4.  **Primary Supplier:** Only the supplier marked `is_primary_supplier = TRUE` is included in the alert.

<!-- end list -->

```javascript
// --- ASSUMED DB INTERFACE AND FRAMEWORK SETUP ---
const express = require('express');
const app = express();
// Mock DB Interface for conceptual implementation (replace with actual ORM)
const db = {
    query: async (sql, params) => { 
        // Returning mock data for demonstration
        if (sql.includes("FROM Inventory_History")) {
             return [{ total_sales: 60 }]; // Mock 60 units sold in 30 days
        }
        return [
            { // Mock low stock item 
                product_id: 123, product_name: "Widget A", sku: "WID-001", threshold: 20,
                warehouse_id: 456, warehouse_name: "Main Warehouse", current_stock: 5,
                supplier_id: 789, supplier_name: "Supplier Corp", supplier_contact_email: "orders@supplier.com"
            }
        ];
    }
};

const RECENT_SALES_DAYS = 30; // Define time window for velocity calculation

// ---------------------------------------------------

app.get('/api/companies/:company_id/alerts/low-stock', async (req, res) => {
    const companyId = parseInt(req.params.company_id, 10);

    // 1. Handle Edge Case: Invalid or missing company ID
    if (isNaN(companyId)) {
        return res.status(400).json({ error: "Invalid Company ID provided." });
    }

    try {
        // --- STEP 1: Find all products/inventory below their threshold for the company ---
        const lowStockQuery = `
            SELECT
                i.product_id, p.name AS product_name, p.sku, p.low_stock_threshold AS threshold,
                i.warehouse_id, w.name AS warehouse_name, i.quantity AS current_stock,
                s.id AS supplier_id, s.name AS supplier_name, s.contact_email AS supplier_contact_email
            FROM Inventory i
            JOIN Products p ON i.product_id = p.id
            JOIN Warehouses w ON i.warehouse_id = w.id
            LEFT JOIN Product_Suppliers ps ON p.id = ps.product_id AND ps.is_primary_supplier = TRUE
            LEFT JOIN Suppliers s ON ps.supplier_id = s.id
            WHERE p.company_id = ? AND i.quantity < p.low_stock_threshold;
        `;
        
        const lowStockItems = await db.query(lowStockQuery, [companyId]);

        // Handle Edge Case: No products are currently below stock threshold
        if (lowStockItems.length === 0) {
            return res.json({ alerts: [], total_alerts: 0 });
        }

        const alerts = [];

        // --- STEP 2: Filter by Sales Activity and Calculate Stockout Days ---
        for (const item of lowStockItems) {
            const startDate = new Date();
            startDate.setDate(startDate.getDate() - RECENT_SALES_DAYS);

            // Calculate sales velocity (total units sold) for the specific product/warehouse
            const salesVelocityQuery = `
                SELECT SUM(ABS(quantity_change)) AS total_sales
                FROM Inventory_History
                WHERE inventory_id = (SELECT id FROM Inventory WHERE product_id = ? AND warehouse_id = ?)
                AND transaction_type = 'SALE'
                AND timestamp >= ?;
            `;
            
            const velocityResult = await db.query(salesVelocityQuery, [
                item.product_id, 
                item.warehouse_id, 
                startDate.toISOString()
            ]);
            
            const totalSales = velocityResult[0]?.total_sales || 0;
            
            // Business Rule: Only alert for products with recent sales activity (totalSales > 0)
            if (totalSales > 0) {
                const averageDailyVelocity = totalSales / RECENT_SALES_DAYS;
                
                // Days Until Stockout: Provides the urgency metric
                const daysUntilStockout = item.current_stock / averageDailyVelocity;

                alerts.push({
                    product_id: item.product_id, product_name: item.product_name, sku: item.sku,
                    warehouse_id: item.warehouse_id, warehouse_name: item.warehouse_name,
                    current_stock: item.current_stock, threshold: item.threshold,
                    days_until_stockout: Math.round(daysUntilStockout), 
                    supplier: {
                        id: item.supplier_id || null,
                        name: item.supplier_name || 'No Primary Supplier',
                        contact_email: item.supplier_contact_email || null
                    }
                });
            }
        }

        // --- STEP 3: Return the final response ---
        return res.json({
            alerts: alerts,
            total_alerts: alerts.length
        });

    } catch (error) {
        console.error("Error fetching low-stock alerts:", error);
        // Handle Edge Case: Database/Server error
        return res.status(500).json({ error: "Failed to retrieve low-stock alerts due to a server error." });
    }
});
```

### 💡 Explanation of Approach

1.  **Initial Filter (SQL):** The $\text{lowStockQuery}$ uses a single SQL query with $\text{LEFT JOIN}$s to select all items below their threshold, efficiently filtering by the $\text{company\_id}$ and retrieving all necessary related data ($\text{Warehouse}$ name, $\text{Supplier}$ info) for the final response structure.
2.  **Iterative Velocity Calculation:** Due to the dynamic nature of "recent sales activity," the logic iterates over the initial low-stock list. A separate $\text{salesVelocityQuery}$ is run for each unique $\text{product\_id}$/$\text{warehouse\_id}$ combination to accurately calculate the sales volume over the last 30 days using the $\text{Inventory\_History}$ table.
3.  **Urgency Metric:** The calculated **$\text{days\_until\_stockout}$** provides critical business context, helping the user prioritize reordering based on immediate need, not just static threshold comparison.
4.  **Error Handling:** The code includes checks for invalid input ($\text{400}$), handles the empty result set, and wraps the main logic in a `try/catch` block to ensure a controlled failure ($\text{500}$ response) if a database or server error occurs.