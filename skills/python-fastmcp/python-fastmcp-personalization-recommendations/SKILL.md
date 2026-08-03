---
name: python-fastmcp-personalization-recommendations
description: Guides teams to build Personalization and Recommendation Systems with FastMCP — aggregating unified user profiles across CRM/Analytics silos (profile://, behavior://, content://), real-time context boosting, multi-factor scoring tools (get_recommendations), explainability metadata, cold-start strategies, and privacy metrics (Demographic Parity).
---

# Python FastMCP: Personalization & Recommendation Systems

This skill helps AI design and build real-time recommendation engines and personalization services using FastMCP. By wrapping scattered customer data (CRM demographics, web analytics behavior, purchase histories, product catalogs) into unified FastMCP resources and recommendation tools (`get_recommendations`), organizations deliver context-aware, explainable, and cross-platform personalized experiences.

---

## When to use this skill

Use this skill when you need to:

- build a recommendation engine that adapts in real time to user behavior, time of day, location, and device context,
- aggregate fragmented user profile data across enterprise silos into unified FastMCP resources (`profile://{user_id}`, `behavior://{user_id}`, `content://{item_id}`),
- implement multi-factor item scoring combining content matching, user experience level, item popularity, and real-time context boosts,
- mitigate the **Cold-Start Problem** for new users or items using hybrid fallback weighting (e.g. 70% popularity / 30% content similarity decaying over time),
- return explainability metadata alongside recommendations (`"Based on your interest in AI and morning reading context"`),
- enforce privacy-preserving personalization (data minimization, anonymization, fairness monitoring via Demographic Parity and Equalized Odds).

---

## Outcome

Produce a FastMCP recommendation server or integration that:

- exposes user profiles, behavioral logs, and content catalogs as standardized resources (`resources/list`, `resources/read`),
- implements recommendation tools (`@mcp.tool() get_recommendations`) returning scored item lists and human-readable explanation factors,
- applies real-time contextual boosts (time of day, device type) without waiting for offline model retraining cycles,
- provides cross-platform consistency for web, mobile, support, and conversational AI agents.

---

## Instructions for the AI

1. **Model Personalization Data as FastMCP Resources**
   - **User Profiles (`profile://{user_id}`):** Demographic information, explicit interests, experience level, and subscription tier.
   - **Behavioral Logs (`behavior://{user_id}`):** Interaction tuples (timestamp, action, item_id, context attributes: device, location, time_of_day).
   - **Content Catalog (`content://{item_id}`):** Item metadata, category tags, difficulty rating, and global popularity score.
   - Example resource structure:
     ```python
     import json
     from pydantic import BaseModel
     from mcp.server.fastmcp import FastMCP

     mcp = FastMCP("personalization-engine")

     USER_PROFILES = {
         "usr-101": {"interests": ["ai", "python", "data-engineering"], "level": "advanced"},
         "usr-102": {"interests": ["design", "ui", "ux"], "level": "beginner"},
     }

     CONTENT_CATALOG = {
         "art-01": {"title": "FastMCP Architecture", "category": "ai", "tags": ["ai", "python", "advanced"], "popularity": 0.9},
         "art-02": {"title": "Design Principles", "category": "design", "tags": ["design", "ui", "beginner"], "popularity": 0.75},
     }

     @mcp.resource("profile://{user_id}")
     def get_user_profile(user_id: str) -> str:
         """Return user preferences and experience level."""
         profile = USER_PROFILES.get(user_id)
         if not profile:
             raise ValueError(f"User {user_id} not found.")
         return json.dumps(profile)

     @mcp.resource("content://catalog")
     def get_content_catalog() -> str:
         """Return the full content item catalog."""
         return json.dumps(CONTENT_CATALOG)
     ```

2. **Implement Multi-Factor Scoring Tool with Context Boosts**
   - Implement `@mcp.tool() get_recommendations(user_id: str, context: dict, limit: int = 3)`.
   - Calculate item scores using weighted factors:
     - **Tag Matching (40%):** Overlap between user interests and item tags.
     - **Experience Level Matching (30%):** User skill level matching item difficulty tag.
     - **Popularity Weight (20%):** Global item engagement score.
     - **Real-Time Context Boost (10%):** Bonus for matching situational context (e.g. morning device boost).
   - Filter out items the user has already viewed or purchased.

3. **Handle Cold-Start Scenarios**
   - For new users with empty behavioral histories, use a hybrid cold-start model:
     - Initial scoring: $Score = 0.7 \times Popularity + 0.3 \times ContentSimilarity$.
     - Transition dynamically to multi-factor scoring as user interactions accumulate.

4. **Return Explainability & Transparency Metadata**
   - Recommendation tool output MUST include clear explanation factors for user trust and UI display.
   - Example explanation output:
     ```json
     {
       "recommendations": [
         {
           "item_id": "art-01",
           "title": "FastMCP Architecture",
           "score": 0.88,
           "explanation": "Matched interests (ai, python), experience level (advanced), and popular in category."
         }
       ],
       "context_applied": {"time_of_day": "morning", "device": "mobile"}
     }
     ```

5. **Enforce Privacy, Fairness, and Data Minimization**
   - Collect and query only the minimum necessary user attributes for scoring.
   - Monitor algorithmic fairness across demographic groups (evaluating Demographic Parity and Equalized Odds metrics).
   - Support user opt-out and profile editing tools (`update_user_preferences`, `reset_interaction_history`).

---

## Decision points and guidance

- **Batch Retraining vs Real-Time Inference?** Compute feature vectors and historical interaction scores asynchronously; apply situational context boosts (device, time of day, location) at runtime during `get_recommendations` tool execution.
- **How to prevent filter bubbles?** Introduce a small exploration factor (e.g. 10% random high-quality item injection) into recommendation sets to expose users to novel topics.
- **Where to maintain session context?** Pass situational context in the `context` argument of `get_recommendations` or derive context from client metadata in `_meta`.

---

## Quality criteria

- **Unified Resource Access:** Profiles, interaction logs, and catalogs are queryable via standard MCP URIs (`profile://`, `behavior://`, `content://`).
- **Explainable Output:** Recommendations include transparent score breakdowns and rationale text.
- **Cold-Start Resilient:** New users receive high-quality popularity/category fallback recommendations.
- **Context-Aware:** Situational context signals dynamically adjust ranking outputs without offline re-indexing.

---

## Example prompts

- "Build a FastMCP recommendation server with get_recommendations tool supporting interest matching and context boosts."
- "Implement a unified user profile resource provider combining CRM data and recent behavioral logs."
- "Add cold-start fallback scoring and explainability metadata to our FastMCP personalization engine."

---

## Related guidance

- python-fastmcp-resource-providers
- python-fastmcp-security-discovery
- python-fastmcp-enterprise-knowledge-management
- python-fastmcp-client-integration
