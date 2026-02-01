# API Documentation - GPS & Shipping Integration

## Overview
This document describes the order API endpoints for integrating GPS location and shipping data with your backend server.

---

## POST /api/orders

**Purpose:** Create a new order with GPS location and shipping address

### Request Headers
```
POST /api/orders HTTP/1.1
Content-Type: application/json
Authorization: Bearer {AUTH_TOKEN}  (optional, if user is logged in)
```

### Request Body
```json
{
  "product": "bag",
  "amount": 180.50,
  "name": "John Doe",
  "email": "john.doe@example.com",
  "shippingAddress": "123 Main Street, Apartment 4B, New York, NY 10001",
  "location": {
    "lat": 40.712776,
    "lng": -74.005974,
    "accuracy": 25
  },
  "timestamp": "2026-02-01T14:30:00.000Z"
}
```

### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product` | string | Yes | Product identifier/key (e.g., 'bag', 'watch') |
| `amount` | number | Yes | Order amount in USD (with decimals) |
| `name` | string | Yes | Customer billing name |
| `email` | string | No | Customer email address |
| `shippingAddress` | string | Yes | Full shipping address |
| `location.lat` | number | No | Latitude coordinate |
| `location.lng` | number | No | Longitude coordinate |
| `location.accuracy` | number | No | GPS accuracy in meters |
| `timestamp` | string | No | ISO 8601 timestamp |

### Response (Success)
```json
{
  "success": true,
  "order_id": "ORD-2026-001234",
  "message": "Order placed successfully",
  "status": "pending",
  "estimatedDelivery": "2-4 business days"
}
```

### Response (Error)
```json
{
  "success": false,
  "error": "Invalid shipping address",
  "code": "INVALID_ADDRESS",
  "status": 400
}
```

### HTTP Status Codes
```
200 OK              - Order created successfully
400 Bad Request     - Missing or invalid fields
401 Unauthorized    - Auth token invalid
403 Forbidden       - User not allowed to place order
409 Conflict        - Duplicate order detected
500 Server Error    - Internal server error
```

---

## GET /api/orders

**Purpose:** Retrieve orders (admin only)

### Request Headers
```
GET /api/orders HTTP/1.1
Authorization: Bearer {ADMIN_TOKEN}
```

### Query Parameters
```
?limit=10           - Number of orders to return (default: 50)
?offset=0           - Pagination offset (default: 0)
?sort=-timestamp    - Sort field (- for descending)
?status=pending     - Filter by status: pending, shipped, delivered
```

### Response
```json
{
  "success": true,
  "count": 10,
  "total": 245,
  "orders": [
    {
      "order_id": "ORD-2026-001234",
      "product": "bag",
      "amount": 180.50,
      "name": "John Doe",
      "email": "john@example.com",
      "shippingAddress": "123 Main St, New York, NY 10001",
      "location": {
        "lat": 40.712776,
        "lng": -74.005974,
        "accuracy": 25
      },
      "timestamp": "2026-02-01T14:30:00.000Z",
      "status": "pending"
    }
  ]
}
```

---

## POST /api/login

**Purpose:** Authenticate user and get auth token

### Request
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

### Response (Success)
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "email": "user@example.com",
    "name": "John Doe"
  }
}
```

### Response (Error)
```json
{
  "success": false,
  "error": "Invalid credentials",
  "status": 401
}
```

---

## POST /api/register

**Purpose:** Create new user account

### Request
```json
{
  "email": "newuser@example.com",
  "password": "secure_password",
  "name": "Jane Doe"
}
```

### Response (Success)
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 2,
    "email": "newuser@example.com",
    "name": "Jane Doe"
  }
}
```

### Response (Error)
```json
{
  "success": false,
  "error": "Email already exists",
  "code": "EMAIL_EXISTS",
  "status": 409
}
```

---

## Database Schema (Reference)

### Orders Table
```sql
CREATE TABLE orders (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  order_id TEXT UNIQUE NOT NULL,
  product_key TEXT NOT NULL,
  amount DECIMAL(10, 2) NOT NULL,
  buyer_name TEXT NOT NULL,
  buyer_email TEXT,
  shipping_address TEXT NOT NULL,
  
  -- GPS Location
  location_lat DECIMAL(10, 8),
  location_lng DECIMAL(10, 8),
  location_accuracy INTEGER,
  
  -- Status & Timestamps
  status TEXT DEFAULT 'pending',  -- pending, shipped, delivered, cancelled
  order_timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  
  -- Shipping
  tracking_number TEXT,
  shipped_at DATETIME,
  delivered_at DATETIME,
  
  FOREIGN KEY (buyer_id) REFERENCES users(id),
  FOREIGN KEY (product_key) REFERENCES products(key)
);
```

### Indexes for Performance
```sql
CREATE INDEX idx_orders_timestamp ON orders(order_timestamp DESC);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_location ON orders(location_lat, location_lng);
CREATE INDEX idx_orders_buyer ON orders(buyer_email);
```

---

## Implementation Examples

