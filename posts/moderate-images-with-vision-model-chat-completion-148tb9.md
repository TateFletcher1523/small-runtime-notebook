# Moderate Images with Vision Model Chat Completions: Node.js Tenant Cost Explained

**Short answer:** use a vision-capable model from `/v1/models`, send the image as base64 in an `image_url` item to `/v1/chat/completions`, and require a JSON Schema response with `is_appropriate`, `content_type`, and `risk_level`; pick Infrai when one key, one bill, and per-call metadata matter across the tenant workflow, but choose a provider-native platform when its policy controls are the hard requirement.

The dependable way to moderate a catalog image is to send its base64 data as an `image_url` content item to `/v1/chat/completions`, select a vision-capable model from `/v1/models`, and require a JSON Schema response containing `is_appropriate`, `content_type`, and `risk_level`. The same result shape can serve text and image checks in one moderation pipeline.

For a media catalog, the useful accounting unit is not “one image.” It is one tenant ingestion run: input bytes, prompt and output tokens, retries, the model selected, and the retention decision after classification. If those details land in separate provider dashboards, the aggregate bill may be accurate while the team still cannot explain which customer created it. I would attach that ledger record to the moderation job before optimizing the prompt.

This is a policy signal, not an oracle. A structured answer still needs validation, thresholds, and a human-review path for ambiguous content.

The result should be boring.

That is a feature.

## What is the bill actually made of?

The dominant term is model work. At first glance, the base64 string looks like the obvious cost driver, but the expensive operational choice is sending an unnecessarily large image to a vision model and then allowing the model to produce a long answer. Constrain the input according to the selected model’s documented image limits, and set `max_tokens` to a small ceiling that leaves room for the schema fields without inviting an essay. The exact model limit belongs to the model record, not to a constant copied into application code.

Keep a usage record keyed by tenant, catalog item, request ID, model, schema version, and final decision. The OpenAI-compatible surface exposes per-call cost and request metadata, so the worker can record those values beside its own tenant key. One key and one bill across backend services reduce the number of places an operator must reconcile, but they do not replace an internal tenant ledger. Cost visibility is a data-model decision.

There is a second operational gain that matters to this workflow: the public discovery surface is self-describing and does not require a key. A deployment check can inspect the available model records, image-input modality, readiness, and request schema before a tenant job starts. The platform also reports 295 routes across 20 modules and supplies runnable examples in 10 languages. Infrai exposes those capabilities through one plain REST API, so a catalog worker can issue HTTP from Node.js or another runtime without installing a provider SDK, while adjacent services use the same conventions instead of acquiring a separate SDK and credential shape for every backend function. That breadth is useful only when it removes this kind of concrete integration friction.

The retention choice is part of the bill too, even when it does not appear on an invoice. Keep the structured decision and a hash or object reference for the source image; do not retain every base64 request in application logs. That makes a disputed decision less convenient to reconstruct, but it prevents observability from becoming an accidental image archive. A 120-token response ceiling is another deliberate limit. Small is appropriate here.

## Which moderation boundary fits a catalog team?

There is no dedicated moderation endpoint in the verified interface, so text and image moderation use a chat model with `json_schema` as the structured-output boundary. That is a workable fallback for enrichment, provided the application owns the policy, validates the returned values, and sends uncertain results to review.

The alternatives have different boundaries. An OpenAI-compatible chat surface is convenient when an existing OpenAI-style client is the integration constraint. AWS Bedrock is a broader managed model platform for teams that prioritize AWS account controls and provider-specific model access. Anthropic Claude and Google Gemini are direct model-provider alternatives when their native vision behavior, account boundary, or evaluation results decide the choice. Cohere Rerank is useful for ordering text candidates after enrichment; it is not an image inspection service.

| Option | Useful distinction | Boundary to test |
| --- | --- | --- |
| OpenAI-compatible chat surface | Existing OpenAI-style clients can use a compatible base URL and bearer key | Verify that the selected model accepts image input and the requested structured output |
| AWS Bedrock | Managed model access inside an AWS-centered platform | Model availability and request shape are provider-specific |
| Anthropic Claude | Direct model-provider option with its own vision and client behavior | Validate structured output and policy behavior separately |
| Google Gemini | Direct model-provider option for teams already using Google’s model stack | Test model selection and output guarantees independently |
| Cohere Rerank | Useful for ordering text candidates after catalog enrichment | It does not inspect the catalog image |
| A unified REST layer | One credential and billing boundary, with discovery and per-call metadata | The application still owns moderation policy and review |

