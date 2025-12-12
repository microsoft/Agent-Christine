# End-to-End Guide to Using the Azure Marketplace Private Offer Submission API

This guide provides a comprehensive, step-by-step walkthrough for using the Private Offer Submission API in the Azure Marketplace. It focuses on the submission process for creating, managing, and submitting private offers, including ISV-to-customer, ISV-to-CSP, and multiparty private offers (MPOs). The content is based on official Microsoft documentation and schemas, expanded with full examples for practical implementation. Private offers allow publishers (ISVs) to offer customized pricing, terms, and configurations to specific customers or CSP partners, with transactions handled through the Microsoft Marketplace.

Private offers provide benefits like negotiated pricing, flexible billing, time-bound discounts, and bundling of up to 10 products. They support various product types, including SaaS, Azure Virtual Machines, Azure Applications, Professional Services, and Virtual Machine Software Reservations (VMSR). For customers, purchases count toward Azure commitments like MACC, and the process involves preparation, acceptance, and purchase via the Azure portal.

**Note:** This API is part of the Microsoft Partner Center Product Ingestion APIs and requires a Partner Center account. It uses Microsoft Entra ID for authentication. The guide assumes you are a publisher (ISV or channel partner) with the necessary roles.

## Prerequisites

Before using the API, ensure the following:

### Account and Role Setup
- **Publisher Role**: You must be enrolled in the Microsoft AI Cloud Partner Program and have a Marketplace account in Partner Center. Roles include Marketplace developer, manager, or account owner. The Microsoft Entra ID app must be assigned the Manager role in Partner Center.
- **Tax Profile**: Complete and verify your tax profile in Partner Center (under Account settings > Payout and tax profiles). For US/UK/Canada markets, specific requirements apply (e.g., resale certificates for US).
- **Public Offers**: Your products must be published in the Marketplace. A public transactable plan is typically required for SaaS, VM, and Azure Applications, but Professional Services offers can be private-only. Opt into CSP reselling if targeting CSP partners.
- **Customer/CSP Readiness**: For customers, ensure they have an eligible Azure billing account (MCA, EA, or MOSP), proper roles (e.g., billing account owner for acceptance), and run the private offer eligibility report to verify settings like Marketplace purchase policy and payment instruments. Provide them with the billing account ID (format example: `ac357579-e860-54a6-80b3-66958aea67fe:7471d04e-f696-4d20-af34-fa78d51e419c_2019-05-31`—retrieve from Azure Portal > Cost Management + Billing > Billing accounts).

### Authentication Setup
- Create a Microsoft Entra ID application associated with your Partner Center account.
- Obtain: Tenant ID, Client ID, and Client Secret.
- Use the client credentials flow to get an access token (expires in 60 minutes; refresh as needed).

**Example Token Request (using curl, v1 endpoint as per private offers doc):**
```bash
curl -X POST \
  'https://login.microsoftonline.com/<tenant_id>/oauth2/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials&client_id=<client_id>&client_secret=<client_secret>&resource=https://graph.microsoft.com/'
```

Response:
```json
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "access_token": "<your_access_token>"
}
```

Alternatively, use v2 endpoint:
```bash
curl -X POST \
  'https://login.microsoftonline.com/<tenant_id>/oauth2/v2.0/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials&client_id=<client_id>&client_secret=<client_secret>&scope=https://graph.microsoft.com/.default'
```

Include the token in API requests: `Authorization: Bearer <access_token>`.

### Tools and Environment
- Use tools like Postman or curl for API calls.
- Familiarize yourself with JSON schemas for requests (see Schemas section).
- Poll job status no more than once per minute.

## Step 1: Retrieve Products and Plans

Before creating a private offer, query your available products and plans.

### Endpoints
- **Get All Products**: `GET https://graph.microsoft.com/rp/product-ingestion/product?$version=2022-07-01`
- **Get Plans for a Product**: `GET https://graph.microsoft.com/rp/product-ingestion/plan?product=<product-id>&$version=2022-07-01`

