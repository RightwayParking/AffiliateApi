# Webhook Integration Guide for Property Owners

This guide explains how to receive and process webhook notifications from our reservation system when bookings are completed at your property.

## Overview

When a customer completes a reservation at your property, our system will automatically send a webhook notification to your specified endpoint URL. This allows you to integrate reservation data directly into your own systems in real-time.

## What is a Webhook?

A webhook is an HTTP POST request sent to your server containing reservation data in JSON format. Think of it as an automated notification that gets sent to your system whenever a booking is completed.

## Setup Requirements

To receive webhooks from our system, you'll need:

1. **A webhook endpoint URL** - A URL on your server that can receive HTTP POST requests
2. **HTTPS enabled** - Your endpoint must use HTTPS for security
3. **A webhook secret** (optional but recommended) - Used to verify the webhook authenticity

## Configuration

These can be entered on a per property basis in your affiliate backend on https://rightwayparking.com. If you need Contact our support team to configure webhooks for your property. We'll need:

- **Webhook URL**: The endpoint where you want to receive notifications
- **Webhook Secret**: A secret key for signature validation (optional)

Example: `https://your-website.com/webhooks/reservations`

## Webhook Data Structure

When a reservation is completed, you'll receive a JSON payload with the following structure:

```json
{
    "event": "reservation.completed",
    "timestamp": 1625097600,
    "data": {
        "id": 12345,
        "customer": {
            "name": "John Doe",
            "email": "john@example.com",
            "phone": "+1234567890"
        },
        "property": {
            "id": 1,
            "name": "Your Property Name"
        },
        "pricing": {
            "total": "150.00",
            "paid": "50.00",
            "due_at_lot": "100.00",
            "rate": "120.00",
            "taxes": "15.00",
            "fees": "15.00",
            "special_pricing": {
              "type": "holiday_pricing",
              "events": [
                {
                  "description": "Special/Holiday Pricing",
                  "rate": "2.99",
                  "total": "8.97",
                  "days": 3,
                  "start_date": "2025-07-04",
                  "end_date": "2025-07-06",
                  "dates": {
                    "2": "2025-07-04",
                    "3": "2025-07-05",
                    "4": "2025-07-06"
                  }
                },
                {
                  "description": "Special/Holiday Pricing",
                  "rate": "2.50",
                  "total": "2.50",
                  "days": 1,
                  "start_date": "2025-07-07",
                  "end_date": "2025-07-08",
                  "dates": {
                    "5": "2025-07-07"
                   }
                }
              ]
            }
        },
        "reservation": {
            "check_in": "2024-07-15",
            "check_out": "2024-07-20"
        },
        "details": {
            "days": 5,
            "passengers": 4,
            "license_plate": "ABC123",
            "car_make": "Toyota",
            "car_color": "Blue",
            "ship_name": "Cruise Ship",
            "spaces": 1
        },
        "status": "completed",
        "completed_at": "2024-07-15 14:30:00"
    },
    "signature": "abc123..." // Only present if webhook secret is configured
}
```

## Data Field Descriptions

### Root Level
- **event**: Always "reservation.completed" for booking notifications
- **timestamp**: Unix timestamp when the webhook was sent
- **data**: The reservation information (see below)
- **signature**: HMAC signature for verification (if secret configured)

### Reservation Data
- **id**: Unique reservation ID in our system
- **status**: Always "completed" for successful bookings
- **completed_at**: Date/time when the reservation was completed

### Customer Information
- **name**: Customer's full name
- **email**: Customer's email address
- **phone**: Customer's phone number

### Property Information
- **id**: Your property ID in our system
- **name**: Your property name

### Pricing Details
- **total**: Total amount charged to customer
- **paid**: Amount paid upfront
- **due_at_lot**: Amount due at your location
- **rate**: Base rate for the booking
- **taxes**: Tax amount
- **fees**: Additional fees
- **special_pricing**: Array of special pricing applied

### Reservation Dates
- **check_in**: Check-in date (YYYY-MM-DD format)
- **check_out**: Check-out date (YYYY-MM-DD format)