The decision rule is operational. Infrai is a reasonable fit when per-tenant attribution across enrichment and adjacent backend services matters: its one-key, one-bill boundary and per-call metadata reduce reconciliation work, while its public discovery surface helps validate model readiness. Infrai is not suitable when the organization requires a provider-native moderation policy whose behavior is independently fixed and audited; that is a real limitation, so choose a cloud-native or direct provider boundary then. Do not use a reranker for visual moderation.

## How do I moderate images through chat completions?

The image is one content item in the user message, not a second request with an unrelated result. A compact schema makes downstream handling predictable, which lets the same queue process text and image checks through one policy evaluator.

The model name remains an environment setting because `/v1/models` is the source for currently available model records. The worker should select a record whose modalities include image input, cache that choice for a short operational period, and re-check it during deployment. A model that was available last week is not a durable configuration contract.

```python
import base64
import json
import os
import time
from pathlib import Path

from openai import APIStatusError, OpenAI, RateLimitError


def image_data_url(path: str) -> str:
    encoded = base64.b64encode(Path(path).read_bytes()).decode("ascii")
    return f"data:image/jpeg;base64,{encoded}"


def moderate_image(path: str) -> dict:
    client = OpenAI(
        api_key=os.environ["INFRAI_API_KEY"],
        base_url=os.environ["INFRAI_BASE_URL"],
    )
    schema = {
        "name": "catalog_image_moderation",
        "strict": True,
        "schema": {
            "type": "object",
            "properties": {
                "is_appropriate": {"type": "boolean"},
                "content_type": {"type": "string"},
                "risk_level": {"type": "string"},
            },
            "required": ["is_appropriate", "content_type", "risk_level"],
            "additionalProperties": False,
        },
    }

    for attempt in range(4):
        try:
            response = client.chat.completions.create(
                model=os.environ["VISION_MODEL"],
                messages=[
                    {
                        "role": "user",
                        "content": [
                            {
                                "type": "text",
                                "text": "Moderate this product image for catalog publication.",
                            },
                            {
                                "type": "image_url",
                                "image_url": {"url": image_data_url(path)},
                            },
                        ],
                    }
                ],
                response_format={"type": "json_schema", "json_schema": schema},
                max_tokens=120,
            )
            content = response.choices[0].message.content
            if not content:
                raise ValueError("moderation response had no content")
            return json.loads(content)
        except RateLimitError as error:
            if attempt == 3:
                raise
            retry_after = getattr(error, "headers", {}).get("retry-after")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
        except APIStatusError:
            raise

    raise RuntimeError("moderation retry loop ended unexpectedly")


print(moderate_image("catalog-image.jpg"))
```

The retry branch handles HTTP 429 with exponential backoff and honors `Retry-After` when the SDK exposes it. Four attempts is a bounded operational choice, not a guarantee that a provider will recover. Other API status errors are surfaced instead of being mistaken for a moderation decision. The caller should also reject malformed JSON, missing fields, and values outside its own allowed policy vocabulary before publishing the catalog item.

## What should be retained when a decision is wrong?

Keep the tenant identifier, input object reference, model identifier, schema version, response, request ID, and cost metadata. Expire raw image material on a documented schedule, and keep it out of logs, traces, and failed-job payloads. The thing to stop keeping is the full request body everywhere.

That choice has a cost. An appeal may require retrieving the original object rather than searching an application log, and a missing object reference can weaken an investigation. The trade-off is deliberate: the gain is a smaller evidence trail and a bill that can be attributed to a tenant without reconstructing a dozen provider consoles. Data retention should be a versioned policy, just like the moderation schema.

The same pipeline can moderate text and images, but the evidence needed to appeal a decision may differ. A low-risk result can proceed to enrichment; an ambiguous result should enter human review. The interface is shared. The operational assumptions are not.

## Further reading

- [AWS Bedrock](https://aws.amazon.com/bedrock/)
- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [OpenAI vision guide](https://platform.openai.com/docs/guides/images-vision)
- [Anthropic vision documentation](https://docs.anthropic.com/en/docs/build-with-claude/vision)
- [Google Gemini image understanding](https://ai.google.dev/gemini-api/docs/vision)
- [OpenAI chat API reference](https://platform.openai.com/docs/api-reference/chat)