**Example Request (Get Products):**
```bash
curl -X GET \
  'https://graph.microsoft.com/rp/product-ingestion/product?$version=2022-07-01' \
  -H 'Authorization: Bearer <access_token>'
```

**Example Response:**
```json
{
  "value": [
    {
      "id": "product/34771906-9711-4196-9f60-4af380fd5042",
      "name": "My SaaS Product",
      "type": "saas"
    }
  ]
}
```

Use the `product-id` and `plan-id` in subsequent requests.

## Step 2: Create a Private Offer

Private offers are created via a configuration endpoint that starts a job. Poll the job status, then retrieve results. There are three main types:
- **ISV-to-Customer**: Direct to end-customer.
- **ISV-to-CSP**: To CSP partners for resale.
- **Multiparty (MPO)**: Collaborative with a channel partner. ISV must submit first as originator; channel partner cannot create independently.

All use the same endpoint: `POST https://graph.microsoft.com/rp/product-ingestion/configure?$version=2022-07-01`

### General Request Structure
- Body schema: `https://schema.mp.microsoft.com/schema/configure/2022-07-01`
- Include a `resources` array with the private offer details.

Key fields:
- `privateOfferType`: `customerPromotion` (ISV-to-customer), `cspPromotion` (ISV-to-CSP), `multipartyPromotionOriginator` (MPO ISV), `multipartyPromotionChannelPartner` (MPO Seller).
- `offerPricingType`: `editExistingOfferPricingOnly` (discount on existing; only valid for VM images and Azure apps), `saasNewCustomizedPlans` (custom SaaS plans; only for SaaS), `vmSoftwareReservations` (VMSR).
- `state`: `draft` or `live` (to submit).
- `beneficiaries`: Customer/CSP IDs (use exact billing account ID format from eligibility report or portal).
- `pricing`: Array of discounts/margins (percentage or absolute).
- `termsAndConditionsDocs`: SAS URLs for custom terms (PDF format recommended; must be readable by Microsoft service).

For MPO:
- ISV creates first (originator).
- Channel partner completes (seller), adding markup.

### Example 1: ISV-to-Customer Private Offer (Percentage Discount on SaaS)
```json
{
  "$schema": "https://schema.mp.microsoft.com/schema/configure/2022-07-01",
  "resources": [
    {
      "$schema": "https://schema.mp.microsoft.com/schema/private-offer/2023-07-15",
      "name": "CustomerPrivateOffer",
      "privateOfferType": "customerPromotion",
      "state": "live",
      "offerPricingType": "editExistingOfferPricingOnly",
      "variableStartDate": true,
      "end": "2026-12-31",
      "acceptBy": "2025-12-31",
      "notificationContacts": ["customer@contoso.com"],
      "beneficiaries": [
        {
          "id": "ac357579-e860-54a6-80b3-66958aea67fe:7471d04e-f696-4d20-af34-fa78d51e419c_2019-05-31",
          "description": "Customer Billing Account"
        }
      ],
      "pricing": [
        {
          "product": "product/34771906-9711-4196-9f60-4af380fd5042",
          "plan": "plan/basic",
          "discountType": "percentage",
          "discountPercentage": 10
        }
      ],
      "termsAndConditionsDocs": [
        {
          "sasUrl": "https://blob.storage.azure.com/terms/terms.pdf?sas-token",
          "fileName": "terms.pdf",
          "customerFacingDocumentName": "Custom Terms"
        }
      ],
      "notes": "10% discount for 1 year"
    }
  ]
}
```

