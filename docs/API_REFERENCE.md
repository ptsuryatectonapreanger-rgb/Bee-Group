# Bee Group API Reference

Dokumentasi lengkap API endpoints untuk Bee Group Platform.

## Base URL
```
Development: http://localhost:3000
Production: https://api.beegroup.com
```

## Authentication

Semua requests (kecuali login & register) harus include header:
```
Authorization: Bearer {token}
```

---

## 🔐 Authentication Endpoints

### Register User
**POST** `/api/auth/register`

Request body:
```json
{
  "email": "user@example.com",
  "username": "username",
  "password": "password123",
  "firstName": "John",
  "lastName": "Doe",
  "phone": "08123456789"
}
```

Response (201):
```json
{
  "id": "user-123",
  "email": "user@example.com",
  "username": "username",
  "firstName": "John",
  "lastName": "Doe",
  "role": "USER",
  "status": "ACTIVE",
  "createdAt": "2024-06-25T10:00:00Z"
}
```

### Login
**POST** `/api/auth/login`

Request body:
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

Response (200):
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "user-123",
    "email": "user@example.com",
    "username": "username",
    "role": "USER",
    "status": "ACTIVE"
  }
}
```

### Logout
**POST** `/api/auth/logout`

Headers:
```
Authorization: Bearer {token}
```

Response (200):
```json
{
  "message": "Logout successful"
}
```

### Refresh Token
**POST** `/api/auth/refresh`

Request body:
```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