### Reservation Details
- **days**: Number of days booked
- **passengers**: Number of passengers
- **license_plate**: Vehicle license plate
- **car_make**: Vehicle make/brand
- **car_color**: Vehicle color
- **ship_name**: Cruise ship name (if applicable)
- **spaces**: Number of parking spaces booked

## Creating Your Webhook Endpoint

### Basic PHP Example

```php
<?php
// webhook-handler.php

// Get the JSON payload
$input = file_get_contents('php://input');
$data = json_decode($input, true);

// Validate the webhook (if using signature)
if (!validateWebhook($data, 'your-webhook-secret')) {
    http_response_code(401);
    exit('Unauthorized');
}

// Process the reservation
if ($data['event'] === 'reservation.completed') {
    $reservation = $data['data'];
    
    // Your processing logic here
    processReservation($reservation);
    
    // Return success response
    http_response_code(200);
    echo json_encode(['status' => 'success']);
} else {
    http_response_code(400);
    echo json_encode(['status' => 'error', 'message' => 'Unknown event']);
}

function validateWebhook($data, $secret) {
    if (!isset($data['signature'])) {
        return false; // No signature provided
    }
    
    $payload = [
        'event' => $data['event'],
        'timestamp' => $data['timestamp'],
        'data' => $data['data']
    ];
    
    $expectedSignature = hash_hmac('sha256', json_encode($payload), $secret);
    return hash_equals($expectedSignature, $data['signature']);
}

function processReservation($reservation) {
    // Example: Save to database
    // Example: Send confirmation email
    // Example: Update inventory
    // Example: Log the booking
}
?>
```

### Node.js Example

```javascript
const express = require('express');
const crypto = require('crypto');
const app = express();

app.use(express.json());

app.post('/webhooks/reservations', (req, res) => {
    const data = req.body;
    
    // Validate webhook signature
    if (!validateWebhook(data, 'your-webhook-secret')) {
        return res.status(401).json({ error: 'Unauthorized' });
    }
    
    // Process the reservation
    if (data.event === 'reservation.completed') {
        const reservation = data.data;
        
        // Your processing logic here
        processReservation(reservation);
        
        res.json({ status: 'success' });
    } else {
        res.status(400).json({ error: 'Unknown event' });
    }
});

function validateWebhook(data, secret) {
    if (!data.signature) return false;
    
    const payload = {
        event: data.event,
        timestamp: data.timestamp,
        data: data.data
    };
    
    const expectedSignature = crypto
        .createHmac('sha256', secret)
        .update(JSON.stringify(payload))
        .digest('hex');
    
    return crypto.timingSafeEqual(
        Buffer.from(expectedSignature),
        Buffer.from(data.signature)
    );
}

function processReservation(reservation) {
    // Example: Save to database
    // Example: Send confirmation email
    // Example: Update inventory
    console.log('New reservation:', reservation.id);
}

app.listen(3000);
```

## Security: Webhook Signature Verification

If you've configured a webhook secret, each webhook will include a signature for verification. This ensures the webhook actually came from our system.

### How Signature Verification Works

1. We create a payload containing the event, timestamp, and data
2. We generate an HMAC-SHA256 signature using your secret key
3. The signature is included in the webhook
4. You recreate the same signature and compare it

### Why Use Signature Verification?

- **Security**: Prevents malicious requests from fake sources
- **Integrity**: Ensures the data hasn't been tampered with
- **Authentication**: Confirms the webhook came from our system

## Response Requirements

Your webhook endpoint should:

1. **Respond quickly** (within 30 seconds)
2. **Return HTTP 200** for successful processing
3. **Return HTTP 4xx/5xx** for errors
4. **Process idempotently** (handle duplicate webhooks gracefully)

### Example Response Formats

**Success:**
```json
{
    "status": "success",
    "message": "Reservation processed successfully"
}
```

**Error:**
```json
{
    "status": "error",
    "message": "Failed to process reservation",
    "code": "INVALID_DATA"
}
```

## Error Handling & Retries

Our system includes automatic retry logic:

