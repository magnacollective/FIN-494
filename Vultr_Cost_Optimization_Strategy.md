# Vultr Cost Optimization Strategy
## Sentiment Cascade Detection Platform - $250 Budget

**Project Duration:** 4 weeks minimum (Nov 4 - Dec 1)
**Available Budget:** $250 Vultr credits
**Goal:** Maximize runtime while maintaining performance

---

## 🎯 Recommended Strategy: Smart GPU Scheduling

### Total Estimated Cost: **$75-95 for 4 weeks**

This leaves you **$155-175 in reserve** for extensions or emergencies.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│  ALWAYS-ON: CPU Compute Instance ($24/month)        │
│  - 4 vCPU, 8GB RAM                                  │
│  - Data Collector (Python)                          │
│  - API Gateway (Node.js)                            │
│  - Cascade Detector (Node.js)                       │
│  - Alert Engine (Node.js)                           │
│  - Redis (in-memory)                                │
└─────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  ON-DEMAND: GPU Instance ($0.45/hour)               │
│  - NVIDIA A16 16GB                                  │
│  - Runs ONLY during sentiment processing            │
│  - Auto-start via cron → Process → Auto-shutdown    │
│  - ~10 minutes actual usage per hour                │
│  - Effective cost: ~$0.075/hour                     │
└─────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────┐
│  ALWAYS-ON: Managed PostgreSQL ($15/month)          │
│  - 1 vCPU, 2GB RAM, 40GB SSD                        │
│  - Stores all sentiment, cascade, and alert data    │
└─────────────────────────────────────────────────────┘
```

---

## 💰 Detailed Cost Breakdown

### Always-On Services (24/7)

| Service | Specs | Monthly Cost | 4-Week Cost |
|---------|-------|--------------|-------------|
| **Compute Instance** | 4 vCPU, 8GB RAM | $24 | $22 |
| **PostgreSQL** | 1 vCPU, 2GB, 40GB | $15 | $14 |
| **Redis** | Self-hosted on compute | $0 | $0 |
| **Bandwidth** | ~100GB (generous estimate) | $2 | $2 |
| **Subtotal** | | **$41/month** | **$38** |

### On-Demand GPU Service

| Usage Pattern | Hours/Day | Daily Cost | 4-Week Cost |
|---------------|-----------|------------|-------------|
| Every 30 min, 10 min runtime | 0.33 hours | $0.15 | $4.20 |
| **Safety buffer (25%)** | | | **+$1.05** |
| **GPU Subtotal** | | | **$5.25/week** |
| **4 Weeks Total** | | | **$21** |

### Alternative: 24/7 GPU Instance (Not Recommended)

| Service | Monthly Cost | 4-Week Cost | Utilization |
|---------|--------------|-------------|-------------|
| GPU Instance (always on) | $324/month | $298 | **2% actual GPU usage** |

**Analysis:** Running GPU 24/7 wastes **$277** for 4 weeks compared to on-demand strategy.

---

## 🚀 Implementation Plan

### 1. Initial Setup (Day 1-2)

```bash
# Create compute instance
vultr-cli instance create \
  --plan vc2-4c-8gb \
  --region ewr \
  --os "Ubuntu 22.04" \
  --label "sentiment-compute"

# Create managed PostgreSQL
vultr-cli database create \
  --engine postgres \
  --plan vultr-dbaas-startup-cc-1-55-1 \
  --region ewr \
  --label "sentiment-db"

# Create GPU instance (ON-DEMAND ONLY)
vultr-cli instance create \
  --plan vhp-2c-16gb-nvidia-rtx-a4000 \
  --region ewr \
  --os "Ubuntu 22.04" \
  --label "sentiment-gpu-worker"
```

### 2. Smart GPU Scheduling Script

Create `/opt/sentiment/gpu_processor.sh` on compute instance:

```bash
#!/bin/bash

# Start GPU instance
GPU_INSTANCE_ID="your-gpu-instance-id"
echo "[$(date)] Starting GPU instance..."
vultr-cli instance start $GPU_INSTANCE_ID

# Wait for instance to be ready
sleep 60

# Get GPU instance IP
GPU_IP=$(vultr-cli instance get $GPU_INSTANCE_ID --json | jq -r '.main_ip')

# Copy data to GPU instance
echo "[$(date)] Transferring data..."
rsync -avz /tmp/posts_to_process.json root@$GPU_IP:/app/data/

# Run FinBERT processing remotely
echo "[$(date)] Processing with FinBERT..."
ssh root@$GPU_IP "cd /app && python3 process_sentiment.py"

# Download results
echo "[$(date)] Retrieving results..."
rsync -avz root@$GPU_IP:/app/output/sentiment_scores.json /tmp/

# Stop GPU instance
echo "[$(date)] Stopping GPU instance..."
vultr-cli instance stop $GPU_INSTANCE_ID

