1. Client-Facing Section
Welcome to the pave api bank design. 
This document describes a minimal version of the Pave Bank account API, which allows clients to:
- Create business accounts
- View account details
- Deposit funds from external sources
- Withdraw funds to internal or external destinations
- All requests and responses use JSON.

2. API Documentation

2.2 Endpoints

2.2.1 Create Account
POST /accounts

Request
{
    "business_id" : "ULID",
    "name" : "Account"
}

Response
{
    "id" : "ULID",
    "business_id" : "ULID",
    "name" : "Account",
    "account_number" : 1234567890,
    "status" : "active",
    "created_at" : "2025-02-02T12:00:00Z",
    "updated_at" : "2025-02-02T12:00:00Z"
}

Status Codes
- 201 Created - Account successfully created
- 400 Bad Request - Missing fields or invalid input
- 404 Not Found - Business ID doesn't exist
- 500 Internal Server Error

2.2.2 View Account
GET /accounts/{id}

Request
No request body as GET doesn't accept body

Response
{
    "id" : "ULID",
    "business_id" : "ULID",
    "account_number" : 1234567890,
    "name" : "Account",
    "balances" : [
        {
            "currency": "SGD",
            "amount": 200000
        }
    ],
    "subaccounts" : [],
    "status": "active",
    "created_at": "2025-02-02T12:00:00Z",
    "updated_at":2025-02-02T12:00:00Z
}

Status Codes
- 200 OK - Account found
- 400 Bad Request - Invalid ID format
- 404 Not Found - Account not Found

2.2.3 Create Deposit
POST /accounts/{account_id}/deposit

Request
{
    "amount" : 10000,
    "currency" : "SGD",
    "description" : "Deposit",
    "performed_by_user_id" : "ULID"
}

Response
{
    "id": "ULID",
    "account_id": "ULID",
    "amount": 10000,
    "currency": "SGD",
    "description":  "Deposit",
    "performed_by_user_id": "ULID",
    "type": "deposit",
    "status": "completed", 
    "created_at": "2025-02-02T12:05:00Z"
}

Status Codes
- 201 Created - Deposit recorded
- 400 Bad Request - Missing or invalid data
- 404 Not Found - Account doesn't exist
- 500 Internal Server Error

2.2.4 Create Withdrawal
POST /accounts/{account_id}/withdrawal

Request
{
  "amount": 50000,
  "currency": "SGD",
  "description": "withdrawal",
  "performed_by_user_id": "ULID",
  "destination_account_id": "ULID or null",
  "external_bank_details": {
    "account_number": "123456789",
    "bank_code": "DBSSGSGX",
    "country": "SG"
  }
}

Response
{
  "id": "ULID",
  "type": "withdrawal",
  "account_id": "ULID",
  "destination_account_id": "ULID or null",
  "external_bank_details": { ... or null },
  "amount": 50000,
  "currency": "SGD",
  "description": "External withdrawal",
  "performed_by_user_id": "ULID",
  "status": "completed",
  "created_at": "2025-02-02T12:10:00Z" 
}

Status Codes
- 201 Created - Withdrawal recorded
- 400 bad request - Missing or invalid data
- 404 Not Found - Account doesn't exist
- 422 Unprocessable Entity - Insufficient funds
- 500 Internal Server Errror

2.3 Error Format
{
  "error": {
    "code": "invalid_request",
    "message": "The currency field is required."
  }
}

Status Code	Meaning
- 400 Bad Request - Client sent invalid or missing fields
- 401 Unauthorized - Missing or invalid authentication token
- 404 Not Found - Account, business, or transaction does not exist
- 409 Conflict - Duplicate request or constraint violation
- 422 Unprocessable Entity - Insufficient funds, invalid state
- 500 Internal Server Error - Unexpected error on server

3. Engineering Notes
3.1 Data Models
3.1.1 Account
The data model represents a business-owned financial account that supports multi-currency balances, subaccounts, and full lifecycle management.

