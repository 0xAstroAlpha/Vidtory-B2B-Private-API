# Vidtory Core API – Request Samples (base `https://oldapi84.vidtory.net`)

Các mẫu `curl` đã khớp code trong `media-api` và `cookie-api`. Media bắt buộc header `X-API-Key`, output công khai ở `/outputs/...`.

## Media API (X-API-Key: ak_…)

### Image nano2 (POST `/api/image/nano2`)
```bash
IMG_BASE64=$(base64 -w0 ./ref.jpg)
curl -X POST https://oldapi84.vidtory.net/api/image/nano2 \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <AK_API_KEY>" \
  -H "Origin: https://vidtory.net" \
  -d "{
    \"contents\": [{
      \"parts\": [
        { \"text\": \"An office group photo of these people, they are smiling\" },
        { \"inline_data\": { \"mime_type\": \"image/jpeg\", \"data\": \"${IMG_BASE64}\" } }
      ]
    }],
    \"generationConfig\": { \"imageConfig\": { \"aspectRatio\": \"IMAGE_ASPECT_RATIO_LANDSCAPE\" } },
    \"cleanup\": true
  }"
```
Pending (202) trả:
```json
{
  "status": "pending",
  "jobId": "c0c3e8fe-1b4a-4ad5-b6fa-1e4d9f2f8b67",
  "statusUrl": "https://oldapi84.vidtory.net/api/image/nano2/jobs/c0c3e8fe-1b4a-4ad5-b6fa-1e4d9f2f8b67",
  "pollIntervalMs": 5000
}
```
Hoàn tất (200) trả:
```json
{
  "status": "done",
  "jobId": "c0c3e8fe-1b4a-4ad5-b6fa-1e4d9f2f8b67",
  "result": {
    "requestId": "adb7039f-9f2a-4b62-8afd-c503d3dd4efa",
    "prompt": "An office group photo of these people, they are smiling",
    "outputs": [
      {
        "fileName": "nano-brian-1-1764211044921.jpg",
        "size": 122130,
        "contentType": "image/jpeg",
        "url": "https://oldapi84.vidtory.net/outputs/nano-brian-1-1764211044921.jpg"
      }
    ]
  }
}
```
*Lưu ý: `outputs` nằm lồng trong `result`.*

Lỗi auth/thiếu key: 401 `API_KEY_VERIFICATION_ERROR`.
Lỗi 403 Forbidden: Bắt buộc thêm header `Origin: https://vidtory.net` (hoặc domain whitelist) để vượt qua Nginx check.

### Poll nano/nano2 (GET `/api/image/nano2/jobs/:jobId`)
```bash
curl -X GET https://oldapi84.vidtory.net/api/image/nano2/jobs/<JOB_ID> \
  -H "X-API-Key: <AK_API_KEY>"
```
`status: pending` -> 202 như trên; `status: done` -> 200 + outputs; sau ~1h job hết TTL trả 404.

### Image nano v1 (POST `/api/image/nano`)
Cùng body/response như nano2, model GEM_PIX (tối đa 3 hình tham chiếu, 3 credits), poll ở `/api/image/nano/jobs/:jobId`.

### Image classic / multi-elements (POST `/api/image/generate`)
```bash
curl -X POST https://oldapi84.vidtory.net/api/image/generate \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <AK_API_KEY>" \
  -d '{
    "prompt": "Portrait of a warrior",
    "subjects": [{ "url": "https://images.pexels.com/photos/1040880/pexels-photo-1040880.jpeg?w=640" }],
    "styles": [{ "url": "https://images.pexels.com/photos/6625910/pexels-photo-6625910.jpeg?w=640" }],
    "aspectRatio": "IMAGE_ASPECT_RATIO_PORTRAIT",
    "cleanup": true
  }'
```
Response mẫu:
```json
{
  "prompt": "Portrait of a warrior",
  "mediaGenerationIds": {
    "subjects": ["ai-img-subject-1"],
    "scenes": [],
    "styles": ["ai-img-style-1"]
  },
  "urls": [
    "https://oldapi84.vidtory.net/outputs/generated-portrait-1.jpg"
  ],
  "cleanup": { "success": true }
}
```

### Video generate (POST `/api/video/generate`)
```bash
curl -X POST https://oldapi84.vidtory.net/api/video/generate \
  -H "Content-Type: application/json" \
  -H "X-API-Key: <AK_API_KEY>" \
  -d '{
    "prompt": "A dragon flying over mountains",
    "aspectRatio": "VIDEO_ASPECT_RATIO_LANDSCAPE",
    "generationMode": "START_ONLY",
    "upscale": true,
    "cleanup": true
  }'
```
- Endpoints: `/api/video/generate` hoặc alias `/api/generate-video/d7a53315-74b0-4d92-b7bc-0cebc51412e2`
- `generationMode` là bắt buộc (dùng `generationMode` hoặc alias `mode`). Các mode hợp lệ:
  - `START_ONLY`: bắt buộc `startImage`; `endImage` và `referenceImages` bị cấm.
  - `START_AND_END`: bắt buộc cả `startImage` và `endImage`; `referenceImages` bị cấm.
  - `REFERENCE_IMAGES`: bắt buộc `referenceImages` (ít nhất 1 item, nên kèm `imageUsageType`); `startImage`/`endImage` bị cấm; chỉ hỗ trợ aspect landscape (portrait trả 400).