echo "[$(date)] GPU processing complete!"
```

### 3. Cron Schedule (on compute instance)

```cron
# Process sentiment every 30 minutes
0,30 * * * * /opt/sentiment/gpu_processor.sh >> /var/log/sentiment_gpu.log 2>&1

# Daily baseline calculation (runs at midnight, 5 min GPU time)
0 0 * * * /opt/sentiment/baseline_calculator.sh
```

---

## 📊 Cost Comparison Matrix

| Strategy | 4-Week Cost | Pros | Cons | Recommendation |
|----------|-------------|------|------|----------------|
| **Smart GPU Scheduling** | **$59** | Maximum savings, ~75% reserve | Slightly complex setup | ⭐ **BEST** |
| CPU-Only FinBERT | $63 | Simple, no GPU needed | 2-3x slower processing | ⭐ Good backup |
| 24/7 Mid-tier GPU | $130 | Simple, fast | Wastes $70+ | ❌ Not recommended |
| 24/7 High-tier GPU | $180 | Overkill performance | Wastes $120+ | ❌ Avoid |

---

## 🎓 Academic Project Optimizations

### Data Collection Frequency

**Default:** Every 30 minutes
- Cost impact: 48 GPU runs/day = $7.20/day

**Optimization 1:** Reduce to 60 minutes during off-hours
- Midnight-6am: Process every 60 min (6 fewer runs)
- Cost savings: ~$1/day = **$28 saved over 4 weeks**

**Optimization 2:** Skip weekends for stocks (crypto continues)
- Stocks: Mon-Fri only
- Crypto: 24/7
- Cost savings: **$10-15 over 4 weeks**

```python
# Smart scheduling logic
def should_process(ticker, current_time):
    if ticker.asset_type == 'stock':
        # Skip stock processing on weekends
        if current_time.weekday() >= 5:  # Sat/Sun
            return False
        # Skip outside market hours + 2hr buffer
        if current_time.hour < 7 or current_time.hour > 18:
            return False

    # Crypto processes 24/7 but reduce frequency at night
    if current_time.hour >= 0 and current_time.hour < 6:
        # Process every 60 min instead of 30 min
        if current_time.minute != 0:
            return False

    return True
```

### FinBERT Optimization

**Strategy:** Selective FinBERT usage

```python
def analyze_sentiment(text, source, engagement):
    # VADER is free and fast - use for all
    vader_score = vader.polarity_scores(text)['compound']

    # FinBERT is expensive - use selectively
    use_finbert = (
        source == 'news' or  # Always for news
        engagement > 100 or  # High-engagement posts only
        contains_complex_financial_terms(text)  # Technical analysis
    )

    if use_finbert:
        finbert_score = finbert_pipeline(text)[0]['score']
        return {'final': finbert_score, 'vader': vader_score}

    return {'final': vader_score, 'finbert': None}
```

**Impact:** Reduces GPU workload by 60-70%
**Savings:** ~$13 over 4 weeks

---

## 🔧 Alternative: CPU-Only Approach (Backup Plan)

If GPU scheduling is too complex, use CPU-only strategy:

### Single Instance Setup

```
1 High-Memory Compute Instance:
   - 6 vCPU, 16GB RAM ($48/month = $44 for 4 weeks)
   - Run FinBERT on CPU (PyTorch CPU builds)
   - Processing time: 15-20 min per batch (acceptable)

1 Managed PostgreSQL:
   - 1 vCPU, 2GB RAM ($15/month = $14 for 4 weeks)

Total: $58 for 4 weeks
```

**When to use:**
- You want simplicity over optimization
- 15-20 min processing time is acceptable
- You prefer single instance management

**FinBERT CPU Performance:**
- Batch size: 8 posts at a time
- ~100-150 posts per 30-min window
- Processing: 12-18 minutes on 6 vCPU
- Still completes before next collection cycle

---

## 📅 Budget Timeline

### Conservative Estimate (Smart GPU Strategy)

| Week | Always-On | GPU Usage | Weekly Total | Cumulative |
|------|-----------|-----------|--------------|------------|
| Week 1 | $9.50 | $5.25 | $14.75 | $14.75 |
| Week 2 | $9.50 | $5.25 | $14.75 | $29.50 |
| Week 3 | $9.50 | $5.25 | $14.75 | $44.25 |
| Week 4 | $9.50 | $5.25 | $14.75 | $59.00 |
| **Setup/Buffer** | - | - | $10.00 | $69.00 |

**Remaining Credit: $181** (72% of budget)

### What Can You Do With Remaining $181?

1. **Extend project 10 more weeks** ($150) - Great for thesis work
2. **Add development/staging environment** ($24) - Test changes safely
3. **Scale to 30 tickers** (+$20 GPU) - More comprehensive research
4. **Upgrade database for faster queries** (+$15) - Better performance
5. **Keep as emergency fund** - Handle unexpected spikes

---

## 🚨 Cost Monitoring & Alerts

### Set Up Vultr Billing Alerts

```bash
# Set alert at 25% budget ($62.50)
vultr-cli billing alert create --threshold 62.50