Response (200):
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIs..."
}
```

---

## 🏢 Companies Endpoints

### Get All Companies
**GET** `/api/companies`

Query parameters:
- `page` (number, default: 1) - Halaman
- `limit` (number, default: 10) - Items per halaman
- `status` (string) - Filter by status (ACTIVE, INACTIVE, SUSPENDED)
- `type` (string) - Filter by type (HOLDING, SUBSIDIARY, INVESTMENT, etc.)
- `search` (string) - Search by name or code

Response (200):
```json
{
  "data": [
    {
      "id": "company-123",
      "name": "Bee Group Main",
      "code": "BG-MAIN",
      "type": "HOLDING",
      "status": "ACTIVE",
      "email": "info@beegroup.com",
      "phone": "+62213456789",
      "address": "Jl. Jenderal Sudirman No. 123",
      "city": "Jakarta",
      "province": "DKI Jakarta",
      "zipCode": "12910",
      "country": "Indonesia",
      "revenue": 1000000000000,
      "employees": 500,
      "foundedAt": "2020-01-15T00:00:00Z",
      "createdAt": "2024-06-25T10:00:00Z",
      "updatedAt": "2024-06-25T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 15,
    "pages": 2
  }
}
```

### Create Company
**POST** `/api/companies`

Headers:
```
Authorization: Bearer {token}
Content-Type: application/json
```

Request body:
```json
{
  "name": "PT Bee Indonesia",
  "code": "BG-ID",
  "type": "SUBSIDIARY",
  "email": "info@beeindonesia.com",
  "phone": "+62213456789",
  "address": "Jl. Gatot Subroto No. 456",
  "city": "Jakarta",
  "province": "DKI Jakarta",
  "zipCode": "12950",
  "country": "Indonesia",
  "description": "Subsidiary company in Indonesia",
  "revenue": 500000000000,
  "employees": 250,
  "foundedAt": "2021-03-20T00:00:00Z",
  "parentId": "company-123"
}
```

Response (201):
```json
{
  "id": "company-456",
  "name": "PT Bee Indonesia",
  "code": "BG-ID",
  "type": "SUBSIDIARY",
  "status": "ACTIVE",
  "email": "info@beeindonesia.com",
  "phone": "+62213456789",
  "address": "Jl. Gatot Subroto No. 456",
  "city": "Jakarta",
  "province": "DKI Jakarta",
  "zipCode": "12950",
  "country": "Indonesia",
  "description": "Subsidiary company in Indonesia",
  "revenue": 500000000000,
  "employees": 250,
  "foundedAt": "2021-03-20T00:00:00Z",
  "parentId": "company-123",
  "createdAt": "2024-06-25T10:00:00Z",
  "updatedAt": "2024-06-25T10:00:00Z"
}
```

### Get Company Detail
**GET** `/api/companies/:id`

Headers:
```
Authorization: Bearer {token}
```

Response (200):
```json
{
  "id": "company-123",
  "name": "Bee Group Main",
  "code": "BG-MAIN",
  "type": "HOLDING",
  "status": "ACTIVE",
  "email": "info@beegroup.com",
  "phone": "+62213456789",
  "address": "Jl. Jenderal Sudirman No. 123",
  "city": "Jakarta",
  "province": "DKI Jakarta",
  "zipCode": "12910",
  "country": "Indonesia",
  "revenue": 1000000000000,
  "employees": 500,
  "foundedAt": "2020-01-15T00:00:00Z",
  "subsidiaries": [
    {
      "id": "company-456",
      "name": "PT Bee Indonesia",
      "code": "BG-ID",
      "type": "SUBSIDIARY",
      "status": "ACTIVE"
    }
  ],
  "portfolioItems": [
    {
      "id": "portfolio-123",
      "ownershipPercentage": 85.5,
      "acquisitionPrice": 500000000000,
      "currentValue": 650000000000,
      "status": "ACTIVE"
    }
  ],
  "createdAt": "2024-06-25T10:00:00Z",
  "updatedAt": "2024-06-25T10:00:00Z"
}
```

### Update Company
**PUT** `/api/companies/:id`

Headers:
```
Authorization: Bearer {token}
Content-Type: application/json
```

Request body:
```json
{
  "name": "Bee Group Updated",
  "email": "newemail@beegroup.com",
  "phone": "+62219876543",
  "revenue": 1200000000000,
  "employees": 550
}
```

Response (200):
```json
{
  "id": "company-123",
  "name": "Bee Group Updated",
  "code": "BG-MAIN",
  "type": "HOLDING",
  "status": "ACTIVE",
  "email": "newemail@beegroup.com",
  "phone": "+62219876543",
  "revenue": 1200000000000,
  "employees": 550,
  "updatedAt": "2024-06-25T12:00:00Z"
}
```

### Delete Company
**DELETE** `/api/companies/:id`

Headers:
```
Authorization: Bearer {token}
```

Response (200):
```json
{
  "message": "Company deleted successfully"
}
```

---

## 💼 Portfolio Endpoints

### Get Portfolio Summary
**GET** `/api/portfolio`

Headers:
```
Authorization: Bearer {token}
```

Query parameters:
- `companyId` (string, optional) - Filter by specific company
- `status` (string, optional) - ACTIVE, INACTIVE, SUSPENDED

Response (200):
```json
{
  "totalPortfolioValue": 5000000000000,
  "totalInvestment": 3500000000000,
  "totalGain": 1500000000000,
  "gainPercentage": 42.86,
  "numberOfHoldings": 15,
  "activeHoldings": 14,
  "items": [
    {
      "id": "portfolio-123",
      "companyId": "company-456",
      "companyName": "PT Bee Indonesia",
      "ownershipPercentage": 85.5,
      "acquisitionDate": "2021-03-20T00:00:00Z",
      "acquisitionPrice": 500000000000,
      "currentValue": 650000000000,
      "gain": 150000000000,
      "gainPercentage": 30.0,
      "status": "ACTIVE"
    }
  ]
}
```

### Get Portfolio Performance
**GET** `/api/portfolio/performance`

Headers:
```
Authorization: Bearer {token}
```

Query parameters:
- `period` (string, default: '1Y') - 1W, 1M, 3M, 6M, 1Y, ALL
- `companyId` (string, optional)

Response (200):
```json
{
  "period": "1Y",
  "data": [
    {
      "month": "January 2024",
      "value": 4500000000000,
      "gain": 100000000000,
      "gainPercentage": 2.27
    },
    {
      "month": "February 2024",
      "value": 4650000000000,
      "gain": 150000000000,
      "gainPercentage": 3.33
    }
  ],
  "summary": {
    "startValue": 4500000000000,
    "endValue": 5000000000000,
    "totalGain": 500000000000,
    "gainPercentage": 11.11,
    "bestMonth": "February 2024",
    "worstMonth": "March 2024"
  }
}
```

---

## 💰 Financial Records Endpoints

### Get Financial Records
**GET** `/api/financials`

Headers:
```
Authorization: Bearer {token}
```

Query parameters:
- `companyId` (string, required) - Company ID
- `type` (string, optional) - REVENUE, EXPENSE, ASSET, LIABILITY, EQUITY
- `startDate` (string, ISO) - Filter by start date
- `endDate` (string, ISO) - Filter by end date
- `page` (number, default: 1)
- `limit` (number, default: 10)

Response (200):
```json
{
  "data": [
    {
      "id": "financial-123",
      "companyId": "company-456",
      "type": "REVENUE",
      "amount": 100000000000,
      "currency": "IDR",
      "date": "2024-06-25T00:00:00Z",
      "description": "Q2 2024 Revenue",
      "createdAt": "2024-06-25T10:00:00Z"
    }
  ],
  "summary": {
    "totalRevenue": 500000000000,
    "totalExpense": 150000000000,
    "netIncome": 350000000000,
    "totalAssets": 2000000000000,
    "totalLiability": 500000000000,
    "totalEquity": 1500000000000
  },
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 25
  }
}
```

### Create Financial Record
**POST** `/api/financials`

Headers:
```
Authorization: Bearer {token}
Content-Type: application/json
```

Request body:
```json
{
  "companyId": "company-456",
  "type": "REVENUE",
  "amount": 100000000000,
  "currency": "IDR",
  "date": "2024-06-25T00:00:00Z",
  "description": "Q2 2024 Revenue"
}
```

Response (201):
```json
{
  "id": "financial-456",
  "companyId": "company-456",
  "type": "REVENUE",
  "amount": 100000000000,
  "currency": "IDR",
  "date": "2024-06-25T00:00:00Z",
  "description": "Q2 2024 Revenue",
  "createdAt": "2024-06-25T10:00:00Z",
  "updatedAt": "2024-06-25T10:00:00Z"
}
```

---

## 🏗️ Assets Endpoints

### Get Assets
**GET** `/api/assets`

Headers:
```
Authorization: Bearer {token}
```

Query parameters:
- `companyId` (string, optional)
- `type` (string, optional) - PROPERTY, EQUIPMENT, VEHICLE, INVESTMENT, CASH, OTHER
- `status` (string, optional) - ACTIVE, INACTIVE, SUSPENDED
- `page` (number, default: 1)
- `limit` (number, default: 10)

Response (200):
```json
{
  "data": [
    {
      "id": "asset-123",
      "companyId": "company-456",
      "name": "Jakarta Office Building",
      "type": "PROPERTY",
      "value": 50000000000,
      "currency": "IDR",
      "acquiredDate": "2020-01-15T00:00:00Z",
      "status": "ACTIVE",
      "description": "Main office in Jakarta",
      "createdAt": "2024-06-25T10:00:00Z"
    }
  ],
  "summary": {
    "totalValue": 200000000000,
    "byType": {
      "PROPERTY": 100000000000,
      "EQUIPMENT": 50000000000,
      "VEHICLE": 30000000000,
      "INVESTMENT": 20000000000
    }
  },
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 25
  }
}
```

### Create Asset
**POST** `/api/assets`

Headers:
```
Authorization: Bearer {token}
Content-Type: application/json
```

Request body:
```json
{
  "companyId": "company-456",
  "name": "Jakarta Office Building",
  "type": "PROPERTY",
  "value": 50000000000,
  "currency": "IDR",
  "acquiredDate": "2020-01-15T00:00:00Z",
  "description": "Main office in Jakarta"
}
```

Response (201):
```json
{
  "id": "asset-456",
  "companyId": "company-456",
  "name": "Jakarta Office Building",
  "type": "PROPERTY",
  "value": 50000000000,
  "currency": "IDR",
  "acquiredDate": "2020-01-15T00:00:00Z",
  "status": "ACTIVE",
  "description": "Main office in Jakarta",
  "createdAt": "2024-06-25T10:00:00Z",
  "updatedAt": "2024-06-25T10:00:00Z"
}
```

### Update Asset
**PUT** `/api/assets/:id`

Headers:
```
Authorization: Bearer {token}
Content-Type: application/json
```

Request body:
```json
{
  "value": 55000000000,
  "status": "ACTIVE"
}
```

Response (200):
```json
{
  "id": "asset-456",
  "value": 55000000000,
  "status": "ACTIVE",
  "updatedAt": "2024-06-25T12:00:00Z"
}
```

### Delete Asset
**DELETE** `/api/assets/:id`

Headers:
```
Authorization: Bearer {token}
```

Response (200):
```json
{
  "message": "Asset deleted successfully"
}
```

---

## 👥 Users Endpoints

### Get All Users
**GET** `/api/users`

Headers:
```
Authorization: Bearer {token}
```

Query parameters:
- `role` (string, optional) - ADMIN, MANAGER, ANALYST, USER
- `status` (string, optional) - ACTIVE, INACTIVE, SUSPENDED
- `search` (string, optional) - Search by email or username
- `page` (number, default: 1)
- `limit` (number, default: 10)

Response (200):
```json
{
  "data": [
    {
      "id": "user-123",
      "email": "user@example.com",
      "username": "username",
      "firstName": "John",
      "lastName": "Doe",
      "phone": "08123456789",
      "role": "USER",
      "status": "ACTIVE",
      "createdAt": "2024-06-25T10:00:00Z",
      "updatedAt": "2024-06-25T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 50
  }
}
```

### Create User
**POST** `/api/users`

Headers:
```
Authorization: Bearer {token}
Content-Type: application/json
```

Request body:
```json
{
  "email": "newuser@example.com",
  "username": "newuser",
  "password": "SecurePassword123!",
  "firstName": "Jane",
  "lastName": "Smith",
  "phone": "08198765432",
  "role": "MANAGER"
}
```

Response (201):
```json
{
  "id": "user-456",
  "email": "newuser@example.com",
  "username": "newuser",
  "firstName": "Jane",
  "lastName": "Smith",
  "phone": "08198765432",
  "role": "MANAGER",
  "status": "ACTIVE",
  "createdAt": "2024-06-25T10:00:00Z"
}
```

### Get User Detail
**GET** `/api/users/:id`

Headers:
```
Authorization: Bearer {token}
```

Response (200):
```json
{
  "id": "user-123",
  "email": "user@example.com",
  "username": "username",
  "firstName": "John",
  "lastName": "Doe",
  "phone": "08123456789",
  "role": "USER",
  "status": "ACTIVE",
  "createdAt": "2024-06-25T10:00:00Z",
  "updatedAt": "2024-06-25T10:00:00Z"
}
```

### Update User
**PUT** `/api/users/:id`

Headers:
```
Authorization: Bearer {token}
Content-Type: application/json
```

Request body:
```json
{
  "firstName": "John",
  "lastName": "Updated",
  "phone": "08123456789",
  "role": "MANAGER",
  "status": "ACTIVE"
}
```

Response (200):
```json
{
  "id": "user-123",
  "email": "user@example.com",
  "firstName": "John",
  "lastName": "Updated",
  "phone": "08123456789",
  "role": "MANAGER",
  "status": "ACTIVE",
  "updatedAt": "2024-06-25T12:00:00Z"
}
```

### Delete User
**DELETE** `/api/users/:id`

Headers:
```
Authorization: Bearer {token}
```

Response (200):
```json
{
  "message": "User deleted successfully"
}
```

---

## 📊 Reports Endpoints

### Generate Report
**POST** `/api/reports/generate`

Headers:
```
Authorization: Bearer {token}
Content-Type: application/json
```

Request body:
```json
{
  "type": "PORTFOLIO_SUMMARY",
  "companyIds": ["company-123", "company-456"],
  "startDate": "2024-01-01T00:00:00Z",
  "endDate": "2024-06-25T23:59:59Z",
  "format": "PDF"
}
```

Response (201):
```json
{
  "id": "report-123",
  "type": "PORTFOLIO_SUMMARY",
  "status": "GENERATING",
  "format": "PDF",
  "createdAt": "2024-06-25T10:00:00Z",
  "completedAt": null,
  "fileUrl": null
}
```

### Get Reports
**GET** `/api/reports`

Headers:
```
Authorization: Bearer {token}
```

Query parameters:
- `type` (string, optional)
- `status` (string, optional) - GENERATING, COMPLETED, FAILED
- `page` (number, default: 1)
- `limit` (number, default: 10)

Response (200):
```json
{
  "data": [
    {
      "id": "report-123",
      "type": "PORTFOLIO_SUMMARY",
      "status": "COMPLETED",
      "format": "PDF",
      "createdAt": "2024-06-25T10:00:00Z",
      "completedAt": "2024-06-25T10:15:00Z",
      "fileUrl": "https://api.beegroup.com/reports/report-123.pdf"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 25
  }
}
```

### Get Report Detail
**GET** `/api/reports/:id`

Headers:
```
Authorization: Bearer {token}
```

Response (200):
```json
{
  "id": "report-123",
  "type": "PORTFOLIO_SUMMARY",
  "status": "COMPLETED",
  "format": "PDF",
  "createdAt": "2024-06-25T10:00:00Z",
  "completedAt": "2024-06-25T10:15:00Z",
  "fileUrl": "https://api.beegroup.com/reports/report-123.pdf",
  "summary": {
    "totalCompanies": 15,
    "totalAssets": 2500000000000,
    "portfolioValue": 5000000000000
  }
}
```

### Delete Report
**DELETE** `/api/reports/:id`

Headers:
```
Authorization: Bearer {token}
```

Response (200):
```json
{
  "message": "Report deleted successfully"
}
```

---

## 🔍 Health Check

### API Health
**GET** `/api/health`

Response (200):
```json
{
  "status": "ok",
  "timestamp": "2024-06-25T10:00:00Z",
  "version": "1.0.0"
}
```

---

## 📋 Error Handling

### Error Response Format
```json
{
  "error": "Error message",
  "code": "ERROR_CODE",
  "details": {}
}
```

### Common Error Codes

| Status | Code | Message |
|--------|------|---------|
| 400 | BAD_REQUEST | Invalid request parameters |
| 401 | UNAUTHORIZED | Missing or invalid token |
| 403 | FORBIDDEN | Insufficient permissions |
| 404 | NOT_FOUND | Resource not found |
| 409 | CONFLICT | Resource already exists |
| 500 | INTERNAL_ERROR | Internal server error |

### Example Error Response
```json
{
  "error": "Company not found",
  "code": "NOT_FOUND",
  "details": {
    "companyId": "invalid-id"
  }
}
```

---

## 📝 Pagination

Responses yang mendukung pagination mengikuti format:
```json
{
  "data": [],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "pages": 10
  }
}
```

---

## 🔐 Rate Limiting

- **Limit**: 1000 requests per hour per IP
- **Header Response**:
  ```
  X-RateLimit-Limit: 1000
  X-RateLimit-Remaining: 999
  X-RateLimit-Reset: 1624617600
  ```

---

## 📚 Additional Resources

- Postman Collection: [Link to Postman collection]
- API Status: [Link to status page]
- Support: support@beegroup.com