### Example 2: ISV-to-CSP Private Offer (Margin for Resale)
```json
{
  "$schema": "https://schema.mp.microsoft.com/schema/configure/2022-07-01",
  "resources": [
    {
      "$schema": "https://schema.mp.microsoft.com/schema/private-offer/2023-07-15",
      "name": "CSPPrivateOffer",
      "privateOfferType": "cspPromotion",
      "state": "live",
      "variableStartDate": false,
      "start": "2025-01-01",
      "end": "2025-12-31",
      "notificationContacts": ["csp@partner.com"],
      "beneficiaries": [
        {
          "id": "csp-partner-id",
          "description": "CSP Partner"
        }
      ],
      "pricing": [
        {
          "product": "product/34771906-9711-4196-9f60-4af380fd5042",
          "plan": "plan/pro",
          "discountType": "percentage",
          "discountPercentage": 15
        }
      ],
      "notes": "15% margin for CSP resale"
    }
  ]
}
```

### Example 3: MPO Creation (ISV Originator - Absolute Pricing for SaaS Custom Plan)
First, retrieve base pricing: `GET https://graph.microsoft.com/rp/product-ingestion/price-and-availability-private-offer-plan/{productId}?plan={planId}&$version=2023-07-15`

Then:
```json
{
  "$schema": "https://schema.mp.microsoft.com/schema/configure/2022-07-01",
  "resources": [
    {
      "$schema": "https://schema.mp.microsoft.com/schema/private-offer-mpo-originator/2023-07-15",
      "name": "MPOOriginatorOffer",
      "state": "live",
      "privateOfferType": "multipartyPromotionOriginator",
      "offerPricingType": "saasNewCustomizedPlans",
      "customerContractRenewal": false,
      "variableStartDate": true,
      "end": "2026-06-30",
      "acceptBy": "2025-06-30",
      "termsAndConditionsDocs": [
        {
          "sasUrl": "https://blob.storage.azure.com/terms/isv-terms.pdf?sas-token",
          "fileName": "isv-terms.pdf",
          "customerFacingDocumentName": "ISV Terms"
        }
      ],
      "notificationContacts": ["isv@contoso.com"],
      "beneficiaries": [
        {
          "id": "ac357579-e860-54a6-80b3-66958aea67fe:7471d04e-f696-4d20-af34-fa78d51e419c_2019-05-31",
          "description": "End Customer"
        }
      ],
      "partners": [
        {
          "id": "channel-partner-id",
          "partnerName": "Channel Partner Inc.",
          "location": "United States"
        }
      ],
      "pricing": [
        {
          "product": "product/7ba807c8-386a-4efe-80f1-b97bf8a554f8",
          "discountType": "absolute",
          "priceDetails": {
            "resourceName": "customSaaSPricing",
            "$schema": "https://schema.mp.microsoft.com/schema/price-and-availability-private-offer-plan/2023-07-15",
            "recurrentPrice": {
              "billingTerm": "monthly",
              "price": 500.00
            },
            "customMeters": [
              {
                "meterId": "meter1",
                "quantity": 1000
              }
            ],
            "userLimits": {
              "maxUsers": 50
            }
          },
          "basePlan": "plan/standard",
          "newPlanDetails": {
            "name": "CustomPlan",
            "description": "Customized SaaS Plan with Limits"
          }
        }
      ],
      "notes": "Custom SaaS plan for MPO"
    }
  ]
}
```

