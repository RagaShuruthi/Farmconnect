# 🌿 FarmConnect — Hyperlocal 3km Farm-to-Consumer Marketplace

FarmConnect is a real-time, hyperlocal (3 km radius) direct marketplace connecting consumers directly with nearby urban and rural farmers. It features an **Agent-to-Agent (A2A)** request-response broker that automatically matches consumer product requests with nearby farmers' specialties and inventory.

---

## 🚀 Quick Start & Deployment

### Running with Docker (Recommended for integration)
```bash
docker compose up -d --build
```
Access the application at **`http://localhost:8080`**.

### Running Locally
Open `index.html` via any static HTTP server:
```bash
python -m http.server 8080
# OR
npx http-server -p 8080
```

---

## 📐 Architecture & Integration Overview

FarmConnect operates with a modular browser-based JS layer (`Store`, `Agent`, `Matching`, `Geo`, `Auth`, `NotificationSystem`). 

For external apps (such as AI voice engines, backend microservices, mobile apps, or diagnostic toolkits), FarmConnect exposes global JavaScript interfaces and standard JSON schemas (`FarmConnect:AgentRequest:v1` and `FarmConnect:AgentResponse:v1`).

```
┌────────────────────────┐      A2A Request Payload (JSON)       ┌────────────────────────┐
│   External App / Voice  ├────────────────────────────────────►│  FarmConnect A2A Broker│
│   Engine / Consumer UI │                                     │       (Agent.js)       │
└───────────┬────────────┘                                     └───────────┬────────────┘
            │                                                              │
            │  Matching Engine (Haversine 3km)                             │  Broadcasts to
            ▼                                                              ▼
┌────────────────────────┐                                     ┌────────────────────────┐
│  Nearby Farmers (3km)  │◄────────────────────────────────────┤ Farmer AI Agent Pool   │
└────────────────────────┘      A2A Response Payload (JSON)    └────────────────────────┘
```

---

## 📡 API & JavaScript Module Endpoints

### 1. Agent-to-Agent (A2A) Broker (`js/agent.js`)
The `Agent` module handles request broadcasting between consumer agents and farmer agents.

| Endpoint / Method | Description | Parameters | Returns |
| :--- | :--- | :--- | :--- |
| `Agent.dispatchRequest(order)` | Broadcasts an order request to all registered farmer agents within 3 km. | `order`: Order Object | `void` (logs payload & dispatches responses) |
| `Agent.buildRequestPayload(order)` | Constructs `FarmConnect:AgentRequest:v1` JSON payload. | `order`: Order Object | `AgentRequestJSON` |
| `Agent.buildResponsePayload(farmer, order, matched)` | Constructs `FarmConnect:AgentResponse:v1` JSON payload. | `farmer`, `order`, `matched` | `AgentResponseJSON` |
| `Agent.simulateFarmerAgentResponse(farmer, order)` | Evaluates if a farmer agent can supply requested produce based on specialties and inventory. | `farmer`, `order` | `AgentResponseJSON \| null` |

#### A2A Request Schema (`FarmConnect:AgentRequest:v1`)
```json
{
  "schema": "FarmConnect:AgentRequest:v1",
  "requestId": "ord_12345",
  "timestamp": 1770627600000,
  "sentAt": "2026-08-09T09:00:00.000Z",
  "product": "Tomato",
  "quantity": 5,
  "unit": "kg",
  "consumer": {
    "id": "cons_001",
    "name": "Ananya Sharma",
    "location": { "lat": 12.9716, "lng": 77.5946, "label": "Indiranagar" }
  },
  "broadcast": {
    "radiusKm": 3,
    "toAll": true,
    "note": "Request sent to all registered farmers within 3 km"
  }
}
```

#### A2A Response Schema (`FarmConnect:AgentResponse:v1`)
```json
{
  "schema": "FarmConnect:AgentResponse:v1",
  "responseId": "resp_98765",
  "requestId": "ord_12345",
  "timestamp": 1770627601000,
  "farmer": {
    "id": "farm_002",
    "name": "Ramesh Kumar",
    "farmType": "terrace",
    "specialties": ["Tomato", "Spinach", "Brinjal"],
    "location": { "lat": 12.9800, "lng": 77.6000, "label": "Ulsoor" }
  },
  "status": "AVAILABLE",
  "offer": {
    "product": "Tomato",
    "unit": "kg",
    "estimatedPrice": 25,
    "availableQty": 15,
    "autoCreated": false
  }
}
```

---

### 2. Data Store Endpoints (`js/store.js`)
Manages persistence for users, products, orders, and notifications.

| Endpoint / Method | Description | Parameters | Returns |
| :--- | :--- | :--- | :--- |
| `Store.getUsers()` | Retrieves all registered users and pre-seeded farmers. | None | `Array<User>` |
| `Store.getAllFarmers()` | Returns all users with `role: 'farmer'`. | None | `Array<Farmer>` |
| `Store.getUserById(id)` | Retrieves a specific user by ID. | `id`: String | `User \| null` |
| `Store.addUser(userData)` | Registers a new user or farmer. | `userData`: Object | `User` |
| `Store.getProducts()` | Returns all product listings. | None | `Array<Product>` |
| `Store.addProduct(productData)` | Creates a new product listing. | `productData`: Object | `Product` |
| `Store.getOrders()` | Returns all orders. | None | `Array<Order>` |
| `Store.addOrder(orderData)` | Creates a new customer request/order. | `orderData`: Object | `Order` |
| `Store.updateOrderStatus(id, status)`| Updates order status (`pending`, `matched`, `accepted`, `delivered`, `cancelled`). | `id`: String, `status`: String | `Order` |
| `Store.getCurrentUser()` | Gets active user session. | None | `User \| null` |
| `Store.setCurrentUser(user)` | Sets active user session. | `user`: Object | `void` |

