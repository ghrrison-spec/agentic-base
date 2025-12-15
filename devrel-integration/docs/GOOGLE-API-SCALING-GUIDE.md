# Google API Scaling Guide

This document describes the scaling architecture and quota optimization strategies for Google Drive and Docs API integrations.

## Quota Reference

### Google Drive API
| Quota Type | Per User | Per Project |
|------------|----------|-------------|
| Default    | 12,000/min | 12,000/min |

### Google Docs API
| Quota Type | Per User | Per Project |
|------------|----------|-------------|
| Read requests | **300/min** | 3,000/min |
| Write requests | **60/min** | 600/min |

**Key Insight**: Docs API per-user quotas are 40x stricter than Drive API!

## Current Optimizations (Implemented)

### 1. Drive Export API Instead of Docs API
**File**: `src/services/google-docs-monitor.ts`

Instead of using `docs.documents.get()` (300/min quota), we use `drive.files.export()` (12,000/min quota):

```typescript
// BEFORE: Uses Docs API (300/min per user)
const response = await this.docs.documents.get({ documentId });

// AFTER: Uses Drive API (12,000/min per user)
const response = await this.drive.files.export({
  fileId: file.id,
  mimeType: 'text/plain'
});
```

**Trade-off**: Plain text loses rich formatting, but this is acceptable for digest generation.

### 2. Separate Rate Limiters for Drive vs Docs
**File**: `src/services/api-rate-limiter.ts`

| API | Limit Set | Actual Quota | Headroom |
|-----|-----------|--------------|----------|
| `google-drive` | 500/min | 12,000/min | 96% |
| `google-docs-read` | 250/min | 300/min | 17% |
| `google-docs-write` | 50/min | 60/min | 17% |

### 3. Document Content Caching
**File**: `src/services/document-cache.ts`

Redis-based caching with configurable TTL:
- Document content: 15 min TTL
- Document metadata: 5 min TTL
- Change tokens: Persistent

```typescript
// Check cache first
const cached = await documentCache.get(documentId);
if (cached) return cached;

// Fetch from API and cache
const content = await fetchFromAPI(documentId);
await documentCache.set({ id: documentId, content, ... });
```

### 4. Drive Changes API for Incremental Polling
**File**: `src/services/drive-changes-monitor.ts`

Instead of scanning all folders on every run, use Changes API:

```
BEFORE: Scan all folders → O(folders × documents) API calls
AFTER:  Get changes since last run → O(changed documents) API calls
```

For a typical run with 100 documents and 5 changes:
- Old method: ~105 API calls
- New method: ~6 API calls (**94% reduction**)

## Future Optimizations (Roadmap)

### 5. Google Drive Push Notifications (Webhooks)

**Status**: Not yet implemented

Instead of polling for changes, let Google push notifications to your server.

#### Setup Steps

1. **Create a webhook endpoint**:
```typescript
// POST /webhooks/google-drive
app.post('/webhooks/google-drive', (req, res) => {
  const resourceState = req.headers['x-goog-resource-state'];
  const channelId = req.headers['x-goog-channel-id'];

  if (resourceState === 'change') {
    // Fetch changed document
    const fileId = req.body.fileId;
    await processChangedDocument(fileId);
  }

  res.status(200).send('OK');
});
```

2. **Register watch on files/folders**:
```typescript
const response = await drive.files.watch({
  fileId: folderId,
  requestBody: {
    id: uuid(),  // Unique channel ID
    type: 'web_hook',
    address: 'https://your-domain.com/webhooks/google-drive',
    expiration: Date.now() + 7 * 24 * 60 * 60 * 1000, // 7 days max
  }
});
```

3. **Renew watch before expiration** (max 7 days):
```typescript
// Cron job: every 6 days
await renewDriveWatch(folderId);
```

#### Requirements
- HTTPS endpoint with valid SSL certificate
- Publicly accessible URL (no localhost)
- Domain verification in Google Cloud Console

#### Quota Impact
- **Polling**: O(documents) calls per interval
- **Push**: O(1) call per change (near real-time!)

### 6. Batch Requests

