# n8n Workflow Configuration Guide

Panduan lengkap untuk konfigurasi workflow n8n dalam sistem Affiliate Marketing Automation.

## 🏗️ Arsitektur Workflow n8n

```
n8n (Orchestration)
    │
    ├─→ Cron Trigger (Schedule)
    ├─→ API Trigger (Manual/Webhook)
    └─→ Event Trigger (Database changes)
         │
         ├─→ Product Fetch Workflow
         ├─→ Content Generation Workflow
         ├─→ Video Rendering Workflow
         ├─→ Upload Workflow
         └─→ Analytics Workflow
```

## 📋 Workflow Utama

### 1. Product Fetch Workflow

Workflow ini mengambil produk dari platform afiliasi (Shopee, TikTok Shop, Tokopedia) secara berkala.

#### Workflow Configuration

```json
{
  "name": "Product Fetch - Affiliate",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "hours",
              "hoursInterval": 2
            }
          ]
        }
      },
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "method": "GET",
        "url": "http://api:8000/api/v1/products/fetch",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {}
      },
      "name": "Call API - Fetch Products",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [450, 300],
      "credentials": {
        "httpHeaderAuth": {
          "id": "1",
          "name": "API Auth"
        }
      }
    },
    {
      "parameters": {
        "operation": "insert",
        "document": {
          "__rl": true,
          "value": "={{ $json }}",
          "mode": "json"
        },
        "collection": "products"
      },
      "name": "Save to Database",
      "type": "n8n-nodes-base.mongoDb",
      "typeVersion": 1,
      "position": [650, 300]
    },
    {
      "parameters": {
        "channel": "product_fetch",
        "message": "=Fetched {{ $json.length }} products from affiliate API"
      },
      "name": "Send Notification",
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1,
      "position": [850, 300]
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [
        [
          {
            "node": "Call API - Fetch Products",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Call API - Fetch Products": {
      "main": [
        [
          {
            "node": "Save to Database",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Save to Database": {
      "main": [
        [
          {
            "node": "Send Notification",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

#### Setup Steps

1. **Create Schedule Trigger**
   - Buka n8n editor
   - Add node → Schedule Trigger
   - Set interval: Every 2 hours
   - Cron expression: `0 */2 * * *`

2. **Configure API Call**
   - Add HTTP Request node
   - Method: GET
   - URL: `http://api:8000/api/v1/products/fetch`
   - Authentication: Header Auth
   - Add header: `Authorization: Bearer YOUR_JWT_TOKEN`

3. **Save to Database**
   - Add PostgreSQL node
   - Operation: Insert
   - Table: products
   - Map API response to database columns

4. **Send Notification**
   - Add Telegram node
   - Configure Telegram credentials
   - Channel: product_fetch
   - Message: Dynamic with product count

### 2. Content Generation Workflow

Workflow ini memicu AI content generation untuk produk baru.

#### Workflow Configuration

```json
{
  "name": "AI Content Generation",
  "nodes": [
    {
      "parameters": {
        "pollTimes": {
          "item": [
            {
              "mode": "everyMinute"
            }
          ]
        }
      },
      "name": "PostgreSQL Trigger",
      "type": "n8n-nodes-base.postgresTrigger",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict"
          },
          "conditions": [
            {
              "id": "condition-1",
              "leftValue": "={{ $json.status }}",
              "rightValue": "pending",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        }
      },
      "name": "Filter Pending Products",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [450, 300]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://api:8000/api/v1/products/{{ $json.id }}/generate-content",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {}
      },
      "name": "Call API - Generate Content",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [650, 200]
    },
    {
      "parameters": {
        "operation": "update",
        "table": "products",
        "column": "id",
        "value": "={{ $json.id }}",
        "data": "={{ { status: 'generating' } }}"
      },
      "name": "Update Status - Generating",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2,
      "position": [650, 400]
    }
  ],
  "connections": {
    "PostgreSQL Trigger": {
      "main": [
        [
          {
            "node": "Filter Pending Products",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Filter Pending Products": {
      "main": [
        [
          {
            "node": "Call API - Generate Content",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Update Status - Generating",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

#### Setup Steps

1. **Create Database Trigger**
   - Add PostgreSQL Trigger node
   - Table: products
   - Poll interval: Every minute
   - Filter: status = 'pending'

2. **Filter Products**
   - Add IF node
   - Condition: status equals 'pending'
   - True path: Continue to generation
   - False path: Skip

3. **Call API for Generation**
   - Add HTTP Request node
   - Method: POST
   - URL: `http://api:8000/api/v1/products/{id}/generate-content`
   - This triggers the AI worker

4. **Update Status**
   - Add PostgreSQL node
   - Operation: Update
   - Update status to 'generating'

### 3. Video Rendering Workflow

Workflow ini memicu video rendering setelah content generation selesai.

#### Workflow Configuration

