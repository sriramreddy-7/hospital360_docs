# API Documentation

Hospital360 provides a comprehensive RESTful API for integration with external systems and custom applications.

## 📋 Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Base URL and Versioning](#base-url-and-versioning)
- [Common Headers](#common-headers)
- [Response Format](#response-format)
- [Error Handling](#error-handling)
- [Rate Limiting](#rate-limiting)
- [API Endpoints](#api-endpoints)
- [Code Examples](#code-examples)
- [SDKs and Libraries](#sdks-and-libraries)

## 🔍 Overview

The Hospital360 API allows you to:

- Manage patient records and appointments
- Access medical history and test results
- Handle billing and insurance information
- Generate reports and analytics
- Integrate with third-party healthcare systems
- Automate routine administrative tasks

### API Features

- **RESTful Design** - Standard HTTP methods and status codes
- **JSON Format** - All requests and responses use JSON
- **Authentication** - OAuth 2.0 and API key support
- **Pagination** - Efficient handling of large datasets
- **Filtering** - Advanced query capabilities
- **Real-time Updates** - WebSocket support for live data
- **Comprehensive Documentation** - Interactive API explorer

## 🔐 Authentication

Hospital360 API supports multiple authentication methods:

### API Key Authentication

```http
GET /api/v1/patients
Authorization: Bearer YOUR_API_KEY
```

### OAuth 2.0

1. **Authorization Request**
```http
GET /oauth/authorize?
    client_id=YOUR_CLIENT_ID&
    response_type=code&
    redirect_uri=YOUR_REDIRECT_URI&
    scope=read:patients,write:appointments
```

2. **Token Exchange**
```http
POST /oauth/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "code": "AUTH_CODE",
  "client_id": "YOUR_CLIENT_ID",
  "client_secret": "YOUR_CLIENT_SECRET",
  "redirect_uri": "YOUR_REDIRECT_URI"
}
```

### JWT Token Usage

```http
GET /api/v1/patients
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## 🌐 Base URL and Versioning

- **Base URL**: `https://api.hospital360.com`
- **Current Version**: `v1`
- **Full Base URL**: `https://api.hospital360.com/api/v1`

### Version Headers

```http
Accept: application/vnd.hospital360.v1+json
```

## 📤 Common Headers

All API requests should include:

```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer YOUR_TOKEN
X-Request-ID: unique-request-id (optional)
```

## 📊 Response Format

### Success Response

```json
{
  "status": "success",
  "data": {
    "id": 12345,
    "name": "John Doe",
    "email": "john.doe@example.com"
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "request_id": "req_123456789"
  }
}
```

### Paginated Response

```json
{
  "status": "success",
  "data": [...],
  "pagination": {
    "current_page": 1,
    "per_page": 20,
    "total": 150,
    "total_pages": 8,
    "next_page": 2,
    "prev_page": null
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z"
  }
}
```

## ⚠️ Error Handling

### Error Response Format

```json
{
  "status": "error",
  "error": {
    "code": "PATIENT_NOT_FOUND",
    "message": "Patient with ID 12345 was not found",
    "details": {
      "patient_id": 12345,
      "timestamp": "2024-01-15T10:30:00Z"
    }
  },
  "meta": {
    "request_id": "req_123456789"
  }
}
```

### HTTP Status Codes

| Code | Description |
|------|-------------|
| 200 | OK - Request successful |
| 201 | Created - Resource created successfully |
| 400 | Bad Request - Invalid request parameters |
| 401 | Unauthorized - Authentication required |
| 403 | Forbidden - Insufficient permissions |
| 404 | Not Found - Resource not found |
| 409 | Conflict - Resource already exists |
| 422 | Unprocessable Entity - Validation errors |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error - Server error |

### Common Error Codes

- `INVALID_REQUEST` - Malformed request
- `AUTHENTICATION_FAILED` - Invalid credentials
- `INSUFFICIENT_PERMISSIONS` - Access denied
- `RESOURCE_NOT_FOUND` - Requested resource doesn't exist
- `VALIDATION_ERROR` - Input validation failed
- `RATE_LIMIT_EXCEEDED` - Too many requests

## 🚦 Rate Limiting

API requests are limited to prevent abuse:

- **Standard Users**: 100 requests/minute
- **Premium Users**: 1000 requests/minute
- **Enterprise Users**: 10000 requests/minute

### Rate Limit Headers

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1642248600
```

## 📡 API Endpoints

### Authentication Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/auth/login` | User login |
| POST | `/auth/logout` | User logout |
| POST | `/auth/refresh` | Refresh token |
| GET | `/auth/me` | Get current user |

### Patient Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/patients` | List all patients |
| POST | `/patients` | Create new patient |
| GET | `/patients/{id}` | Get patient by ID |
| PUT | `/patients/{id}` | Update patient |
| DELETE | `/patients/{id}` | Delete patient |
| GET | `/patients/{id}/history` | Get medical history |

### Appointments

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/appointments` | List appointments |
| POST | `/appointments` | Create appointment |
| GET | `/appointments/{id}` | Get appointment |
| PUT | `/appointments/{id}` | Update appointment |
| DELETE | `/appointments/{id}` | Cancel appointment |

### Medical Records

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/medical-records` | List medical records |
| POST | `/medical-records` | Create medical record |
| GET | `/medical-records/{id}` | Get specific record |
| PUT | `/medical-records/{id}` | Update record |

### Billing

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/billing/invoices` | List invoices |
| POST | `/billing/invoices` | Create invoice |
| GET | `/billing/invoices/{id}` | Get invoice |
| POST | `/billing/payments` | Process payment |

## 💻 Code Examples

### JavaScript (Node.js)

```javascript
const axios = require('axios');

// Configure API client
const api = axios.create({
  baseURL: 'https://api.hospital360.com/api/v1',
  headers: {
    'Authorization': 'Bearer YOUR_API_KEY',
    'Content-Type': 'application/json'
  }
});

// Get patient list
async function getPatients() {
  try {
    const response = await api.get('/patients');
    return response.data;
  } catch (error) {
    console.error('Error:', error.response.data);
  }
}

// Create new patient
async function createPatient(patientData) {
  try {
    const response = await api.post('/patients', patientData);
    return response.data;
  } catch (error) {
    console.error('Error:', error.response.data);
  }
}
```

### Python

```python
import requests
import json

class Hospital360API:
    def __init__(self, api_key):
        self.base_url = 'https://api.hospital360.com/api/v1'
        self.headers = {
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json'
        }
    
    def get_patients(self):
        response = requests.get(
            f'{self.base_url}/patients',
            headers=self.headers
        )
        return response.json()
    
    def create_patient(self, patient_data):
        response = requests.post(
            f'{self.base_url}/patients',
            headers=self.headers,
            json=patient_data
        )
        return response.json()

# Usage
api = Hospital360API('YOUR_API_KEY')
patients = api.get_patients()
```

### cURL

```bash
# Get patients
curl -X GET "https://api.hospital360.com/api/v1/patients" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json"

# Create patient
curl -X POST "https://api.hospital360.com/api/v1/patients" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "first_name": "John",
    "last_name": "Doe",
    "email": "john.doe@example.com",
    "phone": "+1234567890",
    "date_of_birth": "1990-01-15"
  }'
```

## 📚 SDKs and Libraries

Official SDKs are available for:

- **JavaScript/Node.js**: `npm install hospital360-api`
- **Python**: `pip install hospital360-sdk`
- **PHP**: `composer require hospital360/api-client`
- **Java**: Maven/Gradle dependency available
- **C#/.NET**: NuGet package available

### JavaScript SDK Example

```javascript
const Hospital360 = require('hospital360-api');

const client = new Hospital360({
  apiKey: 'YOUR_API_KEY',
  environment: 'production' // or 'sandbox'
});

// Get patients with pagination
const patients = await client.patients.list({
  page: 1,
  per_page: 20,
  filter: {
    status: 'active'
  }
});

// Create appointment
const appointment = await client.appointments.create({
  patient_id: 12345,
  doctor_id: 67890,
  date: '2024-01-20',
  time: '10:00:00',
  type: 'consultation'
});
```

---

For detailed endpoint documentation, visit our [Interactive API Explorer](https://api.hospital360.com/docs) or refer to the individual endpoint documentation files in this directory.