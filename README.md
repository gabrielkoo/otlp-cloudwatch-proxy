# otlp-cloudwatch-proxy

Lambdaless OTLP proxy to Amazon CloudWatch. Routes OpenTelemetry (OTLP/HTTP) telemetry — metrics, traces, and logs — through API Gateway REST API to CloudWatch OTLP endpoints using [AWS Service Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started-aws-proxy.html) with SigV4 signing.

**No Lambda. No collector. No code.**

```
AI Coding Tool (OTEL SDK)
  ↓ OTLP/HTTP + x-api-key
API Gateway REST API (API Key auth)
  ├→ POST /v1/metrics  → SigV4 → CloudWatch Metrics
  ├→ POST /v1/traces   → SigV4 → X-Ray
  └→ POST /v1/logs     → SigV4 → CloudWatch Logs
```

## One-Click Deploy

CloudWatch OTLP metrics ingestion is in [public preview](https://aws.amazon.com/about-aws/whats-new/2026/04/amazon-cloudwatch-opentelemetry-metrics/) (Apr 2, 2026). Traces and logs OTLP endpoints are GA in most regions. Deploy to a supported region:

| Region | Deploy |
|--------|--------|
| US East (N. Virginia) | [![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/create/review?templateURL=https://raw.githubusercontent.com/gabrielkoo/otlp-cloudwatch-proxy/main/template.yaml&stackName=otlp-cloudwatch-proxy) |
| US West (Oregon) | [![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks/create/review?templateURL=https://raw.githubusercontent.com/gabrielkoo/otlp-cloudwatch-proxy/main/template.yaml&stackName=otlp-cloudwatch-proxy) |
| Asia Pacific (Singapore) | [![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-1#/stacks/create/review?templateURL=https://raw.githubusercontent.com/gabrielkoo/otlp-cloudwatch-proxy/main/template.yaml&stackName=otlp-cloudwatch-proxy) |
| Asia Pacific (Sydney) | [![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=ap-southeast-2#/stacks/create/review?templateURL=https://raw.githubusercontent.com/gabrielkoo/otlp-cloudwatch-proxy/main/template.yaml&stackName=otlp-cloudwatch-proxy) |
| Europe (Ireland) | [![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/create/review?templateURL=https://raw.githubusercontent.com/gabrielkoo/otlp-cloudwatch-proxy/main/template.yaml&stackName=otlp-cloudwatch-proxy) |

> Traces and logs work in additional regions. See [CloudWatch OTLP Endpoints](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-OTLPEndpoint.html) for the full list.

## After Deployment

1. Get your API key:

```bash
# From CloudFormation outputs
aws cloudformation describe-stacks \
  --stack-name otlp-cloudwatch-proxy \
  --query "Stacks[0].Outputs[?OutputKey=='GetApiKeyCommand'].OutputValue" \
  --output text | bash
```

2. Get your endpoint:

```bash
aws cloudformation describe-stacks \
  --stack-name otlp-cloudwatch-proxy \
  --query "Stacks[0].Outputs[?OutputKey=='OtlpEndpoint'].OutputValue" \
  --output text
```

## Configure Your Tools

### Claude Code

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=http/json
export OTEL_EXPORTER_OTLP_ENDPOINT=https://xxx.execute-api.us-west-2.amazonaws.com/prod
export OTEL_EXPORTER_OTLP_HEADERS=x-api-key=your-api-key
export OTEL_SERVICE_NAME=claude-code
```

[Claude Code monitoring docs](https://docs.anthropic.com/en/docs/claude-code/monitoring-usage)

### Claude CoWork (Team & Enterprise)

Configure via Admin Settings → Cowork → Monitoring in [claude.ai](https://claude.ai):

- OTLP endpoint: your APIGW URL
- OTLP protocol: `http/json`
- OTLP headers: `x-api-key=your-api-key`

[CoWork monitoring docs](https://support.claude.com/en/articles/14477985-monitor-claude-cowork-activity-with-opentelemetry)

### GitHub Copilot CLI

```bash
# Copilot CLI 1.0.4+ supports OTel natively
export COPILOT_OTEL_ENDPOINT=https://xxx.execute-api.us-west-2.amazonaws.com/prod
export COPILOT_OTEL_HEADERS=x-api-key=your-api-key
```

[Copilot CLI OTel reference](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-command-reference#opentelemetry-monitoring)

### Gemini CLI

```bash
export GEMINI_CLI_OTEL_EXPORT_ENDPOINT=https://xxx.execute-api.us-west-2.amazonaws.com/prod
```

[Gemini CLI telemetry docs](https://geminicli.com/docs/cli/telemetry/)

### Any OTEL SDK

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=https://xxx.execute-api.us-west-2.amazonaws.com/prod
export OTEL_EXPORTER_OTLP_HEADERS=x-api-key=your-api-key
export OTEL_EXPORTER_OTLP_PROTOCOL=http/json
```

## IAM Policy Details

The execution role follows least-privilege:

- **Metrics**: `cloudwatch:PutMetricData` with `cloudwatch:namespace` condition key — only writes to namespaces you specify (default: `OTLPMetrics,claude-code,copilot-cli,gemini-cli,cursor`)
- **Traces**: `xray:PutTraceSegments` + `xray:PutTelemetryRecords` — X-Ray doesn't support resource-level restrictions
- **Logs**: `logs:PutLogEvents` + `logs:DescribeLogStreams` + `logs:CreateLogStream` — scoped to the specific log group you configure

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `StageName` | `prod` | API Gateway stage name |
| `OtlpLogGroupName` | `otlp-logs` | CloudWatch Logs log group for OTLP logs |
| `MetricNamespaces` | `OTLPMetrics,claude-code,...` | Allowed metric namespaces (comma-separated) |

## Cost

| Component | Cost |
|-----------|------|
| API Gateway | ~$3.50 / million requests |
| CloudWatch Metrics | Free during OTel preview; [standard pricing](https://aws.amazon.com/cloudwatch/pricing/) after GA |
| CloudWatch Logs | [Standard pricing](https://aws.amazon.com/cloudwatch/pricing/) |
| Lambda | $0 (there is none) |

## Gotchas

- **XRay traces need CloudWatch Logs as destination.** Run `aws xray update-trace-segment-destination --destination CloudWatchLogs` before sending traces.
- **REST API only.** HTTP API doesn't support the `AWS` integration type needed for SigV4 service proxying.
- **OTLP JSON recommended.** Set `OTEL_EXPORTER_OTLP_PROTOCOL=http/json`. Protobuf also works but JSON is easier to debug in APIGW execution logs.
- **Region matters.** OTLP metrics are preview-only in 5 regions. Traces and logs are available in most regions.

## References

- [CloudWatch OTLP Endpoints](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-OTLPEndpoint.html)
- [CloudWatch OTel metrics announcement (Apr 2, 2026)](https://aws.amazon.com/about-aws/whats-new/2026/04/amazon-cloudwatch-opentelemetry-metrics/)
- [Introducing OpenTelemetry & PromQL in CloudWatch](https://aws.amazon.com/blogs/mt/introducing-opentelemetry-promql-support-in-amazon-cloudwatch/)
- [API Gateway AWS Service Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/getting-started-aws-proxy.html)

## License

MIT
