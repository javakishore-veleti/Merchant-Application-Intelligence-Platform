# AWS Bedrock LLMOps

## Purpose

Integrate Amazon Bedrock LLMs for MAIP anomaly detection, alert summarization, and policy compliance.

---

## Prerequisites

```bash
# ALWAYS set profile first
export AWS_PROFILE=maip-${MAIP_ENV:-dev} AWS_REGION=us-east-1

# Verify model access
aws bedrock list-foundation-models \
  --query "modelSummaries[?contains(modelId, 'claude')].modelId"
```

---

## Recommended Models

| Model | Use Case | Cost |
|-------|----------|------|
| `anthropic.claude-3-sonnet-20240229-v1:0` | Anomaly detection, analysis | Medium |
| `anthropic.claude-3-haiku-20240307-v1:0` | Alert summarization | Low |

---

## Validated Commands

### Invoke Model - Anomaly Detection

```bash
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-sonnet-20240229-v1:0 \
  --content-type application/json \
  --body '{
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 1024,
    "messages": [{
      "role": "user",
      "content": "Analyze this telemetry for anomalies: CPU: 95%, Memory: 88%, Kafka Lag: 50000, Response Time: 2500ms. Normal baselines: CPU <70%, Memory <75%, Lag <1000, Response <500ms."
    }]
  }' \
  output.json

cat output.json | jq -r '.content[0].text'
```

### Invoke Model - Alert Summarization

```bash
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-haiku-20240307-v1:0 \
  --content-type application/json \
  --body '{
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 512,
    "messages": [{
      "role": "user",
      "content": "Summarize for on-call: [Alert 1: High CPU on order-service, Alert 2: Kafka lag increasing, Alert 3: RDS connections exhausted]. Provide root cause and actions."
    }]
  }' \
  output.json
```

---

## Integration Code

### Python (PySpark Jobs)

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

def detect_anomaly(telemetry_data: dict) -> str:
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',
        contentType='application/json',
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 1024,
            "messages": [{
                "role": "user",
                "content": f"Analyze for anomalies: {json.dumps(telemetry_data)}"
            }]
        })
    )
    result = json.loads(response['body'].read())
    return result['content'][0]['text']
```

### Java (Spring Boot Services)

```java
// pom.xml: software.amazon.awssdk:bedrock-runtime

import software.amazon.awssdk.services.bedrockruntime.BedrockRuntimeClient;
import software.amazon.awssdk.services.bedrockruntime.model.InvokeModelRequest;
import software.amazon.awssdk.core.SdkBytes;

@Service
public class AnomalyDetectionService {
    
    private final BedrockRuntimeClient bedrock = BedrockRuntimeClient.builder()
        .region(Region.US_EAST_1)
        .build();
    
    public String analyzeAnomaly(String telemetryJson) {
        String payload = String.format("""
            {
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": "Analyze: %s"}]
            }
            """, telemetryJson);
        
        InvokeModelRequest request = InvokeModelRequest.builder()
            .modelId("anthropic.claude-3-sonnet-20240229-v1:0")
            .contentType("application/json")
            .body(SdkBytes.fromUtf8String(payload))
            .build();
        
        return bedrock.invokeModel(request).body().asUtf8String();
    }
}
```

---

## Guardrails (Optional)

### Create Guardrail

```bash
aws bedrock create-guardrail \
  --name maip-pii-filter \
  --description "Filter PII from MAIP LLM responses" \
  --blocked-input-messaging "Input contains sensitive data" \
  --blocked-outputs-messaging "Response filtered for compliance"
```

---

## Cost Monitoring

```bash
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Bedrock"]}}'
```

---

## Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `AccessDeniedException` | Model not enabled | Enable model in Bedrock console |
| `ValidationException` | Wrong payload format | Check `anthropic_version` field |
| `ThrottlingException` | Rate limit | Implement exponential backoff |
| `ModelTimeoutException` | Response too slow | Reduce `max_tokens` or use Haiku |

---

## Best Practices

1. **Use Haiku for simple tasks** - faster and cheaper
2. **Use Sonnet for analysis** - better reasoning
3. **Cache responses** - avoid redundant calls
4. **Set max_tokens** - always limit output length
5. **Implement retries** - handle throttling gracefully
