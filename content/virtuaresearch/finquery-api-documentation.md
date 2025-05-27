---
date: 2025-05-09 16:18:36+05:30
title: Finquery API Documentation
---

Base URLs:

* <a href="http://localhost:8000">Develop Env: http://localhost:8000</a>

# Authentication

* API Key (apikey-header-X-Secret-Key)
    - Parameter Name: **X-Secret-Key**, in: header. 

<a id="opIdreportsWebhook"></a>
## POST Reports Webhook

POST /dataset/reports-webhook/{client_id}

Webhook to call when the reports of a specific client are updated

### Params

|Name|Location|Type|Required|Description|
|---|---|---|---|---|
|client_id|path|string| yes |Client id to update reports for|
|X-Secret-Key|header|string| yes |none|

> Response Examples

> 200 Response

```json
{
  "message": "string",
  "total_reports": 0,
  "deleted_old_reports": 0
}
```

### Responses

|HTTP Status Code |Meaning|Description|Data schema|
|---|---|---|---|
|200|[OK](https://tools.ietf.org/html/rfc7231#section-6.3.1)|none|Inline|
|404|[Not Found](https://tools.ietf.org/html/rfc7231#section-6.5.4)|none|Inline|
|500|[Internal Server Error](https://tools.ietf.org/html/rfc7231#section-6.6.1)|none|Inline|

### Responses Data Schema

HTTP Status Code **200**

|Name|Type|Required|Restrictions|Title|description|
|---|---|---|---|---|---|
|» message|string|true|none||none|
|» total_reports|integer|true|none||none|
|» deleted_old_reports|integer|true|none||none|

HTTP Status Code **404**

|Name|Type|Required|Restrictions|Title|description|
|---|---|---|---|---|---|
|» message|string|true|none||none|

HTTP Status Code **500**

|Name|Type|Required|Restrictions|Title|description|
|---|---|---|---|---|---|
|» message|string|true|none||none|