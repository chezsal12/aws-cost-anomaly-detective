# Workshop: Deploy Your Cost Detective in 60 Minutes

**Duration**: 60-90 minutes  
**Level**: Intermediate  
**Prerequisites**: AWS account, basic Lambda/Python knowledge

---

## Learning Objectives

By the end of this workshop, you will:
- Deploy AI-powered cost monitoring using Amazon Bedrock
- Understand event-driven serverless architectures
- Implement multi-signal correlation for root cause analysis
- Configure intelligent alerting with Slack and email

---

## Architecture Overview

```
EventBridge → Lambda → Bedrock (Claude) → Alerts
       ↓         ↓
   Schedule   Cost Explorer
              CloudTrail
              AWS Config
              CloudWatch
```

---

## Module 1: Setup (10 minutes)

### 1.1 Verify Amazon Bedrock Access

> **Note**: As of 2026, Bedrock models are automatically enabled when first invoked. No manual activation needed!

Verify you can access Bedrock:

```bash
# List available Claude models
aws bedrock list-foundation-models \
  --region us-east-1 \
  --by-provider anthropic \
  --query 'modelSummaries[?contains(modelId, `claude`)].{ModelId:modelId,Name:modelName}' \
  --output table
```

You should see models like:
- Claude Haiku 4.5
- Claude Sonnet 4
- Others

That's it! Models are ready to use.

### 1.2 Clone Repository

```bash
git clone https://github.com/chezsal12/aws-cost-anomaly-detective.git
cd aws-cost-anomaly-detective
```

---

## Module 2: Deploy Infrastructure (15 minutes)

### 2.1 Deploy via CloudFormation

```bash
aws cloudformation deploy \
  --template-file cloudformation/deployment-template.yaml \
  --stack-name cost-detective-workshop \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    AlertEmail=YOUR_EMAIL@example.com \
    ThresholdPercentage=30 \
    ScheduleExpression="rate(1 hour)"
```

### 2.2 Verify Stack Creation

```bash
aws cloudformation describe-stacks \
  --stack-name cost-detective-workshop \
  --query 'Stacks[0].StackStatus'
```

**Expected Output**: `CREATE_COMPLETE`

### 2.3 Confirm Email Subscription

- Check your email for SNS subscription confirmation
- Click "Confirm subscription"

---

## Module 3: Deploy Lambda Code (10 minutes)

### 3.1 Package Lambda Function

```bash
cd src
pip install -r ../requirements.txt -t .
zip -r ../function.zip .
cd ..
```

### 3.2 Update Lambda Function

```bash
# Get function name from CloudFormation
FUNCTION_NAME=$(aws cloudformation describe-stacks \
  --stack-name cost-detective-workshop \
  --query 'Stacks[0].Outputs[?OutputKey==`LambdaFunctionArn`].OutputValue' \
  --output text | cut -d: -f7)

# Deploy code
aws lambda update-function-code \
  --function-name $FUNCTION_NAME \
  --zip-file fileb://function.zip
```

---

## Module 4: Test the System (15 minutes)

### 4.1 Manual Test Invocation

```bash
aws lambda invoke \
  --function-name $FUNCTION_NAME \
  --payload '{}' \
  response.json

cat response.json
```

### 4.2 Check Logs

```bash
aws logs tail /aws/lambda/$FUNCTION_NAME --follow
```

### 4.3 Generate Test Anomaly (Optional)

Create a cost spike by launching expensive resources:

```bash
# Launch large EC2 instance for testing
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type m5.8xlarge \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Workshop,Value=CostDetective}]'

# Wait 1-2 hours for Cost Explorer to reflect change
# Lambda will detect the anomaly on next run
```

**Remember to terminate**: 
```bash
aws ec2 terminate-instances --instance-ids i-xxxxx
```

---

## Module 5: Configure Slack Alerts (10 minutes)

### 5.1 Create Slack Webhook

