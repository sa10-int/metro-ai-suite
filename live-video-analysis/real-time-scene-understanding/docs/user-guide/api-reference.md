# API Reference

The backend is a FastAPI application that serves REST APIs under the `/api` prefix for stream lifecycle management, alert retrieval, frame-registry inspection, and runtime UI configuration.

## Interactive API docs

When the stack is running, FastAPI provides OpenAPI/Swagger UI at:

- `http://localhost:9100/docs`

If the backend is exposed on a different host or port, adjust accordingly.

## REST Endpoints

### Health Check

- `GET /api/health` - Liveness and basic runtime status

#### Health Response Schema

```json
{
	"status": "healthy",
	"streams_active": 0,
	"uptime_seconds": 12.4,
	"timestamp": "2026-09-07T10:20:30.123456+00:00"
}
```

### Runtime Config

- `GET /api/runtime-config.js` - Browser runtime configuration as JavaScript (`window.RUNTIME_CONFIG = {...};`)

This endpoint is consumed by the UI to resolve WebRTC signaling and metrics service URLs.

### Streams

- `GET /api/streams` - List all active streams and live processing state
- `POST /api/streams` - Create/start a new stream pipeline
- `DELETE /api/streams/{stream_id}` - Stop and remove a stream pipeline

#### Create Stream Request Schema (`POST /api/streams`)

```json
{
	"stream_id": "camera-lobby",
	"url": "rtsp://example.com/stream",
	"alert_prompt": "Look for visible fire and describe its location.",
	"deep_analyzer_prompt": "Review the clip for evidence of fire and summarize what happens."
}
```

Notes:

- `url` is required.
- `alert_prompt` and `deep_analyzer_prompt` are required.
- `stream_id` defaults to `default` if omitted/empty.
- `url` must use `rtsp://` or `rtsps://`. HTTP and HTTPS URLs are not supported.
- `stream_id`, when provided, must contain 1-64 letters, digits, underscores, or hyphens.
- Each prompt is limited to 4096 characters.

#### Create Stream Response Schema

```json
{
	"status": "added",
	"stream_id": "camera-lobby"
}
```

#### List Streams Response Schema (`GET /api/streams`)

```json
{
	"streams": [
		{
			"stream_id": "camera-lobby",
			"url": "rtsp://example.com/stream",
			"alert_prompt": "Look for visible fire and describe its location.",
			"deep_analyzer_prompt": "Review the clip for evidence of fire and summarize what happens.",
			"publishing": true,
			"codec": "h264",
			"resolution": "576x320",
			"reconnect_count": 0,
			"whep_path": "/camera-lobby/whep",
			"caption": "Yes",
			"caption_ts": 1757240432.117,
			"ttft_ms": 73.2,
			"tpot_ms": 18.7,
			"throughput_tps": 54.1,
			"alert_count": 3
		}
	]
}
```

#### Delete Stream Response Schema (`DELETE /api/streams/{stream_id}`)

```json
{
	"status": "removed",
	"stream_id": "camera-lobby"
}
```

### Alerts

- `GET /api/streams/{stream_id}/alerts` - Paginated alert listing (`limit`, `offset` query params)
- `GET /api/streams/{stream_id}/alerts/{frame_id}` - Alert detail record
- `GET /api/streams/{stream_id}/alerts/{frame_id}/thumbnail` - Alert thumbnail image (`image/jpeg`)
- `GET /api/streams/{stream_id}/alerts/{frame_id}/video` - Alert video clip (`video/mp4`)

#### List Alerts Response Schema (`GET /api/streams/{stream_id}/alerts`)

```json
{
	"total": 2,
	"alerts": [
		{
			"frame_id": "9dd96a2f-17b9-41a0-a1c5-e3f8f76b07a7",
			"trigger_caption": "Yes",
			"confirmed": true,
			"thumbnail_url": "/api/streams/camera-lobby/alerts/9dd96a2f-17b9-41a0-a1c5-e3f8f76b07a7/thumbnail",
			"uploaded_at": "2026-09-07T10:17:40.100000+00:00"
		}
	]
}
```

#### Alert Detail Response Schema (`GET /api/streams/{stream_id}/alerts/{frame_id}`)

```json
{
	"stream_id": "camera-lobby",
	"frame_id": "9dd96a2f-17b9-41a0-a1c5-e3f8f76b07a7",
	"trigger_caption": "Yes",
	"confirmed": true,
	"thumbnail_url": "/api/streams/camera-lobby/alerts/9dd96a2f-17b9-41a0-a1c5-e3f8f76b07a7/thumbnail",
	"description": "Short multi-frame analysis summary.",
	"metrics": {
		"ttft_ms": 70.2,
		"throughput_tps": 50.4
	},
	"timings": {
		"trigger_caption_ts": 1788791840.1,
		"segment_finalized_ts": 1788791845.0,
		"segment_finalize_wait_ms": 4900.0,
		"analysis_enqueued_ts": 1788791845.0,
		"analysis_started_ts": 1788791845.2,
		"queue_wait_ms": 200.0,
		"segment_ready_ts": 1788791845.3,
		"segment_readiness_wait_ms": 100.0,
		"frame_sampling_started_ts": 1788791845.3,
		"frame_sampling_completed_ts": 1788791845.6,
		"frame_sampling_duration_ms": 300.0,
		"generation_started_ts": 1788791845.6,
		"generation_completed_ts": 1788791857.6,
		"generation_duration_ms": 12000.0,
		"storage_upload_started_ts": 1788791857.6,
		"storage_upload_completed_ts": 1788791858.1,
		"storage_upload_duration_ms": 500.0,
		"alert_indexed_ts": 1788791858.1
	},
	"model": "Qwen/Qwen3-VL-8B-Instruct",
	"device": "GPU",
	"uploaded_at": "2026-09-07T10:17:40.100000+00:00",
	"video_url": "/api/streams/camera-lobby/alerts/9dd96a2f-17b9-41a0-a1c5-e3f8f76b07a7/video"
}
```

