---
name: python-fastmcp-multimodal-applications
description: Guides teams to build Multimodal AI Applications (vision, text, audio, tabular data) with FastMCP — implementing Adaptive Routing, Orchestrated Pipelines, Collaborative Processing, cross-modal context sharing, confidence propagation, and multi-session synthesis.
---

# Python FastMCP: Multimodal Applications

This skill helps AI design and build multimodal applications (integrating vision, text, speech/audio, and tabular data) using FastMCP. By wrapping modality-specific models (computer vision object detectors, speech-to-text engines, multimodal LLMs) into standardized FastMCP resources (`@mcp.resource()`) and analysis tools (`@mcp.tool()`), developers eliminate custom API glue code and enable cross-modal context sharing and dynamic capability routing.

---

## When to use this skill

Use this skill when you need to:

- process heterogeneous inputs (images, document scans, audio clips, text prompts) within a unified reasoning framework,
- implement **Adaptive Routing** where input types automatically dispatch to specialized FastMCP vision, speech, or LLM servers,
- implement **Cross-Modal Context Sharing** (e.g. passing vision bounding-box metadata to an LLM to correct OCR errors or generate image captions),
- propagate confidence scores across modalities (weighting vision models higher for visual queries and language models for text queries),
- execute multi-modal processing pipelines in parallel using `asyncio.gather`,
- group multi-modal artifacts (e.g. image + audio transcript + post text) into analysis sessions (`create_session`, `synthesize_insights`).

---

## Outcome

Produce a FastMCP multimodal server or orchestrator that:

- exposes modality resources (`images`, `texts`, `audios`) and tools (`analyze_content`, `create_session`, `synthesize_insights`),
- routes incoming media streams to appropriate vision/audio/text processing handlers,
- shares intermediate context annotations (`confidence`, `bounding_boxes`, `timestamps`, `lastModified`) across processing stages,
- synthesizes aggregate confidence scores across modalities.

---

## Instructions for the AI

1. **Model Multimodal Capabilities as FastMCP Tools and Resources**
   - **Resources:** Expose raw media resources or processed embeddings (`image://{id}`, `audio://{id}`, `text://{id}`).
   - **Tools:** Expose modality analysis tools (`analyze_vision`, `transcribe_audio`, `synthesize_multimodal_insights`).
   - Example implementation pattern:
     ```python
     from typing import Dict, List, Any
     from pydantic import BaseModel
     from mcp.server.fastmcp import FastMCP

     mcp = FastMCP("multimodal-analyzer")

     class VisionAnalysis(BaseModel):
         detected_objects: List[str]
         ocr_text: str
         confidence: float

     class AudioAnalysis(BaseModel):
         transcript: str
         speaker_count: int
         confidence: float

     @mcp.tool()
     def analyze_image(image_uri: str) -> str:
         """Perform object detection and OCR text extraction on an image."""
         # Simulate vision model inference
         analysis = VisionAnalysis(
             detected_objects=["laptop", "chart", "person"],
             ocr_text="Q4 Revenue Growth +25%",
             confidence=0.92,
         )
         return analysis.model_dump_json()

     @mcp.tool()
     def analyze_audio(audio_uri: str) -> str:
         """Perform speech-to-text transcription and speaker diarization."""
         analysis = AudioAnalysis(
             transcript="Welcome everyone to our quarterly financial review.",
             speaker_count=2,
             confidence=0.88,
         )
         return analysis.model_dump_json()
     ```

2. **Implement Cross-Modal Context Sharing**
   - Pass outputs from visual or audio analysis tools directly into downstream LLM prompt contexts.
   - Example cross-modal synthesis tool:
     ```python
     @mcp.tool()
     def synthesize_insights(session_id: str, image_json: str, audio_json: str) -> str:
         """Synthesize cross-modal insights by combining vision analysis and speech transcripts."""
         import json
         img_data = json.loads(image_json)
         audio_data = json.loads(audio_json)

         # Calculate weighted cross-modal confidence score
         avg_confidence = (img_data["confidence"] + audio_data["confidence"]) / 2.0

         synthesis = {
             "session_id": session_id,
             "summary": f"Meeting with {audio_data['speaker_count']} speakers discussing slide containing '{img_data['ocr_text']}'.",
             "visual_context": img_data["detected_objects"],
             "spoken_context": audio_data["transcript"],
             "cross_modal_confidence": avg_confidence,
         }
         return json.dumps(synthesis, indent=2)
     ```

3. **Orchestrate Parallel Multimodal Pipelines**
   - Use `asyncio.gather` to execute vision, speech, and text analysis tools concurrently to minimize total request latency.

4. **Implement Adaptive Routing**
   - Inspect input MIME types or payload metadata and dynamically dispatch requests to target FastMCP vision or audio servers.

5. **Track Modality Timestamps & Synchronization**
   - For video or streaming audio, attach timestamp intervals (`start_ms`, `end_ms`) to tool outputs for frame alignment.

---

## Decision points and guidance

- **Single Multimodal Server vs Specialized Domain Servers?** Use specialized servers for distinct modalities (e.g. GPU-accelerated Vision Server vs Audio Server) when deployment infrastructure differs; wrap them in an Orchestrated Pipeline client.
- **How to handle missing modalities?** When an audio clip or image is missing in a multi-modal post, generate text summaries using available modalities and state missing inputs in synthesis responses.
- **When to re-query vision models?** If language model synthesis detects low confidence or contradictory OCR text, trigger targeted sub-crop vision analysis tools.

---

## Quality criteria

- **Unified Protocol:** Vision, text, and speech analysis components communicate over standard JSON-RPC FastMCP tools.
- **Concurrent Processing:** Independent modality analysis tasks run concurrently via `asyncio.gather`.
- **Confidence Propagated:** Synthesized outputs evaluate confidence scores across all contributing modalities.
- **Clear Metadata:** Extracted visual bounding boxes, OCR text, and speech transcripts are structured using Pydantic models.

---

## Example prompts

- "Build a FastMCP server that exposes vision object detection and audio transcription tools."
- "Implement an adaptive routing pipeline that sends images to a vision FastMCP server and audio to a speech FastMCP server."
- "Synthesize cross-modal insights by combining image OCR context with audio transcripts."

---

## Related guidance

- python-fastmcp-server-basics
- python-fastmcp-resource-providers
- python-fastmcp-multi-agent-orchestration
- python-fastmcp-client-integration