```json
{
  "name": "Video Rendering",
  "nodes": [
    {
      "parameters": {
        "pollTimes": {
          "item": [
            {
              "mode": "everyMinute"
            }
          ]
        }
      },
      "name": "PostgreSQL Trigger",
      "type": "n8n-nodes-base.postgresTrigger",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true
          },
          "conditions": [
            {
              "id": "condition-1",
              "leftValue": "={{ $json.content_status }}",
              "rightValue": "completed",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            },
            {
              "id": "condition-2",
              "leftValue": "={{ $json.video_status }}",
              "rightValue": "pending",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        }
      },
      "name": "Filter Ready for Render",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [450, 300]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://api:8000/api/v1/videos/create",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ { product_id: $json.id, script: $json.script, caption: $json.caption, hashtags: $json.hashtags } }}",
        "options": {}
      },
      "name": "Create Video Record",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [650, 300]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://api:8000/api/v1/videos/{{ $json.id }}/render",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "options": {}
      },
      "name": "Trigger Rendering",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [850, 300]
    }
  ],
  "connections": {
    "PostgreSQL Trigger": {
      "main": [
        [
          {
            "node": "Filter Ready for Render",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Filter Ready for Render": {
      "main": [
        [
          {
            "node": "Create Video Record",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Create Video Record": {
      "main": [
        [
          {
            "node": "Trigger Rendering",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

#### Setup Steps

1. **Create Database Trigger**
   - Add PostgreSQL Trigger node
   - Table: products
   - Poll interval: Every minute
   - Filter: content_status = 'completed' AND video_status = 'pending'

2. **Filter Products**
   - Add IF node
   - Check if content is ready and video is pending

3. **Create Video Record**
   - Add HTTP Request node
   - Method: POST
   - URL: `http://api:8000/api/v1/videos/create`
   - Body: Product data with generated content

4. **Trigger Rendering**
   - Add HTTP Request node
   - Method: POST
   - URL: `http://api:8000/api/v1/videos/{id}/render`
   - This triggers the render worker

### 4. Upload Workflow

Workflow ini mengupload video ke platform sosial media setelah rendering selesai.

#### Workflow Configuration

```json
{
  "name": "Social Media Upload",
  "nodes": [
    {
      "parameters": {
        "pollTimes": {
          "item": [
            {
              "mode": "everyMinute"
            }
          ]
        }
      },
      "name": "PostgreSQL Trigger",
      "type": "n8n-nodes-base.postgresTrigger",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true
          },
          "conditions": [
            {
              "id": "condition-1",
              "leftValue": "={{ $json.render_status }}",
              "rightValue": "completed",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            },
            {
              "id": "condition-2",
              "leftValue": "={{ $json.upload_status }}",
              "rightValue": "pending",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        }
      },
      "name": "Filter Ready for Upload",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [450, 300]
    },
    {
      "parameters": {
        "platform": "tiktok"
      },
      "name": "Split by Platform",
      "type": "n8n-nodes-base.splitOut",
      "typeVersion": 1,
      "position": [650, 300]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://api:8000/api/v1/videos/{{ $json.id }}/upload",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ { platform: 'tiktok' } }}",
        "options": {}
      },
      "name": "Upload to TikTok",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [850, 200]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://api:8000/api/v1/videos/{{ $json.id }}/upload",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ { platform: 'instagram' } }}",
        "options": {}
      },
      "name": "Upload to Instagram",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [850, 400]
    }
  ],
  "connections": {
    "PostgreSQL Trigger": {
      "main": [
        [
          {
            "node": "Filter Ready for Upload",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Filter Ready for Upload": {
      "main": [
        [
          {
            "node": "Split by Platform",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Split by Platform": {
      "main": [
        [
          {
            "node": "Upload to TikTok",
            "type": "main",
            "index": 0
          },
          {
            "node": "Upload to Instagram",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

#### Setup Steps

1. **Create Database Trigger**
   - Add PostgreSQL Trigger node
   - Table: videos
   - Poll interval: Every minute
   - Filter: render_status = 'completed' AND upload_status = 'pending'

2. **Filter Videos**
   - Add IF node
   - Check if video is ready for upload

3. **Split by Platform**
   - Add Split Out node
   - Create separate branches for each platform

4. **Upload to Platforms**
   - Add HTTP Request nodes for each platform
   - TikTok: `http://api:8000/api/v1/videos/{id}/upload`
   - Instagram: Same endpoint with platform parameter
   - Facebook: Same endpoint with platform parameter

### 5. Analytics Workflow

Workflow ini mengambil analytics data dari platform sosial media.

#### Workflow Configuration

