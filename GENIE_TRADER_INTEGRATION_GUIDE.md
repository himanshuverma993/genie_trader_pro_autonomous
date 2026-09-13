# GitHub Actions Integration: daily_stock_analysis → Genie Trader Pro v8.2

**Complete guide for triggering Genie Trader Pro autonomously after stock analysis completes, with real-time data passing and cross-repo orchestration.**

---

## 📋 Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│ Workflow 1: daily_stock_analysis (ZhuLinsen/daily_stock...)  │
│ ├─ Run: 10:00 UTC Mon-Fri                                   │
│ ├─ Analyze stocks → Generate signals                         │
│ ├─ Output: JSON report with Buy/Sell signals                │
│ └─ Trigger: repository_dispatch event                        │
└────────────────┬─────────────────────────────────────────────┘
                 │ (Pass analysis data + tokens)
                 ▼
┌──────────────────────────────────────────────────────────────┐
│ Workflow 2: Genie Trader Pro (your repo)                    │
│ ├─ Listen: repository_dispatch trigger                       │
│ ├─ Receive: Stock signals + market analysis                 │
│ ├─ Execute: genie_trader_pro_v8_2.py (autonomous quant lab) │
│ ├─ Paper Trade: Live hourly daemon                          │
│ └─ Output: Trade journal + performance metrics              │
└──────────────────────────────────────────────────────────────┘
```

---

## 1️⃣ REQUIRED GITHUB SECRETS & ENVIRONMENT VARIABLES

### A. Primary Secrets (daily_stock_analysis repo)

#### AI Configuration (Required - Pick ONE)
```yaml
GEMINI_API_KEY              # Google Gemini (FREE) ✅
DEEPSEEK_API_KEY            # DeepSeek
OPENAI_API_KEY              # OpenAI (Paid)
ANTHROPIC_API_KEY           # Claude
AIHUBMIX_KEY                # AIHubMix (aggregator)
```

#### Stock Data (Required)
```yaml
STOCK_LIST=600519,AAPL,00700.HK,2330.TW  # Stocks to analyze
```

#### Notification/Output (Required for signals)
```yaml
# Choose at least ONE for getting signals:
TELEGRAM_BOT_TOKEN          # Telegram notification
TELEGRAM_CHAT_ID            # Your chat ID
EMAIL_SENDER                # Email
EMAIL_PASSWORD              # Email password
DISCORD_WEBHOOK_URL         # Discord
WECHAT_WEBHOOK_URL          # WeChat
```

#### Data Sources (Optional - Free fallbacks exist)
```yaml
TUSHARE_TOKEN               # A-share data (optional)
TICKFLOW_API_KEY            # Real-time quotes (optional)
TAVILY_API_KEYS             # News search (optional)
SERPAPI_API_KEYS            # Google search (optional)
```

### B. Cross-Repo Orchestration Secrets

**Add these to BOTH repositories:**

#### GitHub Personal Access Token (PAT)
```yaml
GITHUB_PAT_CROSS_REPO
# Scopes needed: repo, workflow, actions:write
# How to create: Settings → Developer settings → Personal access tokens → New token (classic)
# Required permissions:
#   - repo (full control of private repositories)
#   - workflow (update GitHub Action workflows)
#   - actions:write (ability to trigger actions)
```

#### Repository Information (Variables, not secrets)
```yaml
GENIE_TRADER_REPO_OWNER=himanshuverma993              # Your username/org
GENIE_TRADER_REPO_NAME=genie_trader_pro_autonomous   # Repo name
STOCK_ANALYSIS_REPO_OWNER=himanshuverma993
STOCK_ANALYSIS_REPO_NAME=daily_stock_analysis_fork   # Your forked repo
```

#### Communication Channel (for cross-repo status)
```yaml
WEBHOOK_CALLBACK_URL        # Optional: Your backend webhook
```

### C. Complete Secrets Setup Checklist

```
[✅] GEMINI_API_KEY or DEEPSEEK_API_KEY
[✅] STOCK_LIST
[✅] TELEGRAM_BOT_TOKEN + TELEGRAM_CHAT_ID (or Email/Discord)
[✅] GITHUB_PAT_CROSS_REPO (for cross-repo trigger)
[✅] GENIE_TRADER_REPO_OWNER
[✅] GENIE_TRADER_REPO_NAME
[✅] Optional: TUSHARE_TOKEN, TAVILY_API_KEYS, etc.
```

---

## 2️⃣ CROSS-REPO TRIGGERING VIA repository_dispatch

### Step 1: Add Trigger to daily_stock_analysis Workflow

**File:** `.github/workflows/00-daily-analysis.yml`

**Add this step at the END (before artifact upload):**

```yaml
      - name: Generate Analysis Output
        id: analysis_output
        run: |
          # Check if analysis generated reports
          if [ -d "reports" ] && [ "$(ls -A reports 2>/dev/null)" ]; then
            # Extract latest report
            LATEST_REPORT=$(ls -t reports/*.json 2>/dev/null | head -1)
            if [ -n "$LATEST_REPORT" ]; then
              echo "report_path=$LATEST_REPORT" >> $GITHUB_OUTPUT
              echo "analysis_status=success" >> $GITHUB_OUTPUT
              # Pretty print for logging
              echo "📊 Analysis Report Generated:"
              cat "$LATEST_REPORT" | head -50
            fi
          else
            echo "analysis_status=no_reports" >> $GITHUB_OUTPUT
          fi
        
      - name: Trigger Genie Trader Pro Workflow
        if: steps.analysis_output.outputs.analysis_status == 'success'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            
            // Read analysis report
            const reportPath = '${{ steps.analysis_output.outputs.report_path }}';
            let analysisData = {};
            
            try {
              const fileContent = fs.readFileSync(reportPath, 'utf8');
              analysisData = JSON.parse(fileContent);
              console.log('✅ Loaded analysis data:', Object.keys(analysisData));
            } catch (error) {
              console.log('⚠️ Could not read report file:', error.message);
              analysisData = { 
                timestamp: new Date().toISOString(),
                status: 'analysis_completed'
              };
            }
            
            // Prepare trigger payload
            const clientPayload = {
              source: 'daily_stock_analysis',
              timestamp: new Date().toISOString(),
              trigger_mode: 'post_analysis',
              stock_analysis: analysisData,
              run_id: context.runId,
              run_number: context.runNumber
            };
            
            console.log('📤 Dispatching to Genie Trader with payload:');
            console.log(JSON.stringify(clientPayload, null, 2));
            
            // Trigger Genie Trader workflow
            try {
              await github.rest.repos.createDispatchEvent({
                owner: '${{ vars.GENIE_TRADER_REPO_OWNER }}',
                repo: '${{ vars.GENIE_TRADER_REPO_NAME }}',
                event_type: 'stock_analysis_complete',
                client_payload: clientPayload
              });
              console.log('✅ Successfully triggered Genie Trader Pro workflow');
            } catch (error) {
              console.error('❌ Failed to trigger Genie Trader:', error.message);
              throw error;
            }
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_PAT_CROSS_REPO }}
```

### Step 2: Alternative Method - Using REST API (cURL)

**If you prefer REST API over actions/github-script:**

```yaml
      - name: Trigger Genie Trader via REST API
        if: success()
        run: |
          # Get latest report
          REPORT=$(ls -t reports/*.json 2>/dev/null | head -1)
          
          if [ -z "$REPORT" ]; then
            echo "❌ No report found, skipping trigger"
            exit 0
          fi
          
          # Read report content
          REPORT_CONTENT=$(cat "$REPORT" | jq -c '.')
          
          # Create payload
          PAYLOAD=$(cat <<EOF
          {
            "event_type": "stock_analysis_complete",
            "client_payload": {
              "source": "daily_stock_analysis",
              "timestamp": "$(date -Iseconds)",
              "trigger_mode": "post_analysis",
              "stock_analysis": $REPORT_CONTENT,
              "run_id": "${{ github.run_id }}",
              "run_number": "${{ github.run_number }}"
            }
          }
          EOF
          )
          
          echo "📤 Sending payload to Genie Trader:"
          echo "$PAYLOAD" | jq '.'
          
          # Trigger via REST API
          curl -X POST \
            -H "Authorization: token ${{ secrets.GITHUB_PAT_CROSS_REPO }}" \
            -H "Accept: application/vnd.github.v3+json" \
            -d "$PAYLOAD" \
            https://api.github.com/repos/${{ vars.GENIE_TRADER_REPO_OWNER }}/${{ vars.GENIE_TRADER_REPO_NAME }}/dispatches
          
          echo "✅ Trigger dispatched successfully"
```

---

## 3️⃣ GENIE TRADER LISTENER WORKFLOW

### Create New Workflow: genie_trader_listener.yml

**File:** `.github/workflows/genie_trader_listener.yml` (in genie_trader_pro_autonomous repo)

```yaml
name: Genie Trader - Post Analysis Execution

on:
  repository_dispatch:
    types: [stock_analysis_complete]
  workflow_dispatch:
    inputs:
      stock_analysis_json:
        description: 'Stock analysis JSON (optional override)'
        required: false
        type: string
      force_live_mode:
        description: 'Force live mode (paper trading)'
        required: false
        type: boolean
        default: false

env:
  PYTHON_VERSION: '3.11'
  GENIE_TRADER_MODE: paper  # or 'live' for real trading

jobs:
  execute_genie_trader:
    runs-on: ubuntu-latest
    timeout-minutes: 120  # 2 hours for autonomous trading session
    
    steps:
      - name: Checkout Genie Trader Repository
        uses: actions/checkout@v4
        with:
          repository: ${{ github.repository }}
          token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: 'pip'
      
      - name: Install Dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          python -c "import ccxt; print('✅ CCXT import OK')"
          python -c "import xgboost; print('✅ XGBoost import OK')"
          python -c "import torch; print('✅ PyTorch import OK')"
      
      - name: Parse Stock Analysis Input
        id: parse_input
        run: |
          # Create analysis data file from dispatch event
          cat > /tmp/analysis_input.json <<'EOF'
          ${{ toJson(github.event.client_payload.stock_analysis) }}
          EOF
          
          # Validate JSON
          if python -c "import json; json.load(open('/tmp/analysis_input.json'))" 2>/dev/null; then
            echo "✅ Valid analysis JSON received"
            echo "analysis_data_available=true" >> $GITHUB_OUTPUT
          else
            echo "⚠️ No valid analysis data, using defaults"
            echo "analysis_data_available=false" >> $GITHUB_OUTPUT
            cat > /tmp/analysis_input.json <<'EOF'
          {
            "timestamp": "$(date -Iseconds)",
            "source": "manual_trigger",
            "stocks": []
          }
          EOF
          fi
          
          # Display received data
          echo "📊 Received Analysis Data:"
          python -m json.tool /tmp/analysis_input.json | head -30
      
      - name: Prepare Genie Trader Environment
        run: |
          # Create necessary directories
          mkdir -p checkpoints journals logs reports data
          
          # Export analysis data for genie_trader script
          export STOCK_ANALYSIS_DATA="/tmp/analysis_input.json"
          echo "STOCK_ANALYSIS_DATA=$STOCK_ANALYSIS_DATA" >> $GITHUB_ENV
          
          # Set trading parameters from environment or defaults
          export STARTING_EQUITY="${{ secrets.GENIE_STARTING_EQUITY || '10000' }}"
          export RISK_PER_TRADE="${{ secrets.GENIE_RISK_PER_TRADE || '0.01' }}"
          echo "STARTING_EQUITY=$STARTING_EQUITY" >> $GITHUB_ENV
          echo "RISK_PER_TRADE=$RISK_PER_TRADE" >> $GITHUB_ENV
      
      - name: Execute Genie Trader Pro (Paper Trading)
        id: genie_execution
        timeout-minutes: 90
        run: |
          # Run Genie Trader with analysis signals
          python genie_trader_pro_v8_2_autonomous_quant_lab.py \
            --paper \
            --analysis-file "${{ env.STOCK_ANALYSIS_DATA }}" \
            --starting-equity "${{ env.STARTING_EQUITY }}" \
            --risk-per-trade "${{ env.RISK_PER_TRADE }}" \
            2>&1 | tee genie_execution.log
          
          EXECUTION_STATUS=$?
          
          if [ $EXECUTION_STATUS -eq 0 ]; then
            echo "execution_status=success" >> $GITHUB_OUTPUT
            echo "✅ Genie Trader executed successfully"
          else
            echo "execution_status=failed" >> $GITHUB_OUTPUT
            echo "❌ Genie Trader execution failed (exit code: $EXECUTION_STATUS)"
          fi
          
          # Extract key metrics if available
          if [ -f "journals/trade_journal.csv" ]; then
            TRADE_COUNT=$(wc -l < journals/trade_journal.csv)
            echo "trades_count=$TRADE_COUNT" >> $GITHUB_OUTPUT
            echo "📈 Executed $TRADE_COUNT trades"
          fi
      
      - name: Generate Performance Report
        if: always()
        run: |
          # Compile results from execution
          cat > genie_trader_report.json <<'EOF'
          {
            "execution_timestamp": "$(date -Iseconds)",
            "source_trigger": "${{ github.event.client_payload.source || 'manual' }}",
            "execution_status": "${{ steps.genie_execution.outputs.execution_status }}",
            "trades_executed": ${{ steps.genie_execution.outputs.trades_count || 0 }},
            "starting_equity": ${{ env.STARTING_EQUITY }},
            "risk_per_trade": ${{ env.RISK_PER_TRADE }},
            "run_id": "${{ github.run_id }}",
            "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          }
          EOF
          
          echo "📊 Execution Report:"
          python -m json.tool genie_trader_report.json
      
      - name: Upload Execution Artifacts
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: genie-trader-execution-${{ github.run_id }}
          path: |
            journals/
            checkpoints/
            logs/
            genie_trader_report.json
            genie_execution.log
          retention-days: 30
      
      - name: Notify Execution Status
        if: always()
        run: |
          STATUS="${{ steps.genie_execution.outputs.execution_status }}"
          
          if [ "$STATUS" = "success" ]; then
            NOTIFICATION_TITLE="✅ Genie Trader Execution Successful"
            EMOJI="📈"
          else
            NOTIFICATION_TITLE="❌ Genie Trader Execution Failed"
            EMOJI="⚠️"
          fi
          
          MESSAGE="$EMOJI $NOTIFICATION_TITLE\n"
          MESSAGE="$MESSAGE\nTrades Executed: ${{ steps.genie_execution.outputs.trades_count || 0 }}"
          MESSAGE="$MESSAGE\nStarting Equity: ${{ env.STARTING_EQUITY }}"
          MESSAGE="$MESSAGE\nRun: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
          
          echo "📤 Notification would be sent:"
          echo -e "$MESSAGE"
          
          # Optional: Send to Telegram/Email/Webhook
          # echo "$MESSAGE" | telegram-send or mail, etc.
      
      - name: Trigger Next Analysis Cycle (Optional)
        if: steps.genie_execution.outputs.execution_status == 'success'
        uses: actions/github-script@v7
        with:
          script: |
            // Optional: Trigger back to stock analysis or external systems
            console.log('✅ Genie Trader cycle completed successfully');
            console.log('Next automatic analysis: As per daily_stock_analysis schedule');
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

---

## 4️⃣ DATA PASSING: SIGNAL INTEGRATION PATTERNS

### Pattern A: JSON File Passing (Recommended)

```python
# In Genie Trader script: Load analysis signals
import json

def load_stock_analysis(analysis_file_path):
    """Load stock analysis data from daily_stock_analysis workflow"""
    try:
        with open(analysis_file_path, 'r') as f:
            analysis_data = json.load(f)
        
        # Extract buy/sell signals
        signals = {
            'buy_signals': [],
            'sell_signals': [],
            'hold_signals': [],
            'timestamp': analysis_data.get('timestamp')
        }
        
        # Parse analysis results
        for stock in analysis_data.get('stocks', []):
            symbol = stock.get('code')
            decision = stock.get('decision')  # 'BUY', 'SELL', 'HOLD'
            confidence = stock.get('confidence', 0)
            
            if decision == 'BUY' and confidence > 0.65:
                signals['buy_signals'].append({
                    'symbol': symbol,
                    'confidence': confidence,
                    'source': 'daily_stock_analysis'
                })
            elif decision == 'SELL' and confidence > 0.65:
                signals['sell_signals'].append({
                    'symbol': symbol,
                    'confidence': confidence,
                    'source': 'daily_stock_analysis'
                })
        
        return signals
    except Exception as e:
        print(f"⚠️ Error loading analysis: {e}")
        return None

# Usage in main trading loop
analysis_file = os.environ.get('STOCK_ANALYSIS_DATA')
if analysis_file and os.path.exists(analysis_file):
    signals = load_stock_analysis(analysis_file)
    print(f"📊 Loaded {len(signals['buy_signals'])} buy signals from analysis")
    # Use signals in trading logic
```

### Pattern B: Environment Variables

```yaml
# In workflow, export key signals as env vars
- name: Export Analysis Signals as Environment
  run: |
    # Parse JSON and extract key signals
    BUY_SIGNALS=$(jq -r '.buy_signals | @csv' /tmp/analysis_input.json)
    MARKET_SENTIMENT=$(jq -r '.market_sentiment' /tmp/analysis_input.json)
    RISK_LEVEL=$(jq -r '.risk_level' /tmp/analysis_input.json)
    
    echo "BUY_SIGNALS=$BUY_SIGNALS" >> $GITHUB_ENV
    echo "MARKET_SENTIMENT=$MARKET_SENTIMENT" >> $GITHUB_ENV
    echo "RISK_LEVEL=$RISK_LEVEL" >> $GITHUB_ENV
```

### Pattern C: Database/Webhook Callback

```python
# Alternative: Send signals to external database
import requests

def send_signals_to_webhook(signals, webhook_url):
    """Send analysis signals to external service"""
    payload = {
        'timestamp': datetime.now().isoformat(),
        'signals': signals,
        'source': 'daily_stock_analysis'
    }
    
    try:
        response = requests.post(
            webhook_url,
            json=payload,
            headers={'Authorization': f'Bearer {os.environ.get("WEBHOOK_TOKEN")}'},
            timeout=10
        )
        response.raise_for_status()
        print(f"✅ Signals sent to webhook: {response.status_code}")
    except Exception as e:
        print(f"❌ Webhook delivery failed: {e}")

# Usage
webhook_url = os.environ.get('WEBHOOK_CALLBACK_URL')
if webhook_url:
    send_signals_to_webhook(signals, webhook_url)
```

---

## 4️⃣ GITHUB TOKEN SCOPES & PERMISSIONS

### GitHub Personal Access Token (PAT) Configuration

**Go to:** Settings → Developer settings → Personal access tokens → New token (classic)

**Required Scopes:**

| Scope | Permission | Why Needed |
|-------|-----------|-----------|
| `repo` | Full control of private repositories | Read/write access to repo code |
| `workflow` | Update GitHub Action workflows | Modify .github/workflows files |
| `actions:write` | **REQUIRED for repository_dispatch** | Trigger workflows in other repos |

### Minimal Scopes for Cross-Repo Trigger ONLY

If you only need to **trigger workflows** (not modify code):

```
✅ actions:write        # Can trigger workflows
✅ contents:read        # Can read repository files
✅ repository_dispatch  # Can create dispatch events
```

### Token Generation Script

```bash
# Using GitHub CLI (easiest)
gh auth login

# Create token with exact scopes
gh api user/repos --input /dev/stdin <<'EOF'
{
  "scopes": ["repo", "workflow", "actions:write"]
}
EOF

# Or manually at:
# https://github.com/settings/tokens/new
# Scopes: repo, workflow, actions:write
# Name: github-actions-cross-repo
# Expiration: 90 days (for security)
```

### Token Usage in Workflow

```yaml
- name: Trigger External Workflow
  run: |
    curl -X POST \
      -H "Authorization: token ${{ secrets.GITHUB_PAT_CROSS_REPO }}" \
      -H "Accept: application/vnd.github.v3+json" \
      -d '{"event_type":"stock_analysis_complete","client_payload":{}}' \
      https://api.github.com/repos/OWNER/REPO/dispatches
```

### Security Best Practices

1. **Use Personal Access Tokens (PAT), NOT personal GitHub password**
2. **Store as Repository Secret** (Settings → Secrets and variables → Actions)
3. **Minimal scopes** - Only grant what's needed
4. **Short expiration** - Set to 90 days, rotate regularly
5. **Rotate tokens** - Create new one, update secret, delete old one
6. **Monitor usage** - Check https://github.com/settings/tokens/audit

### Token Restrictions (Optional Security Layer)

```yaml
# GitHub CLI to create restricted token
gh api user/repos \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  --input - <<'EOF'
{
  "scopes": ["repo", "workflow", "actions:write"],
  "name": "genie-trader-cross-repo",
  "expires_at": "2026-12-13T00:00:00Z"
}
EOF
```

---

## 🚀 COMPLETE SETUP CHECKLIST

### Phase 1: Prepare Both Repositories

- [ ] Fork/Clone `daily_stock_analysis` repo
- [ ] Create `genie_trader_pro_autonomous` repo (or use existing)
- [ ] Both repos are GitHub-accessible

### Phase 2: Create GitHub PAT

- [ ] Go to: https://github.com/settings/tokens/new
- [ ] Select scopes: `repo`, `workflow`, `actions:write`
- [ ] Set expiration: 90 days
- [ ] Copy token (save securely!)

### Phase 3: Add Secrets to daily_stock_analysis Repo

```
Settings → Secrets and variables → Actions → New repository secret

Add:
[✅] GEMINI_API_KEY = your_api_key
[✅] STOCK_LIST = 600519,AAPL,00700.HK
[✅] TELEGRAM_BOT_TOKEN = your_token
[✅] TELEGRAM_CHAT_ID = your_chat_id
[✅] GITHUB_PAT_CROSS_REPO = your_pat_token
[✅] GENIE_TRADER_REPO_OWNER = your_username
[✅] GENIE_TRADER_REPO_NAME = genie_trader_pro_autonomous
```

### Phase 4: Update daily_stock_analysis Workflow

- [ ] Edit `.github/workflows/00-daily-analysis.yml`
- [ ] Add trigger step (Section 2️⃣ above)
- [ ] Test manual trigger first

### Phase 5: Create Genie Trader Listener Workflow

- [ ] Create `.github/workflows/genie_trader_listener.yml`
- [ ] Add in Genie Trader repo
- [ ] Test manual dispatch first

### Phase 6: Add Secrets to Genie Trader Repo

```
Settings → Secrets and variables → Actions → New repository secret

Add:
[✅] GENIE_STARTING_EQUITY = 10000
[✅] GENIE_RISK_PER_TRADE = 0.01
[✅] GENIE_TRADER_MODE = paper  # or 'live'
```

### Phase 7: Test Full Pipeline

1. Manual trigger daily_stock_analysis workflow
2. Verify report is generated
3. Check if Genie Trader workflow is triggered
4. Monitor Genie Trader execution logs
5. Verify trade journal created

### Phase 8: Set Up Automatic Schedule

- [ ] Configure cron in daily_stock_analysis (default: `0 10 * * 1-5`)
- [ ] Genie Trader will auto-trigger on signal
- [ ] Monitor artifacts for trade results

---

## 🔧 TROUBLESHOOTING

### Problem: "repository_dispatch event not received"

**Solution:**
```yaml
# Check if token has write permissions
- name: Debug Dispatch
  run: |
    curl -i -X GET \
      -H "Authorization: token ${{ secrets.GITHUB_PAT_CROSS_REPO }}" \
      https://api.github.com/user
    # Should return 200 OK with user info
```

### Problem: "Workflow file not found after trigger"

**Solution:**
```yaml
# Ensure listener workflow file exists and is committed
# File must be on main/master branch
# Run: git push origin genie_trader_listener.yml

# Test manual trigger first:
# Actions → Select workflow → Run workflow
```

### Problem: "Analysis data not passed to Genie Trader"

**Solution:**
```python
# In Genie Trader, check environment variable
import os
import json

analysis_file = os.environ.get('STOCK_ANALYSIS_DATA')
print(f"Analysis file: {analysis_file}")

if analysis_file and os.path.exists(analysis_file):
    with open(analysis_file) as f:
        data = json.load(f)
    print(f"✅ Loaded {len(data)} items")
else:
    print(f"⚠️ Analysis file not found")
```

### Problem: "Token authentication failed"

**Solution:**
```bash
# Verify token scopes
curl -H "Authorization: token YOUR_PAT" \
  https://api.github.com/user/scopes

# Should include: repo, workflow, actions

# If missing, regenerate token with correct scopes
```

### Problem: "Workflow runs but no trades executed"

**Solution:**
```yaml
# Check genie_execution.log
- name: Debug Genie Execution
  if: failure()
  run: |
    echo "=== GENIE TRADER LOG ==="
    tail -100 genie_execution.log
    
    echo "=== ANALYSIS INPUT ==="
    cat /tmp/analysis_input.json | head -30
```

---

## 📊 MONITORING & ALERTS

### Set Up Notifications for Cross-Repo Triggers

```yaml
- name: Send Execution Alert
  if: always()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: |
      Genie Trader Execution: ${{ steps.genie_execution.outputs.execution_status }}
      Trades: ${{ steps.genie_execution.outputs.trades_count }}
      Report: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### Dashboard Query (GitHub GraphQL)

```graphql
query {
  repository(owner: "himanshuverma993", name: "genie_trader_pro_autonomous") {
    workflows(first: 1) {
      nodes {
        name
        runs(first: 10) {
          nodes {
            createdAt
            status
            conclusion
            runNumber
          }
        }
      }
    }
  }
}
```

---

## 📚 REFERENCE LINKS

- **GitHub Actions Docs:** https://docs.github.com/en/actions
- **repository_dispatch Event:** https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch
- **GitHub Script Action:** https://github.com/actions/github-script
- **PAT Management:** https://github.com/settings/tokens

---

## ✅ SUCCESS CRITERIA

After setup, you should see:

1. ✅ Daily stock analysis runs at 10:00 UTC
2. ✅ Analysis report generated in `reports/` folder
3. ✅ Genie Trader workflow auto-triggers after 5-30 seconds
4. ✅ Genie Trader receives analysis signals via environment
5. ✅ Paper trading executes with analysis-based signals
6. ✅ Trade journal saved in `journals/trade_journal.csv`
7. ✅ Performance metrics available in artifacts
8. ✅ Full audit trail in GitHub Actions logs

---

**Document Version:** 1.0 | **Last Updated:** September 2026
