# Order & Payment Microservices

Two independent microservices built in Go using Clean Architecture.

---

## Architecture Overview

```
Client
  │
  ▼
Order Service (port 8080)
  │  POST /orders → saves order as "Pending", calls Payment Service
  │  GET  /orders/:id
  │  PATCH /orders/:id/cancel
  │
  ▼
Payment Service (port 8081)
  │  POST /payments → validates amount, returns Authorized/Declined
  │  GET  /payments/:order_id
  │
  ├── order_db (PostgreSQL)
  └── payment_db (PostgreSQL)
```

## Why Two Services?

Each service has its own database and runs independently. This follows the microservices principle: services are loosely coupled and can be deployed, scaled, or failed independently. The Order Service does not share models or a database with the Payment Service.

## Project Structure

```
order-service/
├── cmd/service/main.go         → entry point, wiring
├── domain/order.go             → Order entity
├── usecase/
│   ├── order_usecase.go        → business logic
│   ├── order_repository.go     → repository interface
│   └── payment_client.go       → HTTP client for Payment Service
├── repository/
│   ├── db.go                   → DB connection
│   └── order_repo.go           → PostgreSQL implementation
├── transport/http/
│   └── order_handler.go        → Gin HTTP handlers
└── migrations/
    └── 001_create_orders.sql

payment-service/
├── cmd/service/main.go
├── domain/payment.go
├── usecase/
│   ├── payment_usecase.go
│   └── payment_repository.go
├── repository/
│   ├── db.go
│   └── payment_repo.go
├── transport/http/
│   └── payment_handler.go
└── migrations/
    └── 001_create_payments.sql
```

## Clean Architecture Layers

| Layer       | Responsibility                          |
|-------------|----------------------------------------|
| domain      | Entities only — no logic               |
| usecase     | All business logic lives here          |
| repository  | Database access, implements interfaces |
| transport   | HTTP handlers — no business logic      |

**Rule:** handlers call usecase, usecase calls repository. No logic in handlers.

---

## How to Run

### 1. Create databases

```sql
CREATE DATABASE order_db;
CREATE DATABASE payment_db;
```

### 2. Run migrations

```bash
psql -d order_db -f order-service/migrations/001_create_orders.sql
psql -d payment_db -f payment-service/migrations/001_create_payments.sql
```

### 3. Start Payment Service

```bash
cd payment-service
go run cmd/service/main.go
# runs on :8081
```

### 4. Start Order Service

```bash
cd order-service
go run cmd/service/main.go
# runs on :8080
```

---

## API Endpoints

### Order Service

| Method | Path                   | Description                        |
|--------|------------------------|------------------------------------|
| POST   | /orders                | Create order, triggers payment     |
| GET    | /orders/:id            | Get order by ID                    |
| PATCH  | /orders/:id/cancel     | Cancel order (Pending only)        |

**POST /orders**
```json
{
  "customer_id": "cust-1",
  "item_name": "Laptop",
  "amount": 50000
}
```

Response:
```json
{
  "ID": "uuid",
  "CustomerID": "cust-1",
  "ItemName": "Laptop",
  "Amount": 50000,
  "Status": "Paid",
  "CreatedAt": "2024-01-01T00:00:00Z"
}
```

### Payment Service

| Method | Path                      | Description               |
|--------|---------------------------|---------------------------|
| POST   | /payments                 | Process payment           |
| GET    | /payments/:order_id       | Get payment by order ID   |

---

## Business Rules

- `amount` must be > 0 (validated in both services)
- `amount` uses `int64` — never float
- If `amount > 100000` → payment is **Declined** → order becomes **Failed**
- If `amount <= 100000` → payment is **Authorized** → order becomes **Paid**
- Only **Pending** orders can be cancelled. Paid or Failed orders cannot be cancelled.

---

## Failure Handling

### Payment Service Down / Timeout

- Order Service uses `http.Client` with a **2-second timeout**
- If Payment Service is unreachable or times out:
  - Order status is set to `"Failed"`
  - Order Service returns HTTP `503 Service Unavailable`
- The order is **always saved first** as `"Pending"` before calling Payment Service, ensuring no data is lost even if payment fails

### Payment Declined

- Order is saved, payment is saved with status `"Declined"`, order status updated to `"Failed"`
- HTTP `200` is returned with the order showing `Status: "Failed"`

---

## Bonus: Idempotency-Key

Pass `Idempotency-Key` header when calling `POST /payments` to prevent duplicate payments for the same order:

```
Idempotency-Key: unique-request-id-123
```

If a payment for the given `order_id` already exists and the key is provided, the existing payment is returned instead of creating a new one.

---

## Testing Scenarios

### 1. Normal flow (amount ≤ 100000)
```bash
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_id":"c1","item_name":"Book","amount":5000}'
# Expected: Status "Paid"
```

### 2. Payment declined (amount > 100000)
```bash
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_id":"c1","item_name":"Car","amount":200000}'
# Expected: Status "Failed"
```

### 3. Payment service OFF
```bash
# Stop payment-service, then:
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"customer_id":"c1","item_name":"Phone","amount":10000}'
# Expected: HTTP 503, order Status "Failed"
```

### 4. Cancel order
```bash
curl -X PATCH http://localhost:8080/orders/{id}/cancel
# Expected: Status "Cancelled" (only if was "Pending")
```

### 5. Cancel paid order
```bash
curl -X PATCH http://localhost:8080/orders/{paid-order-id}/cancel
# Expected: 400 "only pending orders can be cancelled"
```