For operations on multiple files, use batch requests:

```typescript
// Instead of N individual requests:
for (const fileId of fileIds) {
  await drive.files.get({ fileId });
}

// Use 1 batch request:
const batch = drive.newBatch();
fileIds.forEach(fileId => {
  batch.add(drive.files.get({ fileId }));
});
await batch.execute();
```

**Quota Impact**: Batch counts as 1 request (up to 100 operations per batch).

### 7. Exponential Backoff with Jitter

Already implemented in `api-rate-limiter.ts`, but can be enhanced:

```typescript
const backoffMs = Math.min(
  initialBackoff * Math.pow(2, retries) + Math.random() * 1000, // Add jitter
  maxBackoff
);
```

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    SCALABLE ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │ Google Drive │────►│   Webhook    │────►│ Redis Queue  │    │
│  │  (Push)      │     │   Endpoint   │     │              │    │
│  └──────────────┘     └──────────────┘     └──────┬───────┘    │
│                                                    │            │
│                                                    ▼            │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │ Changes API  │────►│    Worker    │────►│ Redis Cache  │    │
│  │  (Fallback)  │     │ (Rate-limited)│     │              │    │
│  └──────────────┘     └──────────────┘     └──────┬───────┘    │
│                                                    │            │
│                              ┌─────────────────────┘            │
│                              ▼                                  │
│                       ┌──────────────┐                         │
│                       │   Processor  │                         │
│                       │ (Transforms) │                         │
│                       └──────────────┘                         │
│                                                                  │
│  API Call Flow:                                                 │
│  1. Webhook receives change → 0 API calls                       │
│  2. Check cache → 0 API calls if hit                           │
│  3. Fetch content via Drive Export → 1 API call (Drive quota)  │
│  4. Cache result → 0 API calls                                  │
│                                                                  │
│  Total: 0-1 API calls per document change                       │
│  (vs 2-3 calls in naive implementation)                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Monitoring & Alerts

### Metrics to Track
1. **API Quota Usage**: Percentage of quota consumed
2. **Cache Hit Rate**: Target > 80%
3. **Rate Limit Hits**: Should be near zero
4. **API Errors**: Track 429s and other errors

### Alert Thresholds
| Metric | Warning | Critical |
|--------|---------|----------|
| Quota Usage | 70% | 90% |
| Cache Hit Rate | < 60% | < 40% |
| Rate Limits/hour | > 5 | > 20 |
| API Errors/hour | > 10 | > 50 |

### Dashboard Queries

```typescript
// Get current quota status
const stats = await apiRateLimiter.getStatistics();
console.log(`Drive API: ${stats.apiConfigs['google-drive'].maxRequestsPerMinute}/min`);

// Get cache performance
const cacheStats = await documentCache.getStats();
console.log(`Cache hit rate: ${cacheStats.hitRate}%`);
```

## Requesting Quota Increases

If you need higher quotas:

1. Go to Google Cloud Console → APIs & Services → Google Docs API
2. Click "Quotas" tab
3. Click "Edit Quotas" for the limit you need increased
4. Fill out the form with:
   - Current usage
   - Expected usage
   - Business justification
5. Google typically responds within 2-3 business days

**Tip**: Before requesting increases, ensure you've implemented all the optimizations above. Google may ask what you've done to reduce API calls.

## Troubleshooting

### "Quota exceeded" errors
1. Check which API is hitting the limit (Drive vs Docs)
2. Review rate limiter statistics
3. Check cache hit rate
4. Consider reducing batch sizes

### Slow document fetches
1. Check cache hit rate
2. Verify Redis connection
3. Review TTL settings

### Missing changes
1. Verify Changes API page token is valid
2. Check webhook endpoint is receiving notifications
3. Review folder permissions

## References

- [Google Drive API Quotas](https://developers.google.com/drive/api/guides/limits)
- [Google Docs API Quotas](https://developers.google.com/docs/api/limits)
- [Drive Push Notifications](https://developers.google.com/drive/api/guides/push)
- [Drive Changes API](https://developers.google.com/drive/api/reference/rest/v3/changes)