1. Go to https://api.slack.com/apps
2. Create New App → From scratch
3. Name: "Cost Detective"
4. Select workspace
5. Features → Incoming Webhooks → Activate
6. Add New Webhook to Workspace
7. Copy webhook URL

### 5.2 Update Lambda Environment

```bash
aws lambda update-function-configuration \
  --function-name $FUNCTION_NAME \
  --environment Variables="{
    SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL,
    THRESHOLD_PERCENTAGE=30,
    LOG_LEVEL=INFO
  }"
```

### 5.3 Test Slack Alert

```bash
aws lambda invoke \
  --function-name $FUNCTION_NAME \
  --payload '{"test": true}' \
  response.json
```

Check your Slack channel for the alert!

---

## Module 6: Understanding the AI Analysis (10 minutes)

### 6.1 How Claude Analyzes Costs

The system sends Claude:
1. **Cost data**: Current vs baseline, spike percentage
2. **CloudTrail events**: Recent API calls (who did what)
3. **Config changes**: Resource modifications
4. **CloudWatch metrics**: Performance data
5. **Historical patterns**: Past anomalies

### 6.2 AI Prompt Structure

```python
# Simplified prompt structure
prompt = f"""
Analyze this AWS cost anomaly:

Cost Data:
- Service: {service}
- Spike: {percentage}%
- Change: ${amount}

Recent Changes:
{cloudtrail_events}

Configuration:
{config_changes}

What caused this spike? Provide:
1. Root cause (1-2 sentences)
2. Remediation steps
3. Estimated savings
"""
```

### 6.3 Review an Analysis

```bash
# Check S3 for detailed reports
BUCKET=$(aws cloudformation describe-stacks \
  --stack-name cost-detective-workshop \
  --query 'Stacks[0].Outputs[?OutputKey==`S3BucketName`].OutputValue' \
  --output text)

aws s3 ls s3://$BUCKET/reports/
```

---

## Module 7: Customization (10 minutes)

### 7.1 Adjust Detection Threshold

Edit `config.yaml.example`:
```yaml
analysis:
  threshold_percentage: 30  # Lower = more sensitive
  min_cost_threshold: 5.0   # Ignore small services
  dedupe_hours: 6           # Re-alert after 6 hours
```

### 7.2 Add Service-Specific Rules

```yaml
alerting:
  rules:
    - service: "AWS Lambda"
      threshold: 20           # More sensitive for Lambda
      severity_override: "High"
    
    - service: "Amazon RDS"
      threshold: 100          # Less sensitive for RDS
      cooldown_hours: 12
```

### 7.3 Enable Multi-Account Monitoring

```yaml
multi_account:
  enabled: true
  role_name: CostDetectiveRole
  account_ids:
    - "123456789012"
    - "210987654321"
```

---

## Module 8: Cleanup (5 minutes)

```bash
# Delete CloudFormation stack
aws cloudformation delete-stack --stack-name cost-detective-workshop

# Verify deletion
aws cloudformation wait stack-delete-complete \
  --stack-name cost-detective-workshop

# Delete S3 bucket (if needed)
aws s3 rb s3://$BUCKET --force
```

---

## 🎓 Key Takeaways

1. **Event-Driven AI**: Claude analyzes costs on-demand, not continuously
2. **Multi-Signal Correlation**: Combines 4+ AWS data sources for context
3. **Intelligent Deduplication**: Prevents alert fatigue
4. **Cost Efficiency**: ~$30-50/month to monitor entire AWS account
5. **Extensible**: Easy to add custom rules and integrations

---

## 📚 Additional Resources

- [Amazon Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [AWS Cost Explorer API](https://docs.aws.amazon.com/aws-cost-management/latest/APIReference/Welcome.html)
- [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)

---

## 🤝 Questions & Feedback

- **GitHub Issues**: https://github.com/chezsal12/aws-cost-anomaly-detective/issues
- **AWS Support**: Your TAM or AWS Support contact

---

**Workshop Complete!** 🎉

You now have AI-powered cost monitoring running in your AWS account.