type Account struct {
    ID            string        `json:"id"`              
    BusinessID    string        `json:"business_id"`     
    Name          string        `json:"name"`           
    AccountNumber int64         `json:"account_number"`  
    Status        string        `json:"status"`          
    Balances      []Balance     `json:"balances"`
    Subaccounts   []Subaccount  `json:"subaccounts"`
    CreatedAt     time.Time     `json:"created_at"`
    UpdatedAt     time.Time     `json:"updated_at"`
}

3.1.2 Transaction
The data model represents any financial movement: deposits, withdrawals, and internal transfers.

type Transaction struct {
    ID                   string               `json:"id"`                     
    Type                 string               `json:"type"`                   
    SourceAccountID      string               `json:"source_account_id"`      
    DestinationAccountID string               `json:"destination_account_id"` 
    ExternalBankDetails  *ExternalBankDetails `json:"external_bank_details"`
    PerformedByUserID    string               `json:"performed_by_user_id"`   
    Amount               int64                `json:"amount"`                 
    Currency             string               `json:"currency"`
    Status               string               `json:"status"`                 
    Description          string               `json:"description"`        
    CreatedAt            time.Time            `json:"created_at"`
}

3.2 Identifier Strategy
- ULID for internal IDs (globally unique)
- Snowflake for account numbers (numeric and customer-friendly)
- Balances array support multicurrency
- Subaccounts allow internal organization
- Timestamps support auditability 

3.3 Money Representation
- All values stored in int64 minor units to avoid floating-point errors.

3.4 Business Rules & Assumptions
- Valid ULIDs required for all IDs
- Amount must be a positive integer
- Withdrawals require sufficient balance
- One of destination_account_id or external_bank_details must be provided
- Never destination_accound_id and external_bank_Details provided together
- All operations atomic
- AccountNumber generated via Snowflake
- Timestamps follow UTC

3.5 Edge Cases
3.5.1 Insufficient funds 
Trigger:
Account SGD balance = 10,000
User request withdrawal of 50,000

API Response
{
  "error": {
    "code": "insufficient_funds",
    "message": "The account does not have sufficient balance to complete this withdrawal."
  }
}
Status Code: 422 Unprocessable Entity

3.5.2 Missing fields 
Trigger:
If a required field is missing, the request is invalid.

API Response
{
  "error": {
    "code": "invalid_request",
    "message": "Field 'missing' is required."
  }
}
Status Code: 400 Bad Request

3.5.3 Invalid ULID 
Trigger:
invalid id 

API Response
{
  "error": {
    "code": "invalid",
    "message": "The provided account ID is not a valid ULID"
  }
}
Status Code: 400 Bad Request

3.5.4 Invalid external bank details 
Trigger:
The information for the details is invalid

API Response
{
  "error": {
    "code": "invalid_external_details",
    "message": "External bank details are incomplete."
  }
}
Status Code: 400 Bad Request

3.5.5 Both destination and external details provided
Trigger:
There is details for both internal bank account and external bank account.

API Response
{
  "error": {
    "code": "invalid_request",
    "message": "Provide either destination_account_id or external_bank_details."
  }
}
Status Code: 400 Bad Request

3.5.6
Neither destination nor external details provided

Trigger 
Missing field in either desination or external bank details

API Response
{
  "error": {
    "code": "missing_destination",
    "message": "Withdrawal must specify a destination account or external bank details."
  }
}

Status Code: 400 Bad Request

3.5.7 Currency Mismatch
Trigger:
Account tries to deposit USD when only having SGD

API Response:
{
  "error": {
    "code": "currency_not_supported",
    "message": "This account does not support the provided currency."
  }
}

Status Code: 422 Unprocessable Entity

3.5.8 Negative or Zero Amount
Trigger: 
Amount: -500 

API Response: 
{
  "error": {
    "code": "invalid_amount",
    "message": "Transaction amount must be a positive integer."
  }
}

Status Code: 400 Bad Request