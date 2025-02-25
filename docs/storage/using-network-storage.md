---
title: Using network storage with chibisafe
summary: How network storage works and setting it up with chibisafe
---

# Using network storage with chibisafe

chibisafe supports adding an S3-compatible Object Storage service to use as the destination for your uploads, meaning you can now offload all your uploads to any S3-compatible storage of choice like AWS S3, Backblaze, Wasabi, etc.

:::tip
  Enabling network storage will prevent using tools like ShareX or uploading directly to the `/api/upload` endpoint as the normal uploading flow changes by getting a signed URL from the network provider.
:::

Settings for each provider might differ slightly but the main idea is that providing the following options, users can upload files directly to the storage:

<img
  src="https://chibisafe.app/j1MSha2TmBBz.png"
  width="1144"
  height="811"
  alt="Object storage settings"
/>

> - Region
> - Bucket name
> - Access key
> - Access secret
> - Endpoint
> - Public URL

These options are common across services that use the same S3 api that AWS provides, so a bit of tinkering of the Endpoint and Public URL should get your files going nicely.

### Bucket policy
You should modify accordingly, but a good starter point for your bucket policy is as follows. You can find the setting in your Bucket -> Permissions -> Bucket Policy.
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AddPerm",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::chibisafe-test/*"
        }
    ]
}
```

### Generic cross-origin resource sharing (CORS)
As a starter point you can use these settings. Locking it down further is up to the user and outside of the chibisafe scope, so take this just as a starter point that needs tighter security. Bucket -> Permissions -> Cross-origin resource sharing (CORS)

```json
[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "PUT"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": []
    }
]
```

### Blackblaze B2: Cross-origin resource sharing (CORS)
Backblaze B2 uses a different rule structure than most S3 providers. It is recommended to use [B2 CLI](https://www.backblaze.com/docs/cloud-storage-command-line-tools) to make changes to your bucket. [Read more](https://www.backblaze.com/docs/cloud-storage-cross-origin-resource-sharing-rules)

```json
[
    {
    	"corsRuleName": "chibisafe",
     	"allowedHeaders": [
            "*"
     	],
     	"allowedOperations": [
            "s3_put"
     	],
     	"allowedOrigins": [
            "*"
     	],
        "exposeHeaders": [],
    	"maxAgeSeconds": 3600
    }
]
```
Run this command to make changes to bucket CORS policy:

```bash
b2 bucket update bucketName allPublic --cors-rules "$(<./cors.json)"
```

### Closing

This should get you up and running but remember, certain options might change depending the provider you use.