# Set alert at 50% budget ($125)
vultr-cli billing alert create --threshold 125

# Set alert at 75% budget ($187.50)
vultr-cli billing alert create --threshold 187.50
```

### Daily Cost Tracking Script

Create `/opt/sentiment/track_costs.sh`:

```bash
#!/bin/bash

CURRENT_SPEND=$(vultr-cli billing history --json | jq '[.[] | .amount] | add')
BUDGET=250
REMAINING=$(echo "$BUDGET - $CURRENT_SPEND" | bc)
PERCENT_USED=$(echo "scale=2; ($CURRENT_SPEND / $BUDGET) * 100" | bc)

echo "=== Vultr Budget Status ==="
echo "Total Budget: \$$BUDGET"
echo "Current Spend: \$$CURRENT_SPEND"
echo "Remaining: \$$REMAINING"
echo "Used: $PERCENT_USED%"

# Send to monitoring system or email
curl -X POST https://your-dashboard/api/budget \
  -H "Content-Type: application/json" \
  -d "{\"remaining\": $REMAINING, \"percent_used\": $PERCENT_USED}"
```

### Weekly Cost Review Checklist

- [ ] Check Vultr billing dashboard
- [ ] Review GPU instance uptime (should be <1% of week)
- [ ] Verify compute instance is properly stopping GPU instances
- [ ] Check for any zombie instances
- [ ] Review bandwidth usage (should be <100GB/week)
- [ ] Confirm database size (should grow ~500MB/week)

---

## 💡 Advanced Optimization Ideas

### 1. Spot/Preemptible GPU Instances (if Vultr offers)

- Save 60-70% on GPU costs
- Risk: Instance can be terminated
- Mitigation: Queue-based processing with retry logic

### 2. Batch Processing Windows

Instead of every 30 minutes:
```
Market Hours (9am-4pm ET): Every 30 min
After Hours (4pm-9pm ET): Every 60 min
Overnight (9pm-9am ET): Every 2 hours
Weekends: Every 2 hours (crypto only)
```

**Savings:** ~$15-20 over 4 weeks

### 3. Hybrid Cloud Strategy

- Vultr: Data collection + database + API
- Hugging Face Inference API: FinBERT processing (FREE tier)
- Savings: Eliminate GPU instance entirely

**Hugging Face Free Tier:**
- 30,000 characters/day
- Your usage: ~15,000 chars/day (150 posts × 100 chars)
- Cost: $0 (within free tier)

### 4. Pre-computed Embeddings

- Generate FinBERT embeddings once for historical data
- Store embeddings in database
- Only process NEW posts with GPU
- Reduces GPU time by 80% for baseline calculations

---

## 🎯 Final Recommendation

### For Maximum Budget Efficiency:

**Use Smart GPU Scheduling + Hugging Face Hybrid**

```
Vultr Services:
- 1 Compute Instance (4 vCPU, 8GB): $22 for 4 weeks
- 1 Managed PostgreSQL (1 vCPU, 2GB): $14 for 4 weeks
- Bandwidth: $2 for 4 weeks

Hugging Face:
- FinBERT Inference API: FREE (within quota)

Total: $38 for 4 weeks
Remaining: $212 (85% of budget)
```

### Implementation Priority:

**Week 1:** Start with CPU-only approach ($58 budget)
- Prove the pipeline works
- Collect initial data
- Validate FinBERT is necessary

**Week 2-4:** Migrate to optimized GPU or Hugging Face
- If FinBERT CPU is too slow: Deploy smart GPU scheduling
- If FinBERT CPU is acceptable: Keep it simple
- If Hugging Face API works: Switch to $38 strategy

---

## 📞 Quick Reference

### Cost Commands

```bash
# Check current month spending
vultr-cli billing history

# List all instances and costs
vultr-cli instance list --json | jq '.[] | {label, plan, monthly_cost}'

# Stop GPU instance immediately
vultr-cli instance stop <gpu-instance-id>

# Delete unused instances
vultr-cli instance delete <instance-id>
```

### Emergency Cost Reduction

If budget is running low:

1. **Immediate (-$20/week):** Reduce collection to every 60 min
2. **Quick (-$15/week):** Switch to VADER-only (no FinBERT)
3. **Moderate (-$10/week):** Skip weekends entirely
4. **Nuclear (-$30/week):** Pause data collection, analyze existing data

---

## ✅ Success Criteria

Your strategy is working if:

- [ ] Week 1 spending: <$20
- [ ] Week 2 spending: <$40 cumulative
- [ ] Week 3 spending: <$60 cumulative
- [ ] GPU instance uptime: <2% of total time
- [ ] Final 4-week cost: <$80
- [ ] Remaining credits: >$170

---

**Strategy Version:** 1.0
**Last Updated:** November 12, 2024
**Author:** Claude Code
**Recommended Approach:** Smart GPU Scheduling + Hugging Face Hybrid ($38-59 for 4 weeks)
