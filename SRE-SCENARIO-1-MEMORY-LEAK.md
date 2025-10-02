# Grubify + Azure SRE Agent — Memory Leak Scenario

This walkthrough demonstrates using Azure SRE Agent to detect and mitigate memory leaks automatically.

**⚠️ Prerequisites**: Complete the [README.md deployment guide](./README.md) first to deploy Grubify with both v1/v2 backend images.

**Estimated Cost:** ~$15.40 - $45.40 for complete scenario (mostly SRE Agent costs)

**Scenario**: Switch to API v1 with memory leak → Generate load → SRE Agent detects and mitigates → GitHub issue created

---

## Quick Start

### Prerequisites
- Azure subscription with Contributor/Owner permissions
- **Azure CLI + azd**: 
  ```bash
  az extension add --name containerapp --upgrade
  ```
- **Tools**: `curl`, `jq`, `git`

### 1) Setup Variables
```bash
# Fill these in with your values
SUBSCRIPTION_ID="your-subscription-id"
ENV_NAME="grubify-memory-demo"
RG="rg-${ENV_NAME}"
APP_LOCATION="eastus2"
SRE_AGENT_LOCATION="swedencentral"

# Login and setup
az login --use-device-code
az account set --subscription "$SUBSCRIPTION_ID"
```

### 2) Ensure v1 API is deployed (memory leak version)
```bash
# Navigate to your deployed Grubify project
cd Grubify

# Verify current backend version
az containerapp show --name $(az containerapp list -g "$RG" --query "[?contains(name, 'api')].name" -o tsv) --resource-group "$RG" --query "properties.template.containers[0].image" -o tsv

# Switch to v1 API if not already (memory leak version)
jq '.parameters.apiImageTag.value = "v1"' infra/main.parameters.json > temp.json
mv temp.json infra/main.parameters.json

# Deploy v1 backend
azd deploy

# Verify v1 is running
API_URL=$(azd env get-values | grep API_BASE_URL | cut -d'=' -f2)
echo "API URL: $API_URL"
```

### 3) Create Azure SRE Agent
1. Go to https://aka.ms/sreagent-portal
2. Create agent: `grubify-memory-sre-agent` in Sweden Central
3. Select your resource group (same as `$RG`)
4. Configure privileged permissions
5. In Resource mapping, connect the API container app to your GitHub repo

### 4) Configure Sev3 Alert for Memory

```bash
# Get API container app name
API_APP=$(az containerapp list -g "$RG" --query "[?contains(name, 'api')].name" -o tsv)

# Create memory leak alert with Sev3 priority
az monitor metrics alert create \
  --name "GrubifyAPI-MemoryLeak" \
  --resource-group "$RG" \
  --scopes "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RG}/providers/Microsoft.App/containerApps/${API_APP}" \
  --condition "avg WorkingSetBytes > 536870912" \
  --description "Grubify API memory leak detection - triggers at 512MB" \
  --evaluation-frequency PT1M \
  --window-size PT5M \
  --severity 3 \
  --enabled true

echo "✅ Sev3 memory alert created"
```

---

## Steps

### 5) Generate cart load to trigger memory leak + 500 errors
```bash
# Get the API URL for load testing
API_URL=$(azd env get-values | grep API_BASE_URL | cut -d'=' -f2)
echo "API URL: $API_URL"

# Load test to trigger memory leak (v1 has memory leak in cart operations)
for i in {1..50}; do
    echo "Load batch $i/50"
    
    # Add many items rapidly (triggers memory leak + 500 errors as memory exhausts)
    for j in {1..10}; do
        curl -s -X POST "${API_URL}/api/cart" \
          -H "Content-Type: application/json" \
          -d '{"foodItemId":"item-'$j'","quantity":'$((j%3+1))'}' \
          -w "Status: %{http_code} " > /dev/null &
    done
    wait
    
    # Brief pause, then clear cart (memory leak persists)
    sleep 2
    curl -s -X DELETE "${API_URL}/api/cart" > /dev/null
    
    # Check progress every 10 batches
    (( i % 10 == 0 )) && echo "  → Memory climbing, cart operations may start returning 500s..."
done

echo "🔥 Load test complete. Cart operations will cause 500 errors as memory exhausts"
echo "Wait 5-10 min for Sev3 alert → SRE Agent action"
```

**Timeline**: 0-5 min (memory climbs, 500s start) → 5-10 min (Sev3 alert) → 10-15 min (SRE response)

### 6) Monitor SRE Agent Response
```bash
# Check if alert fired
az monitor metrics alert show \
  --name "GrubifyAPI-MemoryLeak" \
  --resource-group "$RG" \
  --query "description"

```

---

## Expected Results

### GitHub Issue (auto-created)
- **Title**: "Memory leak detected in Grubify API"
- **Labels**: incident, memory-leak, auto-generated  
- **Content**: SRE Agent diagnosis + recommended fixes

### Cost Impact
| Component | Monthly Cost |
|-----------|-------------|
| Container Apps | $15-30 |
| SRE Agent | $15-25 |
| Monitoring | $5-10 |
| **Total** | **$35-65** |

### Key Learning
**Azure SRE Agent** automatically:
1. Detected memory anomaly via Sev3 alert
2. Ran diagnostics on the container
3. Created incident ticket with root cause analysis
4. Provided mitigation recommendations

---

## Cleanup
```bash
# Remove everything
azd down --force --purge
```

**Next**: Try [SRE-SCENARIO-2-BAD-DEPLOYMENT-ROLLBACK.md](./SRE-SCENARIO-2-BAD-DEPLOYMENT-ROLLBACK.md) for payment failure handling!
- **📋 Incident Management**: Automated GitHub integration with detailed reports
- **📊 Continuous Monitoring**: Enhanced alerting with Sev3 priority classification
- **🔄 Learning Loop**: Feedback mechanism for improved future responses

This scenario showcases the power of Azure SRE Agent for automated incident response, combining AI-driven detection with intelligent mitigation strategies for production reliability.