### Python (Flask)
```python
from flask import request, jsonify
from datetime import datetime
import json

@app.route('/api/orders', methods=['POST'])
def create_order():
    data = request.get_json()
    
    # Validate required fields
    if not data.get('shippingAddress'):
        return jsonify({'error': 'Shipping address required'}), 400
    
    # Validate GPS coordinates if provided
    location = data.get('location')
    if location:
        if not (-90 <= location.get('lat', 100) <= 90):
            return jsonify({'error': 'Invalid latitude'}), 400
        if not (-180 <= location.get('lng', 200) <= 180):
            return jsonify({'error': 'Invalid longitude'}), 400
    
    # Create order
    try:
        order = {
            'order_id': generate_order_id(),
            'product': data['product'],
            'amount': data['amount'],
            'name': data['name'],
            'email': data.get('email'),
            'shippingAddress': data['shippingAddress'],
            'location': location,
            'status': 'pending',
            'timestamp': datetime.utcnow().isoformat() + 'Z'
        }
        
        # Save to database
        db.orders.insert_one(order)
        
        return jsonify({
            'success': True,
            'order_id': order['order_id'],
            'message': 'Order placed successfully'
        }), 200
        
    except Exception as e:
        return jsonify({'error': str(e)}), 500
```

### Node.js (Express)
```javascript
app.post('/api/orders', (req, res) => {
  const { product, amount, name, email, shippingAddress, location } = req.body;
  
  // Validate
  if (!shippingAddress) {
    return res.status(400).json({ error: 'Shipping address required' });
  }
  
  // Validate GPS
  if (location) {
    if (location.lat < -90 || location.lat > 90) {
      return res.status(400).json({ error: 'Invalid latitude' });
    }
    if (location.lng < -180 || location.lng > 180) {
      return res.status(400).json({ error: 'Invalid longitude' });
    }
  }
  
  // Create and save
  const order = {
    order_id: generateOrderId(),
    product,
    amount,
    name,
    email,
    shippingAddress,
    location,
    status: 'pending',
    timestamp: new Date().toISOString()
  };
  
  db.collection('orders').insertOne(order, (err, result) => {
    if (err) {
      return res.status(500).json({ error: err.message });
    }
    res.json({
      success: true,
      order_id: order.order_id,
      message: 'Order placed successfully'
    });
  });
});
```

---

## Error Handling

### Common Errors

```json
{
  "code": "INVALID_ADDRESS",
  "message": "Shipping address must be at least 10 characters",
  "status": 400
}
```

```json
{
  "code": "INVALID_GPS",
  "message": "GPS coordinates out of valid range",
  "status": 400
}
```

```json
{
  "code": "INVALID_AMOUNT",
  "message": "Amount must be greater than 0",
  "status": 400
}
```

---

## Rate Limiting

Recommended rate limits:

```
Customers:     10 requests per minute per IP
Admin:         100 requests per minute per token
Login:         5 attempts per 15 minutes per email
```

---

## Security Best Practices

### 1. Validate All Inputs
```javascript
// Always validate before processing
if (typeof amount !== 'number' || amount <= 0) {
  throw new Error('Invalid amount');
}
```

### 2. Sanitize GPS Data
```javascript
// Validate coordinate ranges
const isValidCoordinate = (lat, lng) => {
  return lat >= -90 && lat <= 90 && lng >= -180 && lng <= 180;
};
```

### 3. Verify Shipping Addresses
```javascript
// Consider using USPS/UPS API for verification
const verifyAddress = async (address) => {
  // Validate format and check with postal service
  return await postalService.validate(address);
};
```

### 4. Encrypt Location Data
```javascript
// Encrypt GPS coordinates in database
const encryptLocation = (location) => {
  return crypto.encrypt(JSON.stringify(location), key);
};
```

### 5. Log All Orders
```python
import logging
logging.info(f"Order created: {order_id} | Location: {location}")
```

---

## Geofencing Integration (Optional)

Use GPS data to define delivery zones:

```python
def calculate_distance(lat1, lng1, lat2, lng2):
    """Calculate distance between two GPS points (Haversine formula)"""
    from math import radians, sin, cos, sqrt, atan2
    
    R = 3959  # Earth's radius in miles
    
    lat1, lng1, lat2, lng2 = map(radians, [lat1, lng1, lat2, lng2])
    dlat = lat2 - lat1
    dlng = lng2 - lng1
    
    a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlng/2)**2
    c = 2 * atan2(sqrt(a), sqrt(1-a))
    
    return R * c  # Distance in miles

def get_shipping_cost(customer_lat, customer_lng, warehouse_lat, warehouse_lng):
    """Calculate shipping cost based on distance"""
    distance = calculate_distance(customer_lat, customer_lng, warehouse_lat, warehouse_lng)
    
    if distance <= 5:
        return 5.00
    elif distance <= 15:
        return 9.99
    else:
        return 14.99
```

---

## Testing

### cURL Examples

**Create Order:**
```bash
curl -X POST http://localhost:8000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "product": "bag",
    "amount": 180.50,
    "name": "John Doe",
    "email": "john@example.com",
    "shippingAddress": "123 Main St, New York, NY 10001",
    "location": {
      "lat": 40.712776,
      "lng": -74.005974,
      "accuracy": 25
    }
  }'
```

**Get Orders:**
```bash
curl -X GET "http://localhost:8000/api/orders?limit=10" \
  -H "Authorization: Bearer YOUR_TOKEN"
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-01 | Initial release with GPS and shipping |

---

**Last Updated:** February 1, 2026  
**Status:** ✅ Production Ready