```json
{
  "name": "Analytics Collection",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "hours",
              "hoursInterval": 6
            }
          ]
        }
      },
      "name": "Schedule Trigger",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "operation": "executeQuery",
        "query": "SELECT id, platform, video_url FROM videos WHERE upload_status = 'completed' AND uploaded_at > NOW() - INTERVAL '7 days'"
      },
      "name": "Get Recent Videos",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2,
      "position": [450, 300]
    },
    {
      "parameters": {
        "platform": "={{ $json.platform }}"
      },
      "name": "Split by Platform",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 2,
      "position": [650, 300]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://api:8000/api/v1/analytics/fetch",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ { video_id: $json.id, platform: $json.platform, video_url: $json.video_url } }}",
        "options": {}
      },
      "name": "Fetch TikTok Analytics",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [850, 200]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "http://api:8000/api/v1/analytics/fetch",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ { video_id: $json.id, platform: $json.platform, video_url: $json.video_url } }}",
        "options": {}
      },
      "name": "Fetch Instagram Analytics",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [850, 400]
    }
  ],
  "connections": {
    "Schedule Trigger": {
      "main": [
        [
          {
            "node": "Get Recent Videos",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get Recent Videos": {
      "main": [
        [
          {
            "node": "Split by Platform",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Split by Platform": {
      "main": [
        [
          {
            "node": "Fetch TikTok Analytics",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Fetch Instagram Analytics",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

#### Setup Steps

1. **Create Schedule Trigger**
   - Add Schedule Trigger node
   - Interval: Every 6 hours
   - Cron expression: `0 */6 * * *`

2. **Get Recent Videos**
   - Add PostgreSQL node
   - Operation: Execute Query
   - Query: Select videos uploaded in last 7 days

3. **Split by Platform**
   - Add Switch node
   - Route based on platform field

4. **Fetch Analytics**
   - Add HTTP Request nodes for each platform
   - Call analytics API endpoint
   - This triggers the analytics worker

## 🔧 Setup Credentials di n8n

### PostgreSQL Credentials

1. Buka n8n → Credentials
2. Add new credential → PostgreSQL
3. Configure:
   - Host: postgres
   - Database: affiliate_db
   - User: affiliate_user
   - Password: (dari .env)
   - Port: 5432

### HTTP Header Auth Credentials

1. Buka n8n → Credentials
2. Add new credential → Header Auth
3. Configure:
   - Name: Authorization
   - Value: Bearer YOUR_JWT_TOKEN

### Telegram Credentials

1. Buka n8n → Credentials
2. Add new credential → Telegram API
3. Configure:
   - Access Token: (dari .env)
   - Chat ID: (dari .env)

### Affiliate API Credentials

Setup credentials untuk setiap platform:

**Shopee:**
- API Key
- Partner ID
- Secret Key

**TikTok Shop:**
- API Key
- Partner ID
- Secret Key

**Tokopedia:**
- Client ID
- Client Secret

## 🎯 Best Practices untuk n8n Workflows

### 1. Error Handling

Tambahkan error handling di setiap workflow:

```json
{
  "parameters": {
    "errorOutput": "error"
  },
  "name": "Error Handler",
  "type": "n8n-nodes-base.errorTrigger",
  "typeVersion": 1,
  "position": [1050, 300]
}
```

### 2. Retry Logic

Gunakan node "Wait" dan "IF" untuk retry logic:

```json
{
  "parameters": {
    "amount": 5,
    "unit": "minutes"
  },
  "name": "Wait Before Retry",
  "type": "n8n-nodes-base.wait",
  "typeVersion": 1.1,
  "position": [750, 300]
}
```

### 3. Logging

Tambahkan logging node untuk tracking:

```json
{
  "parameters": {
    "operation": "append",
    "documentId": {
      "__rl": true,
      "value": "={{ $json.id }}",
      "mode": "id"
    },
    "data": "={{ { timestamp: $now, status: $json.status, message: 'Processing' } }}"
  },
  "name": "Log to Database",
  "type": "n8n-nodes-base.postgres",
  "typeVersion": 2,
  "position": [950, 300]
}
```

### 4. Rate Limiting

Gunakan node "Wait" untuk rate limiting:

```json
{
  "parameters": {
    "amount": 1,
    "unit": "seconds"
  },
  "name": "Rate Limit",
  "type": "n8n-nodes-base.wait",
  "typeVersion": 1.1,
  "position": [550, 300]
}
```

## 📊 Monitoring Workflows

### View Workflow Execution

1. Buka n8n → Executions
2. Filter by workflow name
3. View execution history
4. Check for failed executions

### Setup Workflow Alerts

1. Buka workflow settings
2. Enable "Error Workflow"
3. Configure error notifications
4. Set up Telegram/Slack alerts

### Performance Optimization

- Use batch processing for large datasets
- Implement pagination for API calls
- Cache frequently used data
- Use async processing where possible

## 🔄 Import/Export Workflows

### Export Workflow

1. Buka workflow di n8n editor
2. Click menu (⋮) → Download
3. Save as JSON file

### Import Workflow

1. Buka n8n → Workflows
2. Click "Import from File"
3. Select uploaded JSON file
4. Review and activate

## 🧪 Testing Workflows

### Manual Trigger

1. Buka workflow
2. Click "Execute Workflow"
3. Check execution results
4. Review data flow

### Test with Sample Data

```json
{
  "id": 1,
  "name": "Test Product",
  "price": "Rp 99.000",
  "status": "pending"
}
```

### Debug Mode

1. Enable debug mode in workflow
2. Add "Set" node to inspect data
3. Use "Code" node for transformations
4. Check node outputs

## 📚 Additional Resources

- [n8n Documentation](https://docs.n8n.io/)
- [n8n Workflow Examples](https://n8n.io/workflows/)
- [n8n Community Forum](https://community.n8n.io/)