- Aspect ratio cho phép: `VIDEO_ASPECT_RATIO_LANDSCAPE`, `VIDEO_ASPECT_RATIO_PORTRAIT` (referenceImages chỉ landscape).


Ví dụ per-mode:
```bash

# START_ONLY
curl -X POST https://oldapi84.vidtory.net/api/video/generate \
  -H "Content-Type: application/json" -H "X-API-Key: <AK_API_KEY>" \
  -d '{
    "prompt": "Fly through a neon city",
    "generationMode": "START_ONLY",
    "startImage": { "url": "https://images.pexels.com/photos/210186/pexels-photo-210186.jpeg?w=640" }
  }'

# START_AND_END
curl -X POST https://oldapi84.vidtory.net/api/video/generate \
  -H "Content-Type: application/json" -H "X-API-Key: <AK_API_KEY>" \
  -d '{
    "prompt": "Time-lapse travel",
    "generationMode": "START_AND_END",
    "startImage": { "url": "https://oldapi84.vidtory.net/outputs/start.jpg" },
    "endImage": { "url": "https://oldapi84.vidtory.net/outputs/end.jpg" }
  }'

# REFERENCE_IMAGES (asset library, landscape only)
curl -X POST https://oldapi84.vidtory.net/api/video/generate \
  -H "Content-Type: application/json" -H "X-API-Key: <AK_API_KEY>" \
  -d '{
    "prompt": "Corporate interview b-roll",
    "generationMode": "REFERENCE_IMAGES",
    "referenceImages": [
      { "url": "https://images.pexels.com/photos/3184465/pexels-photo-3184465.jpeg?w=640", "imageUsageType": "IMAGE_USAGE_TYPE_SUBJECT" },
      { "url": "https://images.pexels.com/photos/3184463/pexels-photo-3184463.jpeg?w=640", "imageUsageType": "IMAGE_USAGE_TYPE_STYLE" }
    ],
    "aspectRatio": "VIDEO_ASPECT_RATIO_LANDSCAPE"
  }'
```
Response nhận ngay (202):
```json
{
  "status": "PROCESSING",
  "jobId": "8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015",
  "statusUrl": "https://oldapi84.vidtory.net/api/video/jobs/8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015",
  "downloadUrl": "https://oldapi84.vidtory.net/api/video/jobs/8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015/file",
  "pollIntervalMs": 10000,
  "payload": {
    "jobId": "8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015",
    "status": "PROCESSING",
    "prompt": "A dragon flying over mountains",
    "projectId": "dev-video-project-id",
    "seed": 23456,
    "sceneId": "5f6d1c25-1c4c-45e0-9f84-2d68c88c9c6b",
    "operationName": "operations/video-gen-123",
    "videoModelKey": "veo_3_1_i2v_s_fast_ultra",
    "url": "https://oldapi84.vidtory.net/outputs/video-1737042703123.mp4",
    "video": {
      "fileName": "video-1737042703123.mp4",
      "url": "https://oldapi84.vidtory.net/outputs/video-1737042703123.mp4",
      "downloadUrl": "https://oldapi84.vidtory.net/api/video/jobs/8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015/file"
    },
    "upscale": { "requested": true, "status": "PROCESSING" },
    "ready": false,
    "statusUrl": "https://oldapi84.vidtory.net/api/video/jobs/8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015",
    "downloadUrl": "https://oldapi84.vidtory.net/api/video/jobs/8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015/file",
    "pollIntervalMs": 10000
  }
}
```

Poll trạng thái:
```bash
curl -X GET https://oldapi84.vidtory.net/api/video/jobs/8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015 \
  -H "X-API-Key: <AK_API_KEY>"
```
Khi xong sẽ trả:
```json
{
  "jobId": "8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015",
  "status": "COMPLETE",
  "payload": {
    "ready": true,
    "mediaGenerationId": "labs-media-123",
    "video": {
      "fileName": "video-1737042703123.mp4",
      "mimeType": "video/mp4",
      "size": 47218341,
      "url": "https://oldapi84.vidtory.net/outputs/video-1737042703123.mp4",
      "downloadUrl": "https://oldapi84.vidtory.net/api/video/jobs/8d0f1b4d-4e77-4cb9-97b2-1bd4dbacd015/file"
    },
    "upscale": {
      "status": "MEDIA_GENERATION_STATUS_COMPLETE",
      "video": {
        "fileName": "video-1737042703123-upscaled.mp4",
        "url": "https://oldapi84.vidtory.net/outputs/video-1737042703123-upscaled.mp4"
      }
    }
  }
}
```
Download file (429/202 nếu chưa sẵn): `curl -L -H "X-API-Key: <AK_API_KEY>" https://oldapi84.vidtory.net/api/video/jobs/<JOB_ID>/file -o out.mp4`.

### Video relax generate (POST `/api/video-relax/generate`)

Body/flow giống video generate, model “relaxed” (veo_*_relaxed) và route alias `/api/generate-video-relax/8e3c1c75-5f65-4c83-9d50-1b3a9b0f7b9e`. Chọn mode giống video generate (referenceImages/start+end/text-only/start-only), nhưng portrait vẫn không upscale, referenceImages vẫn chỉ landscape. Response/poll/download giữ nguyên schema.
