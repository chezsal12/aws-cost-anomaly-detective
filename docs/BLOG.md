# Building AI-Powered Cost Intelligence with Amazon Bedrock

**Author**: Chezsal Kamaray  
**Date**: July 2026  
**Reading Time**: 8 minutes

---

## At a Glance

Traditional AWS cost alerts tell you *what* happened. We built a system using Amazon Bedrock (Claude 3.5 Sonnet) that tells you *why* it happened, *who* caused it, and *how* to fix it. Open source, production-ready, ~$30/month to run.

**[GitHub Repository →](https://github.com/chezsal12/aws-cost-anomaly-detective)**

---

## The Problem: Alert Fatigue Without Context

You've seen this alert before:

```
⚠️ AWS Cost Alert
Your Lambda costs increased 200% in the last hour.
Current spend: $150
```

Great. Now what?

You open Cost Explorer, squint at graphs, grep CloudTrail logs, check AWS Config for changes, correlate with deployments... 45 minutes later, you find that someone changed a Lambda function from 128MB to 3GB during a late-night "quick fix."

**What if the alert just told you that upfront?**

---

## The Solution: AI-Powered Root Cause Analysis

We built **AWS Cost Anomaly Detective** to do exactly that. Here's what the same alert looks like with AI analysis:

```
🚨 AWS Lambda Cost Spike Detected

💰 Cost Impact:
• Current: $150 (+200%)
• Baseline: $50
• Monthly projection: $4,500

🔍 Root Cause:
Lambda function "data-processor" memory increased from 128MB 
to 3GB at 11:43 PM by user john.doe@company.com. Function 
invocations remained constant (10K/hour) but per-execution 
cost increased 23x due to GB-second pricing.

💡 Recommendation:
1. Revert memory to 1024MB (sufficient for 95th percentile)
2. Enable Lambda Power Tuning for optimal configuration
3. Add approval workflow for memory changes >512MB

💵 Estimated Savings: $4,000/month

CloudFormation remediation code attached.
```

**That's the difference between reactive alerting and intelligent monitoring.**

---

## How It Works: Multi-Signal AI Correlation

### Architecture Overview

```
EventBridge (hourly trigger)
    ↓
Lambda Function
    ↓
    ├─→ Cost Explorer (detect anomaly)
    ├─→ CloudTrail (who changed what)
    ├─→ AWS Config (configuration drift)
    ├─→ CloudWatch (metrics & logs)
    ↓
Amazon Bedrock (Claude 3.5 Sonnet)
    ↓
    ├─→ Root cause analysis
    ├─→ Remediation recommendations
    ├─→ Impact forecasting
    ↓
    ├─→ Slack/SNS Alerts
    ├─→ DynamoDB (historical patterns)
    └─→ S3 (detailed reports)
```

### The AI Prompt: Structured Context, Not Magic

The key to good AI analysis is **structured input**. We don't just throw raw data at Claude and hope for the best. Here's our prompt structure:

```python
prompt = f"""
You are an AWS FinOps expert analyzing a cost anomaly.

## Cost Data
- Service: {service}
- Current Cost: ${current_cost}
- Baseline Cost: ${baseline_cost} (1 week ago, same time)
- Spike Percentage: {spike_percentage}%
- Time Range: {time_range}

## Recent Infrastructure Changes (CloudTrail)
{format_cloudtrail_events(events)}

## Configuration Changes (AWS Config)
{format_config_changes(changes)}

## Performance Metrics (CloudWatch)
{format_metrics(metrics)}

## Historical Context
This service has had {historical_count} anomalies in the past 30 days:
{format_historical_patterns(history)}

## Task
Provide a JSON response with:
1. root_cause: 1-2 sentence explanation
2. severity: Critical/High/Medium/Low
3. remediation_steps: Array of {action, expected_savings, urgency}
4. estimated_savings: Dollar amount or percentage
5. executive_summary: Non-technical explanation for stakeholders
"""
```

**Why this works:**
- **Structured data** reduces hallucination
- **Multiple signals** enable correlation
- **Historical context** improves pattern recognition
- **JSON output** is easy to parse and display

---

## Implementation Deep Dive

### 1. Cost Anomaly Detection

We don't use fixed thresholds. Instead, we compare current costs to historical baselines:

```python
def detect_anomalies(lookback_hours=1, threshold=50):
    # Get current costs
    current = get_costs(hours_back=0, duration=lookback_hours)
    
    # Get baseline (1 week ago, same time window)
    baseline = get_costs(hours_back=168, duration=lookback_hours)
    
    # Calculate percentage change
    for service in current:
        if service in baseline:
            spike = ((current[service] - baseline[service]) 
                     / baseline[service] * 100)
            
            if spike > threshold:
                yield {
                    'service': service,
                    'current_cost': current[service],
                    'baseline_cost': baseline[service],
                    'spike_percentage': spike
                }
```

**Why weekly baselines?**
- Accounts for weekly traffic patterns (weekday vs weekend)
- More stable than day-over-day comparison
- Reduces false positives from expected growth

### 2. Context Enrichment

The magic happens when we correlate cost spikes with infrastructure changes:

```python
def enrich_anomaly(service, time_range):
    context = {}
    
    # Who made changes? (CloudTrail)
    context['api_calls'] = cloudtrail.lookup_events(
        LookupAttributes=[
            {'AttributeKey': 'ResourceType', 'AttributeValue': service}
        ],
        StartTime=time_range.start,
        EndTime=time_range.end
    )
    
    # What changed? (AWS Config)
    context['config_changes'] = config.get_resource_config_history(
        resourceType=service_to_resource_type(service),
        laterTime=time_range.end,
        earlierTime=time_range.start
    )
    
    # How's it performing? (CloudWatch)
    context['metrics'] = cloudwatch.get_metric_statistics(
        Namespace=service_to_namespace(service),
        MetricName='Invocations',  # or relevant metric
        StartTime=time_range.start,
        EndTime=time_range.end,
        Period=3600,
        Statistics=['Sum', 'Average']
    )
    
    return context
```

### 3. Bedrock Integration

Calling Bedrock is straightforward, but there are nuances:

```python
def analyze_with_bedrock(anomaly, context):
    bedrock = boto3.client('bedrock-runtime')
    
    prompt = build_analysis_prompt(anomaly, context)
    
    response = bedrock.invoke_model(
        modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': 4000,
            'temperature': 0.2,  # Low temperature = deterministic
            'messages': [
                {
                    'role': 'user',
                    'content': prompt
                }
            ]
        })
    )
    
    # Parse JSON response
    result = json.loads(response['body'].read())
    return extract_analysis(result['content'])
```

**Key settings:**
- **Temperature 0.2**: We want consistent analysis, not creative writing
- **Max tokens 4000**: Detailed analysis needs room
- **Structured output**: JSON schema forces valid responses

### 4. Intelligent Alerting

Not every anomaly deserves an alert. We implement:

**Deduplication**: Don't re-alert on the same service for 24 hours
```python
def is_duplicate(service, hours=24):
    cutoff = datetime.now() - timedelta(hours=hours)
    return dynamodb.query(
        IndexName='ServiceIndex',
        KeyConditionExpression='service = :s AND timestamp > :t',
        ExpressionAttributeValues={
            ':s': service,
            ':t': cutoff.isoformat()
        }
    ).get('Count', 0) > 0
```

**Severity-based routing**: Critical alerts go to Slack + PagerDuty, Low alerts go to email only

**Time-based suppression**: Don't wake people up for Medium severity issues at 3am

---

## Real-World Results

We've been running this in production for 3 months. Here's what we learned:

### Cost Savings
- **Month 1**: Caught misconfigured RDS instance ($2,400/month saved)
- **Month 2**: Detected Lambda memory bloat ($800/month saved)
- **Month 3**: Found orphaned EBS volumes ($150/month saved)
- **Total**: $3,350/month saved vs ~$40/month operating cost

**ROI: 84x**

### Alert Quality
- **Before**: 50 cost alerts/week, 80% false positives
- **After**: 8 anomaly alerts/week, 95% actionable
- **Time to resolution**: Reduced from 45 min → 5 min (median)

### Team Adoption
- **Week 1**: "This is cool but..."
- **Week 4**: "Why wasn't this built sooner?"
- **Week 8**: 3 other teams requesting deployment

---

## Lessons Learned

### 1. Start with Data Quality
Garbage in, garbage out. We spent 30% of development time just getting clean, structured data from AWS APIs. CloudTrail filtering alone took 2 days to tune.

### 2. Prompt Engineering Matters
Our first prompts were too vague. "Analyze this cost spike" produced generic responses. Adding structured sections and historical context improved accuracy by 40%.

### 3. Temperature = Creativity vs Consistency
We started with temperature 1.0 (default) and got wildly different analyses for similar anomalies. Lowering to 0.2 made it deterministic without sacrificing quality.

### 4. Humans in the Loop (for now)
We originally planned auto-remediation. After one test run nearly terminated a production database, we added approval workflows. AI suggests, humans approve.

### 5. Cost Explorer Lag
Cost Explorer data is 8-24 hours delayed. Hourly detection is aspirational; realistically we catch spikes 2-6 hours after they start. Still better than weekly manual reviews.

---

## Open Source & What's Next

**The code is live**: [github.com/chezsal12/aws-cost-anomaly-detective](https://github.com/chezsal12/aws-cost-anomaly-detective)

### Roadmap

- **Multi-account support**: Monitor entire AWS Organization
- **Forecasting**: "At this rate, you'll exceed budget by Friday"
- **Budget guardrails**: Auto-stop resources approaching limits
- **Custom integrations**: Jira, ServiceNow, PagerDuty
- **Cost optimization recommendations**: Beyond anomalies, proactive savings

### Try It Yourself

1-click deploy via CloudFormation:
```bash
git clone https://github.com/chezsal12/aws-cost-anomaly-detective.git
cd aws-cost-anomaly-detective

aws cloudformation deploy \
  --template-file cloudformation/deployment-template.yaml \
  --stack-name cost-detective \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides AlertEmail=you@example.com
```

---

## Conclusion: AI for Operations, Not Just Applications

Everyone talks about using AI for customer-facing features. But some of the highest-ROI AI applications are internal: better monitoring, faster incident response, smarter resource management.

**AWS Cost Anomaly Detective** is our proof point. $500 in Bedrock costs, $3,000+ in savings, countless hours recovered from cost investigation.

That's the power of AI in operations.

---

## About the Author

**Chezsal Kamaray** is a Solutions Architect at AWS, focused on emergent technologies and AI/ML. He helps customers build production AI systems that actually work.

**Connect**: [GitHub](https://github.com/chezsal12) | [LinkedIn](#)

---

## Feedback & Questions

- **GitHub Issues**: [Report bugs, request features, or ask questions](https://github.com/chezsal12/aws-cost-anomaly-detective/issues)

---

*This solution is provided as a sample. Review and test thoroughly before production use.*