- **Retry Count**: Up to 3 attempts
- **Retry Timing**: Exponential backoff (immediate, 1 minute, 5 minutes)
- **Retry Conditions**: HTTP 5xx errors or timeouts
- **No Retry**: HTTP 4xx errors (client errors)

## Testing Your Webhook

### Test Payload

You can test your webhook endpoint with this sample payload:

```json
{
    "event": "reservation.completed",
    "timestamp": 1625097600,
    "data": {
        "id": 99999,
        "customer": {
            "name": "Test Customer",
            "email": "test@example.com",
            "phone": "+1234567890"
        },
        "property": {
            "id": 1,
            "name": "Test Property"
        },
        "pricing": {
            "total": "100.00",
            "paid": "25.00",
            "due_at_lot": "75.00",
            "rate": "80.00",
            "taxes": "10.00",
            "fees": "10.00",
            "special_pricing": false
        },
        "reservation": {
            "check_in": "2024-07-15",
            "check_out": "2024-07-18"
        },
        "details": {
            "days": 3,
            "passengers": 2,
            "license_plate": "TEST123",
            "car_make": "Honda",
            "car_color": "Red",
            "ship_name": "Test Ship",
            "spaces": 1
        },
        "status": "completed",
        "completed_at": "2024-07-15 10:00:00"
    }
}
```

### Using cURL to Test

```bash
curl -X POST https://your-website.com/webhooks/reservations \
  -H "Content-Type: application/json" \
  -d @test-payload.json
```

## Common Use Cases

### 1. Inventory Management
Update your parking space availability based on bookings:

```php
function updateInventory($reservation) {
    $checkIn = $reservation['dates']['check_in'];
    $checkOut = $reservation['dates']['check_out'];
    $spaces = $reservation['details']['spaces'];
    
    // Update your inventory system
    markSpacesAsBooked($checkIn, $checkOut, $spaces);
}
```

### 2. Customer Communication
Send custom confirmation emails or SMS:

```php
function sendConfirmation($reservation) {
    $customer = $reservation['customer'];
    $details = $reservation['details'];
    
    sendEmail($customer['email'], 'Booking Confirmed', [
        'name' => $customer['name'],
        'checkin' => $reservation['dates']['check_in'],
        'total' => $reservation['pricing']['total']
    ]);
}
```

### 3. Reporting & Analytics
Track booking metrics:

```php
function recordAnalytics($reservation) {
    $metrics = [
        'property_id' => $reservation['property']['id'],
        'revenue' => $reservation['pricing']['total'],
        'days' => $reservation['details']['days'],
        'passengers' => $reservation['details']['passengers']
    ];
    
    saveToAnalytics($metrics);
}
```

## Troubleshooting

### Common Issues

**Webhook not received:**
- Check if your endpoint URL is correct and accessible
- Verify your server is accepting POST requests
- Check firewall settings

**Signature verification failing:**
- Ensure your secret matches what was configured
- Verify you're using the exact same JSON payload structure
- Check for extra whitespace or encoding issues

**Timeout errors:**
- Ensure your endpoint responds within 30 seconds
- Consider processing webhooks asynchronously

### Getting Help

If you're having trouble with webhook integration:

1. Check your server logs for error messages
2. Test your endpoint with the provided test payload
3. Contact our support team with your property ID and webhook URL

## Best Practices

1. **Always validate signatures** if using a webhook secret
2. **Handle duplicates gracefully** - use the reservation ID to prevent duplicate processing
3. **Respond quickly** - acknowledge receipt immediately, process later if needed
4. **Log webhook requests** for debugging and audit purposes
5. **Use HTTPS** for all webhook endpoints
6. **Implement proper error handling** for malformed data
7. **Monitor webhook failures** and alert on repeated failures

## Support

For technical support with webhook integration:

- **Email**: support@yourcompany.com
- **Documentation**: Include your property ID and webhook URL in support requests
- **Response Time**: We typically respond within 24 hours

---

*This documentation covers webhook integration for reservation notifications. For other API integrations or questions, please contact our support team.*
