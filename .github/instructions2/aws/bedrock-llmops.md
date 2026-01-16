# AWS Bedrock LLMOps - MAIP

## Use Cases
- Anomaly detection in transaction streams
- Intelligent alert summarization
- Policy compliance checking
- Natural language telemetry analysis

## Verify Model Access

```bash
aws bedrock list-foundation-models \
  --query "modelSummaries[?contains(modelId, 'claude')].{Id:modelId,Name:modelName}" \
  --output table
```

## Available Models
- `anthropic.claude-3-sonnet-20240229-v1:0` (recommended for MAIP)
- `anthropic.claude-3-haiku-20240307-v1:0` (faster, lower cost)

## Invoke Model - Anomaly Detection

```bash
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-sonnet-20240229-v1:0 \
  --content-type application/json \
  --body '{
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 1024,
    "messages": [
      {
        "role": "user",
        "content": "Analyze this telemetry data for anomalies: CPU: 95%, Memory: 88%, Kafka Lag: 50000, Response Time: 2500ms. Service: order-service. Normal baselines: CPU <70%, Memory <75%, Lag <1000, Response <500ms."
      }
    ]
  }' \
  output.json

cat output.json | jq -r '.content[0].text'
```

## Invoke Model - Alert Summarization

```bash
aws bedrock-runtime invoke-model \
  --model-id anthropic.claude-3-haiku-20240307-v1:0 \
  --content-type application/json \
  --body '{
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 512,
    "messages": [
      {
        "role": "user",
        "content": "Summarize these alerts for on-call engineer: [Alert 1: High CPU on order-service pod-3, Alert 2: Kafka consumer lag increasing, Alert 3: RDS connection pool exhausted, Alert 4: 5xx errors spike on /api/v1/orders]. Provide root cause hypothesis and recommended actions."
      }
    ]
  }' \
  output.json
```

## Python Integration (for Spark/Flink)

```python
import boto3
import json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

def detect_anomaly(telemetry_data):
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',
        contentType='application/json',
        body=json.dumps({
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 1024,
            "messages": [
                {"role": "user", "content": f"Analyze for anomalies: {telemetry_data}"}
            ]
        })
    )
    return json.loads(response['body'].read())['content'][0]['text']
```

## Java Integration (for Spring Boot)

```java
// Add to pom.xml: software.amazon.awssdk:bedrock-runtime

BedrockRuntimeClient bedrock = BedrockRuntimeClient.builder()
    .region(Region.US_EAST_1)
    .build();

InvokeModelRequest request = InvokeModelRequest.builder()
    .modelId("anthropic.claude-3-sonnet-20240229-v1:0")
    .contentType("application/json")
    .body(SdkBytes.fromUtf8String(jsonPayload))
    .build();

InvokeModelResponse response = bedrock.invokeModel(request);
```

## Cost Monitoring

```bash
# Check Bedrock usage (via Cost Explorer)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Bedrock"]}}'
```

## Guardrails (Optional)

```bash
# Create guardrail for PII filtering
aws bedrock create-guardrail \
  --name maip-pii-filter \
  --description "Filter PII from MAIP LLM responses" \
  --blocked-input-messaging "Input contains sensitive data" \
  --blocked-outputs-messaging "Response filtered for compliance"
```