---

### 3. Matching & Distance Engine (`js/matching.js` & `js/geo.js`)
Calculates distance and routes requests to farmers within 3 km.

| Endpoint / Method | Description | Parameters | Returns |
| :--- | :--- | :--- | :--- |
| `Geo.haversineDistance(loc1, loc2)` | Calculates straight-line distance in km between two GPS coordinates. | `{lat, lng}`, `{lat, lng}` | `Number` (km) |
| `Geo.formatDistance(distKm)` | Formats distance string (e.g. `850 m` or `2.4 km`). | `distKm`: Number | `String` |
| `MatchingEngine.findFarmersForRequest(productName, consumerLoc)` | Finds all farmers within 3 km who specialize in or stock the requested product. | `productName`: String, `consumerLoc`: Location Object | `{ withInventory, withoutInventory, all }` |

---

### 4. Authentication Endpoints (`js/auth.js`)

| Endpoint / Method | Description | Parameters | Returns |
| :--- | :--- | :--- | :--- |
| `Auth.login(phone, role)` | Log in existing user by phone and role. | `phone`: String, `role`: `'consumer' \| 'farmer'` | `{ success: Boolean, user: User, message: String }` |
| `Auth.register(userData)` | Register a new user or farmer. | `userData`: User Object | `{ success: Boolean, user: User, message: String }` |
| `Auth.logout()` | Clears active session. | None | `void` |

---

### 5. Notification System (`js/notifications.js`)

| Endpoint / Method | Description | Parameters | Returns |
| :--- | :--- | :--- | :--- |
| `NotificationSystem.notifyFarmer(farmerId, notif)` | Sends real-time notification to farmer. | `farmerId`: String, `notif`: Notification Object | `Notification` |
| `NotificationSystem.notifyConsumer(consumerId, notif)` | Sends real-time notification to consumer. | `consumerId`: String, `notif`: Notification Object | `Notification` |
| `NotificationSystem.getNotificationsForUser(userId)` | Fetches notification list for user. | `userId`: String | `Array<Notification>` |

---

## 🔗 Integration Guide for External Applications

To integrate FarmConnect with an external backend, mobile app, or AI service (e.g., Python Voice Engine / Disease Detector):

### Integration Option A: Standard JavaScript API Window Injection
If hosting within a web wrapper or single page app:
```javascript
// 1. Authenticate or retrieve consumer
const user = Store.getCurrentUser();

// 2. Create an order request programmatically
const newOrder = Store.addOrder({
  consumerId: user.id,
  consumerName: user.name,
  consumerLocation: user.location,
  productName: "Tomato",
  requestedQty: 3,
  unit: "kg",
  maxPrice: 30,
  status: "pending"
});

// 3. Dispatch via A2A Broker
Agent.dispatchRequest(newOrder);
```

### Integration Option B: REST API Wrapper / Node Express Server
To expose FarmConnect endpoints over HTTP REST for mobile/external apps, add an Express wrapper around the Store/Agent methods:

```javascript
// Server wrapper snippet (express)
app.post('/api/requests', (req, res) => {
  const { consumerId, productName, quantity, unit, location } = req.body;
  const order = Store.addOrder({
    consumerId,
    productName,
    requestedQty: quantity,
    unit,
    consumerLocation: location,
    status: 'pending'
  });
  
  const matches = MatchingEngine.findFarmersForRequest(productName, location);
  Agent.dispatchRequest(order);
  
  res.json({ success: true, orderId: order.id, matchedFarmers: matches.all.length });
});
```

---

## 📊 Data Models Reference

### User / Farmer Model
```typescript
interface User {
  id: string;
  name: string;
  phone: string;
  role: 'consumer' | 'farmer';
  location: {
    lat: number;
    lng: number;
    label: string;
  };
  farmType?: 'terrace' | 'backyard' | 'garden' | 'community_plot' | 'farm_plot' | 'orchard' | 'balcony';
  specialties?: string[];
  rating?: number;
  source?: 'json' | 'manual';
}
```

### Product Model
```typescript
interface Product {
  id: string;
  farmerId: string;
  farmerName: string;
  name: string;
  category: 'Vegetables' | 'Leafy Greens' | 'Fruits' | 'Spices' | 'Herbs' | 'Grains' | 'Flowers';
  price: number;
  unit: 'kg' | 'bunch' | 'piece';
  availableQty: number;
  emoji: string;
}
```

### Order Model
```typescript
interface Order {
  id: string;
  consumerId: string;
  consumerName: string;
  consumerLocation: { lat: number; lng: number; label: string };
  productName: string;
  requestedQty: number;
  unit: string;
  maxPrice?: number;
  status: 'pending' | 'matched' | 'accepted' | 'delivered' | 'cancelled';
  matchedFarmerIds: string[];
  createdAt: number;
}
```

---

## 🛠️ Tech Stack
- **Frontend**: Vanilla JavaScript (ES6+), Modern HTML5, Custom CSS3 Design System
- **Broker**: Agent-to-Agent (A2A) PubSub Request Broker (`Agent.js`)
- **Geolocation**: Haversine Spherical Distance Algorithm (`Geo.js`)
- **Server / Deployment**: Docker, Docker Compose, Nginx Alpine