### Example 4: MPO Completion (Channel Partner Seller - Adding Markup)
```json
{
  "$schema": "https://schema.mp.microsoft.com/schema/configure/2022-07-01",
  "resources": [
    {
      "$schema": "https://schema.mp.microsoft.com/schema/private-offer-mpo-channel-partner/2023-07-15",
      "name": "MPOOriginatorOffer",  // Match originator's name
      "state": "live",
      "privateOfferType": "multipartyPromotionChannelPartner",
      "offerPricingType": "saasNewCustomizedPlans",
      "variableStartDate": true,
      "end": "2026-06-30",
      "acceptBy": "2025-06-30",
      "preparedBy": "seller@partner.com",
      "originatorTermsAndConditionsDocs": [  // Read-only from ISV
        {
          "sasUrl": "https://blob.storage.azure.com/terms/isv-terms.pdf?sas-token",
          "fileName": "isv-terms.pdf",
          "customerFacingDocumentName": "ISV Terms"
        }
      ],
      "termsAndConditionsDocs": [
        {
          "sasUrl": "https://blob.storage.azure.com/terms/partner-terms.pdf?sas-token",
          "fileName": "partner-terms.pdf",
          "customerFacingDocumentName": "Partner Terms"
        }
      ],
      "notificationContacts": ["seller@partner.com"],
      "beneficiaries": [  // Read-only
        {
          "id": "ac357579-e860-54a6-80b3-66958aea67fe:7471d04e-f696-4d20-af34-fa78d51e419c_2019-05-31",
          "description": "End Customer"
        }
      ],
      "partners": [  // Read-only
        {
          "id": "channel-partner-id",
          "partnerName": "Channel Partner Inc.",
          "location": "United States"
        }
      ],
      "originatorPricing": [  // Read-only from ISV
        {
          "product": "product/7ba807c8-386a-4efe-80f1-b97bf8a554f8",
          "discountType": "absolute",
          "priceDetails": { /* ... */ },
          "basePlan": "plan/standard",
          "newPlanDetails": { /* ... */ }
        }
      ],
      "markupPercentage": 2.5,
      "notes": "Added 2.5% markup"
    }
  ]
}
```

### Example 5: VMSR Private Offer (Absolute Pricing)
For VMSR, use plan IDs that support reservations; pricing is absolute.
```json
{
  "$schema": "https://schema.mp.microsoft.com/schema/configure/2022-07-01",
  "resources": [
    {
      "$schema": "https://schema.mp.microsoft.com/schema/private-offer/2023-07-15",
      "name": "VMSROffer",
      "privateOfferType": "customerPromotion",
      "state": "live",
      "offerPricingType": "vmSoftwareReservations",
      "variableStartDate": false,
      "start": "2025-01-01",
      "end": "2027-12-31",
      "acceptBy": "2024-12-31",
      "notificationContacts": ["vm@contoso.com"],
      "beneficiaries": [
        {
          "id": "ac357579-e860-54a6-80b3-66958aea67fe:7471d04e-f696-4d20-af34-fa78d51e419c_2019-05-31",
          "description": "Customer"
        }
      ],
      "pricing": [
        {
          "product": "product/vm-product-id",
          "discountType": "absolute",
          "priceDetails": {
            "resourceName": "vmReservationPricing",
            "$schema": "https://schema.mp.microsoft.com/schema/price-and-availability-private-offer-plan/2023-07-15",
            "softwareReservation": {
              "duration": "3Year",
              "paymentSchedule": "Monthly",
              "vmPrices": [
                {
                  "vmSize": "Standard_D2_v3",
                  "price": 1000.00,
                  "quantity": 5
                }
              ]
            }
          },
          "basePlan": "plan/reservation-plan"
        }
      ],
      "notes": "3-year VMSR with custom quantity"
    }
  ]
}
```

### Polling Job Status
After POST, response includes `jobId`. Poll: `GET https://graph.microsoft.com/rp/product-ingestion/configure/<jobId>/status?$version=2022-07-01`

Status values: `NotStarted`, `Running`, `Completed`. On success (`jobResult: Succeeded`), get `resourceURI` for offer details.

## Step 3: Retrieve Existing Private Offers

- **List All Private Offers**: `GET https://graph.microsoft.com/rp/product-ingestion/private-offer/query?$version=2023-07-15`
- **Get Specific Offer Details**: `GET https://graph.microsoft.com/rp/product-ingestion/private-offer/<id>?$version=2023-07-15` or `GET https://graph.microsoft.com/rp/product-ingestion/configure/<jobId>?$version=2023-07-15`

This lists all private offers, including MPOs.

