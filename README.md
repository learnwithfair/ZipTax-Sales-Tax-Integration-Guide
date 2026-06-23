# ZipTax Sales Tax Integration Guide

[![Youtube][youtube-shield]][youtube-url]
[![Facebook][facebook-shield]][facebook-url]
[![Instagram][instagram-shield]][instagram-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

Thanks for visiting my GitHub account!

This document explains how to integrate ZipTax into a Laravel marketplace or e-commerce application for automated sales tax calculation.

The implementation supports:

* Vendor-level ZipTax API key configuration
* Manual and automated sales tax options
* Vendor-specific enabled U.S. states
* Address-based tax lookup
* Latitude/longitude-based tax lookup
* Postal-code-based tax lookup
* Taxability Information Code (TIC) support
* Secure API key encryption and decryption
* Checkout tax calculation

---

## Table of Contents

1. [What is ZipTax](#what-is-ziptax)
2. [How ZipTax Works](#how-ziptax-works)
3. [Creating a ZipTax Account](#creating-a-ziptax-account)
4. [Creating a ZipTax API Key](#creating-a-ziptax-api-key)
5. [ZipTax API Authentication](#ziptax-api-authentication)
6. [Supported Tax Lookup Methods](#supported-tax-lookup-methods)
7. [Taxability Information Code (TIC)](#taxability-information-code-tic)
8. [Free Plan and TIC Limitation](#free-plan-and-tic-limitation)
9. [Recommended Database Structure](#recommended-database-structure)
10. [Vendor Tax Configuration Flow](#vendor-tax-configuration-flow)
11. [Checkout Tax Calculation Flow](#checkout-tax-calculation-flow)
12. [ZipTax API Request Examples](#ziptax-api-request-examples)
13. [ZipTax API Response Examples](#ziptax-api-response-examples)
14. [Postman API Examples](#postman-api-examples)
15. [Laravel Service Integration Example](#laravel-service-integration-example)
16. [Error Handling](#error-handling)
17. [Security Guidelines](#security-guidelines)
18. [Reference Links](#reference-links)

---

## What is ZipTax

ZipTax is a U.S. sales tax rate lookup service. It helps applications determine the correct sales tax rate based on a customer’s location.

Sales tax rates can vary by:

* State
* County
* City
* Local tax district
* Exact street address

Instead of manually maintaining tax rates for every jurisdiction, the application sends location data to ZipTax and receives the applicable tax rate.

ZipTax is commonly used during checkout, order creation, invoice generation, and tax estimation.

---

## How ZipTax Works

The standard tax calculation flow is:

1. A vendor connects their ZipTax account.
2. The vendor enters their ZipTax API key.
3. The vendor selects the U.S. states where they collect sales tax.
4. A customer enters a shipping address during checkout.
5. The application checks whether the customer’s state is enabled for that vendor.
6. If the state is enabled, the application sends the address to ZipTax.
7. ZipTax returns the applicable tax rate.
8. The application calculates the tax amount.
9. The tax amount is added to the order total.

Example:

```text
Product subtotal: $100.00
ZipTax rate: 7.75%
Tax amount: $7.75
Order total: $107.75
```

---

## Creating a ZipTax Account

Each vendor should create and manage their own ZipTax account.

Recommended setup process:

1. Visit the ZipTax website.
2. Create a vendor account.
3. Complete any required verification.
4. Open the ZipTax dashboard.
5. Create an API key.
6. Configure address-based lookup if checkout tax is calculated from shipping addresses.
7. Copy the API key.
8. Add the API key to the vendor shop settings in the application.

ZipTax website:

```text
https://www.zip.tax/
```

---

## Creating a ZipTax API Key

ZipTax uses an API key to authenticate requests.

The API key belongs to the vendor’s ZipTax account and should be stored securely.

API key creation guide:

```text
https://docs.zip.tax/v-6-0/guides/tutorials/how-to-create-an-api-key
```

Recommended application flow:

1. Vendor selects `ZipTax` from the tax provider dropdown.
2. Vendor enters their ZipTax API key.
3. Backend validates the request.
4. Backend encrypts and stores the API key.
5. Backend decrypts the API key only when calling ZipTax.

Laravel encryption example:

```php
$encryptedApiKey = encrypt($request->ziptax_api_key);
```

Laravel decryption example:

```php
$apiKey = decrypt($shopInfo->ziptax_api_key);
```

---

## ZipTax API Authentication

ZipTax API requests use the API key in the request header.

```http
X-API-Key: YOUR_API_KEY
```

Example:

```bash
curl -H "X-API-Key: YOUR_API_KEY" \
"https://api.zip-tax.com/request/v60?address=200+Spectrum+Center+Dr+Irvine+CA+92618"
```

Important rules:

* API calls must be made from the backend.
* Never expose the API key in frontend code.
* Never return the decrypted API key in API responses.
* Never store the API key in browser local storage.
* Never log API keys.

---

## Supported Tax Lookup Methods

ZipTax supports three main lookup methods.

### 1. By Address

Address lookup is the recommended option for e-commerce checkout.

The application sends a full shipping address and ZipTax returns the applicable tax rate for that location.

Documentation:

```text
https://docs.zip.tax/v-6-0/guides/rest-api/by-address
```

Example:

```http
GET https://api.zip-tax.com/request/v60?address=200%20Spectrum%20Center%20Dr%20Irvine%20CA%2092618
X-API-Key: YOUR_API_KEY
```

Recommended use cases:

* Checkout tax calculation
* Shipping tax calculation
* Order tax calculation
* Invoice tax calculation

### 2. By Latitude and Longitude

This method uses geographic coordinates.

Documentation:

```text
https://docs.zip.tax/v-6-0/guides/rest-api/by-lat-lng
```

Example:

```http
GET https://api.zip-tax.com/request/v60?lat=33.6500&lng=-117.8400
X-API-Key: YOUR_API_KEY
```

Recommended use cases:

* Delivery applications
* Map-based applications
* Location-aware marketplace applications
* Address autocomplete systems that provide coordinates

### 3. By Postal Code

This method uses a ZIP code or postal code.

Documentation:

```text
https://docs.zip.tax/v-6-0/guides/rest-api/by-postal-code
```

Example:

```http
GET https://api.zip-tax.com/request/v60?postal_code=92618
X-API-Key: YOUR_API_KEY
```

Recommended use cases:

* Tax estimate tools
* Early checkout estimates
* Tax calculator pages

Postal-code lookup can be less accurate than a full address because one postal code may contain multiple tax jurisdictions.

---

## Taxability Information Code (TIC)

A Taxability Information Code, also called TIC, identifies the tax classification of a product or service.

Different product types can have different tax rules depending on the state.

Examples:

| Product Type      | Example TIC Purpose                         |
| ----------------- | ------------------------------------------- |
| Food              | May have special tax rules                  |
| Clothing          | May have state-specific exemptions          |
| Prepared food     | May be taxed differently from grocery food  |
| Wellness services | May have different tax treatment            |
| Handmade goods    | May have a different product classification |

TIC should be stored at the most detailed product classification level.

Recommended structure:

```text
categories
    id
    name
    taxability_code

sub_categories
    id
    category_id
    sub_category_name
    taxability_code
```

Product relationship:

```text
Product
    belongsTo SubCategory

SubCategory
    contains taxability_code
```

Laravel example:

```php
$taxabilityCode = $product->subCategory?->taxability_code;
```

---

## Free Plan and TIC Limitation

The ZipTax free plan may not support TIC-based or product-specific taxability calculations.

This means the free plan may allow location-based tax lookup, but may not apply special tax rules based on product classification.

Recommended fallback behavior:

1. Always store TIC in the database.
2. Check whether the vendor’s ZipTax plan supports TIC.
3. If TIC is supported, include it in the ZipTax request.
4. If TIC is not supported, use address-based tax lookup without TIC.
5. Notify the vendor if they need a paid plan for product-specific tax rules.

---

## Recommended Database Structure

### `states`

Stores all 50 U.S. states.

```text
id
name
code
status
created_at
updated_at
```

### `shop_infos`

Stores vendor tax provider settings.

```text
id
user_id
tax_provider
ziptax_api_key
created_at
updated_at
```

Example:

```text
tax_provider = manual
tax_provider = ziptax
```

### `vendor_states`

Stores state-level tax collection settings for each vendor.

```text
id
user_id
state_id
enabled
created_at
updated_at
```

Example:

| user_id | state_id | enabled |
| ------: | -------: | ------: |
|      15 |        5 |       1 |
|      15 |        9 |       0 |
|      15 |       43 |       1 |

---

## Vendor Tax Configuration Flow

The vendor tax settings page should provide:

1. Manual tax setup option.
2. Automated sales tax setup option.
3. Tax provider dropdown.
4. ZipTax API key input field.
5. List of all 50 U.S. states.
6. On/off toggle for each state.
7. Default state toggle value set to `OFF`.
8. Ability to disconnect ZipTax and return to manual tax setup.

Example request to save ZipTax settings:

```json
{
  "tax_provider": "ziptax",
  "ziptax_api_key": "vendor_zip_tax_api_key",
  "states": [
    {
      "id": 5,
      "enabled": true
    },
    {
      "id": 9,
      "enabled": false
    },
    {
      "id": 43,
      "enabled": true
    }
  ]
}
```

When the vendor selects manual tax mode:

```json
{
  "tax_provider": "manual"
}
```

Expected behavior:

* Set `tax_provider` to `manual`.
* Clear or ignore the ZipTax API key.
* Disable all vendor state toggles.
* Stop automated ZipTax calculations.

---

## Checkout Tax Calculation Flow

Recommended flow:

1. Get the product vendor.
2. Get the vendor shop tax settings.
3. Check the selected tax provider.
4. If provider is `manual`, use manual tax logic.
5. If provider is `ziptax`, check the customer shipping state.
6. Check whether that state is enabled for the vendor.
7. If the state is disabled, return zero automated tax.
8. Get the product subcategory TIC.
9. Call ZipTax with the vendor API key and customer location.
10. Calculate the tax amount using the returned rate.
11. Save tax details with the order.

Pseudo-code:

```php
if ($shopInfo->tax_provider !== 'ziptax') {
    return $manualTaxService->calculate($order);
}

$isStateEnabled = VendorState::where('user_id', $vendor->id)
    ->where('state_id', $shippingStateId)
    ->where('enabled', true)
    ->exists();

if (! $isStateEnabled) {
    return [
        'tax_rate' => 0,
        'tax_amount' => 0,
        'message' => 'Tax collection is not enabled for this shipping state.',
    ];
}

$taxabilityCode = $product->subCategory?->taxability_code;

return $zipTaxService->calculate(
    vendor: $vendor,
    address: $shippingAddress,
    amount: $productAmount,
    taxabilityCode: $taxabilityCode
);
```

---

## ZipTax API Request Examples

### By Address

```http
GET https://api.zip-tax.com/request/v60?address=200%20Spectrum%20Center%20Dr%20Irvine%20CA%2092618
X-API-Key: YOUR_API_KEY
```

### By Latitude and Longitude

```http
GET https://api.zip-tax.com/request/v60?lat=33.6500&lng=-117.8400
X-API-Key: YOUR_API_KEY
```

### By Postal Code

```http
GET https://api.zip-tax.com/request/v60?postal_code=92618
X-API-Key: YOUR_API_KEY
```

Tax Rates API reference:

```text
https://docs.zip.tax/v-6-0/api-reference/tax-rates/get-tax-rates-v-60?explorer=true
```

---

## ZipTax API Response Examples

The exact response fields can vary depending on the lookup method and account plan. The following is an example of the type of response the application should normalize internally.

```json
{
  "status": "success",
  "address": {
    "formatted_address": "200 Spectrum Center Dr, Irvine, CA 92618"
  },
  "tax_rate": 0.0775,
  "tax_rate_percent": 7.75,
  "tax_components": {
    "state_rate": 0.06,
    "county_rate": 0.0025,
    "city_rate": 0.005,
    "district_rate": 0.01
  }
}
```

Example normalized Laravel service response:

```json
{
  "success": true,
  "message": "Tax calculated successfully.",
  "data": {
    "shipping_state": "CA",
    "tax_rate": 0.0775,
    "tax_rate_percent": 7.75,
    "subtotal": 100.00,
    "tax_amount": 7.75,
    "total": 107.75
  }
}
```

Example when tax collection is disabled for the shipping state:

```json
{
  "success": true,
  "message": "Tax collection is not enabled for this shipping state.",
  "data": {
    "shipping_state": "FL",
    "tax_rate": 0,
    "tax_rate_percent": 0,
    "subtotal": 100.00,
    "tax_amount": 0,
    "total": 100.00
  }
}
```

Example when ZipTax API key is missing or invalid:

```json
{
  "success": false,
  "message": "Unable to calculate tax. Please verify the ZipTax configuration.",
  "data": []
}
```

---

## Postman API Examples

### ZipTax Request by Address

**Method**

```text
GET
```

**URL**

```text
https://api.zip-tax.com/request/v60?address=200%20Spectrum%20Center%20Dr%20Irvine%20CA%2092618
```

**Headers**

| Key         | Value              |
| ----------- | ------------------ |
| `X-API-Key` | `YOUR_API_KEY`     |
| `Accept`    | `application/json` |

**Postman cURL Import**

```bash
curl --location --request GET \
'https://api.zip-tax.com/request/v60?address=200%20Spectrum%20Center%20Dr%20Irvine%20CA%2092618' \
--header 'X-API-Key: YOUR_API_KEY' \
--header 'Accept: application/json'
```

**Example Response**

```json
{
  "status": "success",
  "tax_rate": 0.0775,
  "tax_rate_percent": 7.75,
  "address": {
    "formatted_address": "200 Spectrum Center Dr, Irvine, CA 92618"
  }
}
```

### ZipTax Request by Latitude and Longitude

**Method**

```text
GET
```

**URL**

```text
https://api.zip-tax.com/request/v60?lat=33.6500&lng=-117.8400
```

**Headers**

| Key         | Value              |
| ----------- | ------------------ |
| `X-API-Key` | `YOUR_API_KEY`     |
| `Accept`    | `application/json` |

**Postman cURL Import**

```bash
curl --location --request GET \
'https://api.zip-tax.com/request/v60?lat=33.6500&lng=-117.8400' \
--header 'X-API-Key: YOUR_API_KEY' \
--header 'Accept: application/json'
```

**Example Response**

```json
{
  "status": "success",
  "tax_rate": 0.0775,
  "tax_rate_percent": 7.75
}
```

### ZipTax Request by Postal Code

**Method**

```text
GET
```

**URL**

```text
https://api.zip-tax.com/request/v60?postal_code=92618
```

**Headers**

| Key         | Value              |
| ----------- | ------------------ |
| `X-API-Key` | `YOUR_API_KEY`     |
| `Accept`    | `application/json` |

**Postman cURL Import**

```bash
curl --location --request GET \
'https://api.zip-tax.com/request/v60?postal_code=92618' \
--header 'X-API-Key: YOUR_API_KEY' \
--header 'Accept: application/json'
```

**Example Response**

```json
{
  "status": "success",
  "postal_code": "92618",
  "tax_rate": 0.0775,
  "tax_rate_percent": 7.75
}
```

---

## Laravel Service Integration Example

Create the service:

```bash
php artisan make:class Services/ZipTaxService
```

Create file:

```text
app/Services/ZipTaxService.php
```

```php
<?php

namespace App\Services;

use App\Models\User;
use Illuminate\Support\Facades\Http;

class ZipTaxService
{
    public function calculateByAddress(
        User $vendor,
        string $address,
        float $subtotal,
        ?string $taxabilityCode = null
    ): array {
        $shopInfo = $vendor->shopInfo;

        if (
            ! $shopInfo ||
            $shopInfo->tax_provider !== 'ziptax' ||
            empty($shopInfo->ziptax_api_key)
        ) {
            return [
                'success' => false,
                'message' => 'ZipTax is not configured for this vendor.',
                'data' => [],
            ];
        }

        $query = [
            'address' => $address,
        ];

        if ($taxabilityCode) {
            $query['taxability_code'] = $taxabilityCode;
        }

        $response = Http::timeout(15)
            ->acceptJson()
            ->withHeaders([
                'X-API-Key' => decrypt($shopInfo->ziptax_api_key),
            ])
            ->get('https://api.zip-tax.com/request/v60', $query);

        if (! $response->successful()) {
            return [
                'success' => false,
                'message' => 'Unable to retrieve tax information from ZipTax.',
                'data' => [],
            ];
        }

        $zipTaxData = $response->json();

        $taxRate = (float) data_get($zipTaxData, 'tax_rate', 0);
        $taxAmount = round($subtotal * $taxRate, 2);

        return [
            'success' => true,
            'message' => 'Tax calculated successfully.',
            'data' => [
                'tax_rate' => $taxRate,
                'tax_rate_percent' => $taxRate * 100,
                'subtotal' => $subtotal,
                'tax_amount' => $taxAmount,
                'total' => round($subtotal + $taxAmount, 2),
                'zip_tax_response' => $zipTaxData,
            ],
        ];
    }
}
```

---

## Error Handling

The application should handle the following scenarios.

| Scenario                           | Recommended Behavior                              |
| ---------------------------------- | ------------------------------------------------- |
| Vendor has no ZipTax configuration | Use manual tax logic or return zero automated tax |
| ZipTax API key is missing          | Return a configuration error                      |
| ZipTax API key is invalid          | Return a clear connection error                   |
| Shipping state is disabled         | Return zero automated tax                         |
| Shipping address is incomplete     | Ask customer to provide a complete address        |
| ZipTax request fails               | Return a tax calculation error                    |
| TIC is unavailable                 | Use location-based lookup if allowed              |
| TIC is unsupported by plan         | Use location-based lookup without TIC             |

---

## Security Guidelines

* Encrypt API keys before storing them.
* Decrypt API keys only when sending backend API requests.
* Never expose API keys to frontend applications.
* Never include API keys in API responses.
* Never log API keys.
* Use HTTP timeouts for external API calls.
* Handle failed API responses safely.
* Store only required tax details with the order.

---

## Reference Links

| Resource                         | URL                                                                                 |
| -------------------------------- | ----------------------------------------------------------------------------------- |
| ZipTax Website                   | https://www.zip.tax/                                                                |
| REST API Overview                | https://docs.zip.tax/v-6-0/guides/rest-api/overview                                 |
| Lookup by Address                | https://docs.zip.tax/v-6-0/guides/rest-api/by-address                               |
| Lookup by Latitude and Longitude | https://docs.zip.tax/v-6-0/guides/rest-api/by-lat-lng                               |
| Lookup by Postal Code            | https://docs.zip.tax/v-6-0/guides/rest-api/by-postal-code                           |
| Create API Key                   | https://docs.zip.tax/v-6-0/guides/tutorials/how-to-create-an-api-key                |
| Tax Rates API Reference          | https://docs.zip.tax/v-6-0/api-reference/tax-rates/get-tax-rates-v-60?explorer=true |

---

## Author

Developed and maintained by [MD. Rahatul Rabbi](https://github.com/learnwithfair).

---

## Follow

[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/github.svg' alt='github' height='30'>](https://github.com/learnwithfair)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/facebook.svg' alt='facebook' height='30'>](https://www.facebook.com/learnwithfair/)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/instagram.svg' alt='instagram' height='30'>](https://www.instagram.com/learnwithfair/)
[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/youtube.svg' alt='YouTube' height='30'>](https://www.youtube.com/@learnwithfair)

---

<!-- MARKDOWN LINKS -->
[youtube-shield]: https://img.shields.io/badge/-Youtube-black.svg?style=flat-square&logo=youtube&color=555&logoColor=white
[youtube-url]: https://youtube.com/@learnwithfair
[facebook-shield]: https://img.shields.io/badge/-Facebook-black.svg?style=flat-square&logo=facebook&color=555&logoColor=white
[facebook-url]: https://facebook.com/learnwithfair
[instagram-shield]: https://img.shields.io/badge/-Instagram-black.svg?style=flat-square&logo=instagram&color=555&logoColor=white
[instagram-url]: https://instagram.com/learnwithfair
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/company/learnwithfair

#learnwithfair #rahtulrabbi #rahatul-rabbi #learn-with-fair