In alert details, `timings` fields ending in `_ts` are Unix timestamps in
seconds; fields ending in `_ms` are durations in milliseconds. `uploaded_at`
is retained for compatibility and is set before the storage upload tasks begin.
Use `timings.storage_upload_completed_ts` for artifact-upload completion and
`timings.alert_indexed_ts` for the server-side alert-index insertion time.

| Field | Stage | Purpose |
| --- | --- | --- |
| `trigger_caption_ts` | Alert VLM | Time the alert caption was produced; reference point for segment-finalization wait. |
| `segment_finalized_ts` | Stream/segment handoff | Time the segment containing the trigger frame was finalized. |
| `segment_finalize_wait_ms` | Alert-to-segment handoff | Time from the alert caption to segment finalization. |
| `analysis_enqueued_ts` | Deep analyzer | Time the finalized-segment job entered the work queue. |
| `analysis_started_ts` | Deep analyzer | Time the worker began processing the job. |
| `queue_wait_ms` | Deep analyzer | Time the job waited in the work queue. |
| `segment_ready_ts` | Deep analyzer | Time the finalized segment became safe to read. |
| `segment_readiness_wait_ms` | Deep analyzer | Time spent waiting for the segment file to become readable. |
| `frame_sampling_started_ts` | Deep analyzer | Time frame extraction and sampling began. |
| `frame_sampling_completed_ts` | Deep analyzer | Time frame extraction and sampling ended. |
| `frame_sampling_duration_ms` | Deep analyzer | Time spent reading and sampling segment frames. |
| `generation_started_ts` | Deep analyzer | Time model generation began. |
| `generation_completed_ts` | Deep analyzer | Time model generation ended. |
| `generation_duration_ms` | Deep analyzer | Time spent generating the analysis. |
| `storage_upload_started_ts` | Storage | Time artifact uploads were dispatched. |
| `storage_upload_completed_ts` | Storage | Time all artifact upload tasks completed. |
| `storage_upload_duration_ms` | Storage | Total duration of the artifact upload tasks. |
| `alert_indexed_ts` | API/index | Time the completed alert was inserted into the app's alert index. |

#### Alerts Query Parameters

- `limit` (default `20`, min `1`, max `100`)
- `offset` (default `0`, min `0`)

### Frame Registry

- `GET /api/registry/stats` - Registry totals and per-stream counts
- `GET /api/registry/stream/{stream_id}` - Latest frame records for one stream (`limit` query param)
- `GET /api/registry/frame/{frame_id}` - Lookup a specific frame record by UUID

#### Registry Stats Response Schema (`GET /api/registry/stats`)

```json
{
	"total": 134,
	"per_stream_capacity": 500,
	"per_stream": {
		"camera-lobby": 84,
		"camera-parking": 50
	}
}
```

#### Registry Records Response Schema

Returned by `GET /api/registry/stream/{stream_id}` (`limit` defaults to `50`).

```json
{
	"records": [
		{
			"frame_id": "9dd96a2f-17b9-41a0-a1c5-e3f8f76b07a7",
			"stream_id": "camera-lobby",
			"segment_path": "segments/camera-lobby_segment_0012.mp4",
			"pts_seconds": 173.2,
			"created_ts": 1757240432.117
		}
	]
}
```

#### Registry Frame Detail Response Schema (`GET /api/registry/frame/{frame_id}`)

```json
{
	"frame_id": "9dd96a2f-17b9-41a0-a1c5-e3f8f76b07a7",
	"stream_id": "camera-lobby",
	"rtsp_url": "rtsp://example.com/stream",
	"segment_path": "segments/camera-lobby_segment_0012.mp4",
	"pts_seconds": 173.2,
	"created_ts": 1757240432.117
}
```

## Streaming Endpoints

The stack also includes two related streaming surfaces outside of FastAPI:

- WebRTC playback signaling and media via MediaMTX WHEP (`/{stream_id}/whep`)
- System metrics via the bundled `metrics-manager` (`intel/metrics-manager`) SSE endpoint (`/metrics/stream`)

### WebRTC Playback (MediaMTX WHEP)

- `POST /{stream_id}/whep`

This endpoint is served by MediaMTX, not FastAPI. The backend returns `whep_path` from `GET /api/streams`, and the UI completes SDP offer/answer negotiation against this WHEP path.

### System Metrics SSE (metrics-manager)

- `GET /metrics/stream`

This endpoint is served by the `metrics-manager` sidecar service (default port `9090`), not by the FastAPI backend. The UI subscribes with `EventSource` for live CPU/GPU/NPU/memory/power telemetry.

Example URL:

- `http://localhost:9090/metrics/stream`

Related metrics capability endpoint:

- `GET /api/v1/capabilities?profile=minimal`

## Error Behavior

Typical API error codes:

- `400` - invalid request payload or malformed path/query values
- `422` - request body violates schema limits (for example, a prompt exceeds 4096 characters)
- `404` - stream, alert, thumbnail, video, or frame record not found
- `409` - stream conflict (for example, duplicate `stream_id`)

## Related docs

- [Get Started](./get-started.md)
- [Troubleshooting](./troubleshooting.md)