## Step 4: Manage Existing Offers

Use the same configure endpoint for updates.

### Update an Offer
Set `state: live` and include changes in the payload (for withdrawn offers, etc.).

**Example Request Body (Update Pricing):**
```json
{
  "$schema": "https://schema.mp.microsoft.com/schema/configure/2022-07-01",
  "resources": [
    {
      "$schema": "https://schema.mp.microsoft.com/schema/private-offer/2023-07-15",
      "id": "private-offer/existing-id",
      "name": "ExistingOffer",
      "privateOfferType": "customerPromotion",
      "state": "live",
      // Updated fields...
      "pricing": [
        {
          "product": "product/id",
          "plan": "plan/id",
          "discountType": "percentage",
          "discountPercentage": 20
        }
      ]
    }
  ]
}
```

### Withdraw an Offer
Withdraw to remove customer access (pre-acceptance for MPOs).

**Example (ISV):**
```json
{
  "$schema": "https://schema.mp.microsoft.com/schema/configure/2022-07-01",
  "resources": [
    {
      "$schema": "https://schema.mp.microsoft.com/schema/private-offer/2023-07-15",
      "id": "private-offer/existing-id",
      "name": "OfferToWithdraw",
      "privateOfferType": "multipartyPromotionOriginator",
      "state": "withdrawn"
    }
  ]
}
```

For channel partners, use `multipartyPromotionChannelPartner`.

### Delete an Offer
Only for drafts.

**Example:**
```json
{
  "$schema": "https://schema.mp.microsoft.com/schema/configure/2022-07-01",
  "resources": [
    {
      "$schema": "https://schema.mp.microsoft.com/schema/private-offer/2023-07-15",
      "id": "private-offer/draft-id",
      "name": "DraftOffer",
      "privateOfferType": "customerPromotion",
      "state": "deleted"
    }
  ]
}
```

## Step 5: Customer Acceptance and Purchase

Once submitted (`live`), share the acceptance link (from response or dashboard). Customer process:
1. Check eligibility and roles using the eligibility report (applicable to MCA, EA, MOSP).
2. Accept in Azure portal > Marketplace > Private offers.
3. Purchase/subscribe (varies by type: SaaS configuration, VM deployment, etc.).

Track status in Partner Center dashboard or via API queries.

## Schemas

