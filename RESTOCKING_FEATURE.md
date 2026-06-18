# Restocking Feature - Implementation Summary

## Completed Work

### Backend Changes ✅

#### 1. `server/data/demand_forecasts.json`
- Added `unit_cost` field to all 9 demand forecast items
- Prices set based on product complexity:
  - WDG-001: $45.00 (gap 150 → $6,750)
  - BRG-102: $125.00 (gap 2 → $250)
  - GSK-203: $28.50 (gap 100 → $2,850)
  - MTR-304: $850.00 (decreasing, gap negative — excluded from recommendations)
  - FLT-405: $12.75 (gap 150 → $1,912)
  - VLV-506: $340.00 (gap 1 → $340)
  - PSU-501: $18.99 (gap 2 → $38)
  - SNR-420: $89.00 (gap 2 → $178)
  - CTL-330: $215.00 (gap 1 → $215)

#### 2. `server/mock_data.py`
- Added `restocking_orders = []` — in-memory list for submitted restocking orders

#### 3. `server/main.py`
- Updated `DemandForecast` model: added `unit_cost: float` field
- Added new Pydantic models:
  - `RestockingOrderItem`: { sku, name, quantity, unit_cost, line_total }
  - `RestockingOrder`: { id, order_number, items, total_cost, budget, submitted_date, expected_delivery, status }
  - `CreateRestockingOrderRequest`: { items, total_cost, budget }
- Added imports: datetime, timedelta, uuid
- Implemented two API endpoints:
  - `POST /api/restocking-orders`: Creates new restocking order, assigns UUID id, generates order_number like "RST-2026-NNNN", sets expected_delivery 14 days out, returns order with status "Processing"
  - `GET /api/restocking-orders`: Returns all submitted restocking orders from in-memory list

### Frontend Changes ✅

#### 1. `client/src/api.js`
- Added `createRestockingOrder(restockingData)`: POST to `/api/restocking-orders`
- Added `getRestockingOrders()`: GET from `/api/restocking-orders`

#### 2. `client/src/views/Restocking.vue` (NEW FILE)
- Full Vue 3 Composition API component with:
  - **Budget slider**: Range $0–$50,000 (step $500), default $10,000, formatted currency display
  - **Recommendations table**: Shows all demand forecast items with positive gap (forecasted > current)
    - Columns: Item Name, SKU, Demand Gap, Unit Cost, Restock Qty, Line Total
    - Checkbox for each item (auto-selected greedily by budget)
    - Sorted by demand gap descending
  - **Summary section**: Total cost, item count, remaining budget
  - **Place Order button**: Submits order to backend, shows success message, resets form
  - Auto-select logic: when budget changes, greedily checks items from top until budget would exceed
  - Manual toggle: users can manually check/uncheck items
  - Loading/error states

#### 3. `client/src/views/Orders.vue` (UPDATED)
- Added new "Submitted Restocking Orders" section above the existing orders table
- Only shows when `restockingOrders.length > 0`
- Table columns: Order #, Submitted Date, Items Count, Total Cost, Expected Delivery, Lead Time, Status badge
- Expandable items detail using same `<details>` pattern as regular orders
- `loadOrders()` now fetches both regular orders and restocking orders in parallel with Promise.all
- Status badge coloring: Processing → warning (yellow), Delivered → success (green)

#### 4. `client/src/App.vue` (UPDATED)
- Added new `<router-link to="/restocking">Restocking</router-link>` in `.nav-tabs`
- Placed between "Demand Forecast" and "Reports" tabs

#### 5. `client/src/main.js` (UPDATED)
- Imported `Restocking` view component
- Added route: `{ path: '/restocking', component: Restocking }`

## Technical Details

### Budget Range Decision
- Based on analysis of all positive-gap items: ~$12,583 total cost
- Set slider to $0–$50,000 with $500 step
- Default budget: $10,000

### Auto-Selection Algorithm
- Fetch all demand forecasts on mount
- Filter to items where `forecasted_demand > current_demand`
- Sort by gap descending
- On budget change, iterate through sorted list and check items greedily until next item would exceed budget
- Allow manual override (users can uncheck selected or check excluded items)

### Order Submission Flow
1. User selects items and clicks "Place Order"
2. Frontend calls `createRestockingOrder` API with selected items and total cost
3. Backend generates:
   - UUID id
   - Order number (RST-2026-NNNN format)
   - Submitted date (now)
   - Expected delivery (now + 14 days)
   - Status: "Processing"
4. Order appended to in-memory `restocking_orders` list
5. Frontend shows success message, resets budget and selection
6. New order appears in Orders tab under "Submitted Restocking Orders" section

### Data Persistence
- Restocking orders persist in-memory during server runtime
- Will reset when backend server restarts (expected for demo)
- Can be upgraded to database storage for production

## How to Test

1. **Start backend**:
   ```bash
   cd server
   uv run python main.py
   ```

2. **Start frontend** (in new terminal):
   ```bash
   cd client
   npm run dev
   ```

3. **Navigate to Restocking tab**:
   - Click "Restocking" in the nav bar at `http://localhost:3000/restocking`

4. **Test the feature**:
   - Adjust budget slider — items should be auto-selected/deselected
   - Manually check/uncheck items
   - Click "Place Order"
   - See success message
   - Navigate to Orders tab
   - Verify "Submitted Restocking Orders" section appears with your order
   - Check expected delivery is ~14 days from now

5. **Test API directly**:
   ```bash
   # Get all demand forecasts (with new unit_cost field)
   curl http://localhost:8001/api/demand | jq

   # Get restocking orders (empty initially)
   curl http://localhost:8001/api/restocking-orders | jq

   # Submit a restocking order
   curl -X POST http://localhost:8001/api/restocking-orders \
     -H "Content-Type: application/json" \
     -d '{
       "items": [
         {"sku": "WDG-001", "name": "Industrial Widget Type A", "quantity": 150, "unit_cost": 45.00, "line_total": 6750.00}
       ],
       "total_cost": 6750.00,
       "budget": 10000.00
     }' | jq
   ```

## Files Modified

- ✅ `server/data/demand_forecasts.json` — added unit_cost
- ✅ `server/mock_data.py` — added restocking_orders list
- ✅ `server/main.py` — added models and endpoints
- ✅ `client/src/api.js` — added methods
- ✅ `client/src/views/Restocking.vue` — new file
- ✅ `client/src/views/Orders.vue` — updated with Submitted Orders section
- ✅ `client/src/App.vue` — added navigation link
- ✅ `client/src/main.js` — added import and route

## Next Steps (Optional)

For production deployment:
1. Add database table for restocking orders (with cascade delete)
2. Add user authentication/authorization
3. Implement audit logging for order submissions
4. Add email notifications on order submission
5. Add filtering to Submitted Orders by date range
6. Persist order history beyond server restart