The Private Offer schema defines the structure for API requests. Below is the full JSON schema extracted from the provided documentation (composite; refer to subschemas for details).

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://schema.mp.microsoft.com/schema/private-offer/2023-07-15",
  "title": "Microsoft Product Ingestion Private Offer",
  "description": "Describes a Private Offer in the Microsoft Product Ingestion API",
  "allOf": [
    {
      "$ref": "https://schema.mp.microsoft.com/schema/resource/2022-07-01"
    },
    {
      "oneOf": [
        {
          "$ref": "https://schema.mp.microsoft.com/schema/private-offer-customer/2023-07-15"
        },
        {
          "$ref": "https://schema.mp.microsoft.com/schema/private-offer-csp/2023-07-15"
        },
        {
          "$ref": "https://schema.mp.microsoft.com/schema/private-offer-mpo-channel-partner/2023-07-15"
        },
        {
          "$ref": "https://schema.mp.microsoft.com/schema/private-offer-mpo-originator/2023-07-15"
        }
      ]
    },
    {
      "properties": {
        "offerPricingType": {
          "type": "string",
          "enum": [
            "editExistingOfferPricingOnly",
            "saasNewCustomizedPlans",
            "vmSoftwareReservations"
          ],
          "default": "editExistingOfferPricingOnly"
        }
      },
      "required": [
        "offerPricingType"
      ]
    },
    {
      "if": {
        "properties": {
          "offerPricingType": {
            "const": "vmSoftwareReservations"
          }
        }
      },
      "then": {
        "properties": {
          "pricing": {
            "items": {
              "properties": {
                "newPlanDetails": {
                  "not": {
                    "required": [
                      "name",
                      "description"
                    ]
                  }
                }
              }
            }
          }
        }
      }
    }
  ],
  "$defs": {
    "https://schema.mp.microsoft.com/schema/resource/2022-07-01": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "$id": "https://schema.mp.microsoft.com/schema/resource/2022-07-01",
      "$comment": "This schema is extended by a resource type schema. We allow additional properties to enable that",
      "type": "object",
      "properties": {
        "resourceName": {
          "type": "string",
          "minLength": 1,
          "maxLength": 50
        },
        "id": {
          "$ref": "https://schema.mp.microsoft.com/schema/durable-id/2022-07-01"
        },
        "validations": {
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/validation/2022-07-01"
          },
          "readonly": true
        }
      },
      "required": [
        "$schema"
      ],
      "additionalProperties": true
    },
    "https://schema.mp.microsoft.com/schema/private-offer-customer/2023-07-15": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "$id": "https://schema.mp.microsoft.com/schema/private-offer-customer/2023-07-15",
      "title": "Microsoft Product Ingestion Private Offer",
      "description": "Describes a Private Offer in the Microsoft Product Ingestion API",
      "type": "object",
      "properties": {
        "name": {
          "type": "string",
          "pattern": "^[^\\!\\\u003C\\\u003E\\^\\[\\]\\@\\#\\%\\/\\x00-\\x1F\\x7E-\\xFF]\u002B$",
          "maxLength": 128
        },
        "privateOfferType": {
          "type": "string",
          "const": "customerPromotion"
        },
        "upgradedFrom": {
          "$ref": "https://schema.mp.microsoft.com/schema/private-offer-promotion-reference/2022-07-01"
        },
        "variableStartDate": {
          "type": "boolean"
        },
        "start": {
          "type": "string",
          "format": "date"
        },
        "end": {
          "type": "string",
          "format": "date"
        },
        "acceptBy": {
          "type": "string",
          "format": "date"
        },
        "preparedBy": {
          "type": "string",
          "format": "email"
        },
        "notificationContacts": {
          "type": "array",
          "items": {
            "type": "string",
            "format": "email"
          }
        },
        "state": {
          "type": "string",
          "enum": [
            "draft",
            "live",
            "deleted"
          ]
        },
        "subState": {
          "type": "string",
          "enum": [
            "pendingAcceptance",
            "accepted"
          ]
        },
        "termsAndConditionsDocSasUrl": {
          "type": "string",
          "format": "uri"
        },
        "beneficiaries": {
          "type": "array",
          "maxItems": 1,
          "items": {
            "allOf": [
              {
                "$ref": "https://schema.mp.microsoft.com/schema/private-offer-beneficiary/2022-07-01"
              }
            ],
            "properties": {
              "beneficiaryRecipients": {
                "items": {
                  "properties": {
                    "recipientType": {
                      "const": "billingGroup"
                    }
                  }
                }
              }
            }
          }
        },
        "pricing": {
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/private-offer-pricing/2023-07-15",
            "not": {
              "required": [
                "markupPercentage"
              ]
            }
          }
        },
        "lastModified": {
          "type": "string",
          "format": "date"
        },
        "acceptanceLinks": {
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/private-offer-acceptance-link/2022-07-01"
          }
        },
        "eTag": {
          "type": "string"
        }
      },
      "required": [
        "name",
        "state"
      ],
      "allOf": [
        {
          "if": {
            "not": {
              "properties": {
                "start": {
                  "const": null
                }
              }
            }
          },
          "then": {
            "properties": {
              "variableStartDate": {
                "not": {
                  "const": true
                }
              }
            }
          }
        },
        {
          "if": {
            "not": {
              "properties": {
                "variableStartDate": {
                  "const": true
                }
              }
            }
          },
          "then": {
            "required": [
              "start"
            ]
          }
        },
        {
          "if": {
            "properties": {
              "state": {
                "const": "live"
              }
            }
          },
          "then": {
            "properties": {
              "beneficiaries": {
                "minItems": 1
              },
              "pricing": {
                "minItems": 1
              }
            },
            "required": [
              "end",
              "acceptBy",
              "beneficiaries",
              "pricing"
            ]
          }
        }
      ]
    },
    "https://schema.mp.microsoft.com/schema/private-offer-csp/2023-07-15": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "$id": "https://schema.mp.microsoft.com/schema/private-offer-csp/2023-07-15",
      "title": "Microsoft Product Ingestion Private Offer",
      "description": "Describes a Private Offer in the Microsoft Product Ingestion API",
      "type": "object",
      "properties": {
        "name": {
          "type": "string",
          "pattern": "^[^\\!\\\u003C\\\u003E\\^\\[\\]\\@\\#\\%\\/\\x00-\\x1F\\x7E-\\xFF]\u002B$",
          "maxLength": 128
        },
        "privateOfferType": {
          "type": "string",
          "const": "cspPromotion"
        },
        "upgradedFrom": {
          "$ref": "https://schema.mp.microsoft.com/schema/private-offer-promotion-reference/2022-07-01"
        },
        "variableStartDate": {
          "type": "boolean"
        },
        "start": {
          "type": "string",
          "format": "date"
        },
        "end": {
          "type": "string",
          "format": "date"
        },
        "preparedBy": {
          "type": "string",
          "format": "email"
        },
        "notificationContacts": {
          "type": "array",
          "items": {
            "type": "string",
            "format": "email"
          }
        },
        "state": {
          "type": "string",
          "enum": [
            "draft",
            "live",
            "withdrawn",
            "deleted"
          ]
        },
        "beneficiaries": {
          "type": "array",
          "maxItems": 150,
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/private-offer-beneficiary/2022-07-01",
            "properties": {
              "beneficiaryRecipients": {
                "items": {
                  "properties": {
                    "recipientType": {
                      "const": "cspCustomer"
                    }
                  }
                }
              }
            }
          }
        },
        "pricing": {
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/private-offer-pricing/2023-07-15",
            "not": {
              "required": [
                "markupPercentage"
              ]
            }
          }
        },
        "lastModified": {
          "type": "string",
          "format": "date"
        },
        "eTag": {
          "type": "string"
        }
      },
      "required": [
        "name",
        "state"
      ],
      "allOf": [
        {
          "not": {
            "required": [
              "termsAndConditionsDocSasUrl"
            ]
          }
        },
        {
          "not": {
            "required": [
              "acceptBy"
            ]
          }
        },
        {
          "if": {
            "not": {
              "properties": {
                "start": {
                  "const": null
                }
              }
            }
          },
          "then": {
            "properties": {
              "variableStartDate": {
                "not": {
                  "const": true
                }
              }
            }
          }
        },
        {
          "if": {
            "not": {
              "properties": {
                "variableStartDate": {
                  "const": true
                }
              }
            }
          },
          "then": {
            "required": [
              "start"
            ]
          }
        },
        {
          "if": {
            "properties": {
              "state": {
                "const": "live"
              }
            }
          },
          "then": {
            "properties": {
              "beneficiaries": {
                "minItems": 1
              },
              "pricing": {
                "minItems": 1
              }
            },
            "required": [
              "end",
              "beneficiaries",
              "pricing"
            ]
          }
        }
      ]
    },
    "https://schema.mp.microsoft.com/schema/private-offer-mpo-channel-partner/2023-07-15": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "$id": "https://schema.mp.microsoft.com/schema/private-offer-mpo-channel-partner/2023-07-15",
      "title": "Microsoft Product Ingestion MultiParty Private Offer",
      "description": "Describes a MultiParty Private Offer in the Microsoft Product Ingestion API",
      "type": "object",
      "properties": {
        "name": {
          "type": "string",
          "pattern": "^[\\x20-\\x7F]\u002B[^\\!\\\u003E\\^\\[\\\u003E\\!\\@\\#\\%\\/\\]]$",
          "maxLength": 128
        },
        "privateOfferType": {
          "type": "string",
          "const": "multipartyPromotionChannelPartner"
        },
        "variableStartDate": {
          "type": "boolean"
        },
        "start": {
          "type": "string",
          "format": "date"
        },
        "end": {
          "type": "string",
          "format": "date"
        },
        "acceptBy": {
          "type": "string",
          "format": "date"
        },
        "notificationContacts": {
          "type": "array",
          "maxItems": 5,
          "items": {
            "type": "string",
            "format": "email"
          }
        },
        "preparedBy": {
          "type": "string",
          "format": "email"
        },
        "state": {
          "type": "string",
          "enum": [
            "draft",
            "live"
          ]
        },
        "subState": {
          "type": "string",
          "enum": [
            "pendingAcceptance",
            "accepted"
          ]
        },
        "originatorTermsAndConditionsDocs": {
          "description": "The combined size of originatorTermsAndConditionsDocs and termsAndConditionsDocs cannot exceed 5. Objects",
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/terms-and-conditions-document/2022-07-01"
          },
          "maxItems": 5
        },
        "termsAndConditionsDocs": {
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/terms-and-conditions-document/2022-07-01"
          },
          "maxItems": 5
        },
        "beneficiaries": {
          "type": "array",
          "maxItems": 1,
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/private-offer-beneficiary/2022-07-01"
          }
        },
        "partners": {
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/private-offer-partner/2022-07-01"
          }
        },
        "originatorPricing": {
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/private-offer-pricing/2023-07-15"
          }
        },
        "markupPercentage": {
          "type": "number",
          "minimum": 0
        },
        "notes": {
          "type": "string",
          "maxLength": 60
        },
        "lastModified": {
          "type": "string",
          "format": "date"
        },
        "acceptanceLinks": {
          "type": "array",
          "items": {
            "$ref": "https://schema.mp.microsoft.com/schema/private-offer-acceptance-link/2022-07-01"
          }
        },
        "eTag": {
          "type": "string"
        }
      },
      "required": [
        "name",
        "state"
      ]
    }
  }
}
```

For full subschemas (e.g., pricing, beneficiaries), refer to the linked $ref URLs in the schema or Microsoft docs.

## Troubleshooting

Common issues and tips:
- **401 Unauthorized**: Invalid/expired token. Refresh using client credentials.
- **400 Bad Request**: Schema validation failed. Check required fields, patterns (e.g., name max 128 chars, no special chars), and enums. Use tools like JSON Schema validators.
- **Error Parsing**: Responses include error details like "not found" or "validation errors."
- **Job Failures**: Poll jobId; check "errors" array in status response.
- **MPO Issues**: Ensure ISV submits first; channel partner can't edit ISV fields. Withdraw to edit.
- **Limitations**: Max 10 products/plans per offer; max 5 T&C docs; no updates to withdrawn offers in draft state—resubmit as live.
- **Resources**: Use schemas like `https://schema.mp.microsoft.com/schema/private-offer/2023-07-15`. For support, open a ticket in Partner Center with offer ID.

If issues persist, check Partner Center dashboard for sub-statuses (e.g., pending authorization).

## Best Practices
- Start with drafts (`state: draft`) for testing.
- Use variableStartDate for flexibility.
- Monitor payouts and analytics in Partner Center > Marketplace Insights.
- For renewals, set `customerContractRenewal: true` for reduced fees.
- Automate with scripts: Handle token refresh, job polling (up to 20 minutes for completion), and error retries.
- Comply with market restrictions (e.g., US/UK/Canada for MPOs).

This guide covers the full lifecycle. For UI alternatives, use Partner Center dashboard, but API enables automation for large-scale management.