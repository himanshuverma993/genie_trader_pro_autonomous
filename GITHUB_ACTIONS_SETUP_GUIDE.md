# GitHub Actions Setup Guide: ZhuLinsen/daily_stock_analysis

**Complete step-by-step guide to run stock analysis directly on GitHub without local deployment.**

---

## 📋 Table of Contents

1. [Workflow File Location](#workflow-file-location)
2. [Required GitHub Secrets & Environment Variables](#required-github-secrets--environment-variables)
3. [Manual Trigger via workflow_dispatch](#manual-trigger-via-workflow_dispatch)
4. [Modify Cron Schedule](#modify-cron-schedule)
5. [Configuration Files](#configuration-files)
6. [Quick Start Checklist](#quick-start-checklist)

---

## 1. Workflow File Location

### Primary Workflow File
**Path:** `.github/workflows/00-daily-analysis.yml`

**Source:** [View on GitHub](https://github.com/ZhuLinsen/daily_stock_analysis/blob/main/.github/workflows/00-daily-analysis.yml)

**Current Settings:**
```yaml
name: Daily Stock Analysis

on:
  # Automatic trigger (UTC-based, will adjust for your timezone)
  schedule:
    - cron: '0 10 * * 1-5'     # Monday-Friday, 10:00 UTC = 18:00 Beijing Time
  
  # Manual trigger option (workflow_dispatch)
  workflow_dispatch:
    inputs:
      mode:
        description: 'Run mode'
        default: 'full'
        options:
          - full          # Complete analysis (stocks + market review)
          - market-only   # Market review only
          - stocks-only   # Stock analysis only
      force_run:
        description: 'Force run (skip trading day check)'
        default: false
        type: boolean
```

**Other Workflow Files in Repository:**
- `ci.yml` - Continuous Integration tests
- `docker-publish.yml` - Docker image publishing
- `desktop-release.yml` - Desktop app releases
- `auto-tag.yml` - Automatic version tagging
- `pr-review.yml` - PR review automation

---

## 2. Required GitHub Secrets & Environment Variables

### Quick Setup (Minimum Required)

To get started with **zero cost**, you need:

| Secret Name | Required | Source | Purpose |
|---|---|---|---|
| `STOCK_LIST` | ✅ Yes | Your list | Stock codes to analyze (e.g., `600519,AAPL,00700.HK`) |
| `GEMINI_API_KEY` **OR** `DEEPSEEK_API_KEY` | ✅ Yes | Google/DeepSeek | AI model for analysis |

### Complete Secrets List

#### AI Model Configuration (Pick at least ONE)

| Secret | Type | How to Get | Free Tier |
|---|---|---|---|
| `GEMINI_API_KEY` | String | [Google AI Studio](https://aistudio.google.com) | **Free** 60 calls/min |
| `DEEPSEEK_API_KEY` | String | [DeepSeek Platform](https://platform.deepseek.com) | Free trial available |
| `OPENAI_API_KEY` | String | [OpenAI Dashboard](https://platform.openai.com/api-keys) | Paid $5/month |
| `ANTHROPIC_API_KEY` | String | [Anthropic Console](https://console.anthropic.com) | Free trial available |
| `AIHUBMIX_KEY` | String | [AIHubMix](https://inferera.com/?aff=CfMq) | Free trial available |
| `ANSPIRE_API_KEYS` | String | [Anspire Open](https://open.anspire.cn/?share_code=QFBC0FYC) | Free 30 yuan credit |

**Recommended for beginners:** `GEMINI_API_KEY` (completely free, no credit card required)

#### Data Source Configuration (Optional - Free defaults included)

| Secret | Purpose | How to Get | Free Plan |
|---|---|---|---|
| `TUSHARE_TOKEN` | A-share historical data | [Tushare Pro](https://tushare.pro) | Free (limited calls) |
| `TICKFLOW_API_KEY` | Real-time quotes enhancement | [TickFlow](https://tickflow.org) | Free tier available |
| `LONGBRIDGE_OAUTH_CLIENT_ID` + `LONGBRIDGE_OAUTH_TOKEN_CACHE_B64` | HK/US stock data | [Longbridge OpenAPI](https://open.longbridge.com) | Free tier |

#### News Search Configuration (Optional - enhances news quality)

| Secret | Purpose | How to Get | Free Plan |
|---|---|---|---|
| `TAVILY_API_KEYS` | Stock news search | [Tavily](https://tavily.com/) | Free 100 calls/month |
| `SERPAPI_API_KEYS` | Google search results | [SerpAPI](https://serpapi.com) | Free 100/month |
| `BOCHA_API_KEYS` | Chinese stock news | [Bocha](https://open.bocha.cn/) | Paid |
| `BRAVE_API_KEYS` | Privacy-focused search | [Brave Search](https://brave.com/search/api/) | Free 100/month |

#### Notification Channels (Choose at least ONE to receive results)

| Secret | Setup | How to Get |
|---|---|---|
| `WECHAT_WEBHOOK_URL` | WeChat Work Bot | [Tutorial](https://work.weixin.qq.com/help) |
| `TELEGRAM_BOT_TOKEN` + `TELEGRAM_CHAT_ID` | Telegram Bot | [BotFather](https://t.me/botfather) |
| `DISCORD_WEBHOOK_URL` | Discord Channel | Right-click channel → Integrations → Webhooks |
| `FEISHU_WEBHOOK_URL` + `FEISHU_WEBHOOK_SECRET` | Feishu (DingTalk) | [Feishu Bot Setup](https://open.larkoffice.com) |
| `EMAIL_SENDER` + `EMAIL_PASSWORD` | Email notification | Gmail or your mail provider |
| `PUSHPLUS_TOKEN` | PushPlus service | [PushPlus](https://www.pushplus.plus/) |
| `SLACK_WEBHOOK_URL` | Slack channel | Slack App settings |

#### Run Configuration (Variables)

```
STOCK_LIST = "600519,AAPL,00700.HK,2330.TW"    # Stock codes (variable or secret)
MARKET_REVIEW_REGION = "cn"                     # Market review region (cn/us/hk)
REPORT_TYPE = "simple"                          # Report verbosity (simple/detailed)
REPORT_LANGUAGE = ""                            # Leave empty for auto-detect
TRADING_DAY_CHECK_ENABLED = "true"             # Skip non-trading days
```

---

### How to Add Secrets to GitHub

1. **Navigate to Repository Settings**
   - Go to: `https://github.com/YOUR_USERNAME/daily_stock_analysis/settings/secrets/actions`

2. **Click "New repository secret"**

3. **Add Each Secret:**
   - **Name:** (e.g., `GEMINI_API_KEY`)
   - **Value:** (Paste your actual API key)
   - Click **"Add secret"**

4. **Example Setup (Minimum to start):**
   ```
   STOCK_LIST = 600519,AAPL
   GEMINI_API_KEY = your_gemini_key_here
   TELEGRAM_BOT_TOKEN = your_telegram_bot_token
   TELEGRAM_CHAT_ID = your_telegram_chat_id
   ```

---

## 3. Manual Trigger via workflow_dispatch

### Method 1: GitHub Web UI (Easiest)

1. **Go to Actions Tab**
   - Navigate to: `https://github.com/YOUR_USERNAME/daily_stock_analysis/actions`

2. **Select Workflow**
   - Click **"Daily Stock Analysis"** (or "每日股票分析")

3. **Click "Run workflow"** button (top right)

4. **Fill in Options:**
   ```
   mode: full          # or "market-only" or "stocks-only"
   force_run: false    # Set to true to skip trading day check
   ```

5. **Click "Run workflow"** button

6. **Monitor Execution:**
   - Watch real-time logs in the workflow run
   - Check "Reports" artifact after completion (30-day retention)

### Method 2: GitHub CLI

```bash
# Install GitHub CLI: https://cli.github.com

# Authenticate
gh auth login

# Trigger workflow with default settings
gh workflow run 00-daily-analysis.yml -R YOUR_USERNAME/daily_stock_analysis

# Trigger with custom parameters
gh workflow run 00-daily-analysis.yml \
  -R YOUR_USERNAME/daily_stock_analysis \
  -f mode=market-only \
  -f force_run=true

# Monitor execution
gh run list -R YOUR_USERNAME/daily_stock_analysis --limit 10
```

### Method 3: cURL (REST API)

```bash
curl -X POST \
  https://api.github.com/repos/YOUR_USERNAME/daily_stock_analysis/actions/workflows/00-daily-analysis.yml/dispatches \
  -H "Authorization: token YOUR_GITHUB_PAT" \
  -H "Content-Type: application/json" \
  -d '{
    "ref": "main",
    "inputs": {
      "mode": "full",
      "force_run": false
    }
  }'
```

---

## 4. Modify Cron Schedule

### Current Schedule
```yaml
schedule:
  - cron: '0 10 * * 1-5'
```

**Meaning:**
- Minute: `0` → Top of hour
- Hour: `10` → 10:00 UTC
- Day of month: `*` → Every day
- Month: `*` → Every month
- Day of week: `1-5` → Monday-Friday (0=Sunday)

**Current execution:** **10:00 UTC = 18:00 Beijing Time, Mon-Fri**

### Common Cron Examples

| Time Zone | Time | Cron Expression | Notes |
|---|---|---|---|
| Beijing (UTC+8) | 09:30 AM | `30 1 * * 1-5` | Market open |
| Beijing (UTC+8) | 18:00 | `0 10 * * 1-5` | After market close (current) |
| US Eastern (UTC-5) | 09:30 AM | `30 14 * * 1-5` | Market open |
| US Eastern (UTC-5) | 16:00 | `21 20 * * 1-5` | After market close |
| Singapore (UTC+8) | 20:00 | `12 * * *` | Every 20:00 |
| Daily (any time) | 12:00 PM | `0 12 * * *` | Every day noon UTC |

### Modify Schedule

**Option 1: Edit Workflow File Directly**

1. Go to `.github/workflows/00-daily-analysis.yml`
2. Click ✏️ (Edit file)
3. Find the schedule section:
   ```yaml
   schedule:
     - cron: '0 10 * * 1-5'
   ```
4. Change to your desired cron (example: `'0 9 * * *'` for daily 9:00 UTC)
5. Commit with message: `Update cron schedule`

**Option 2: Using Cron Calculator**

- Visit: [crontab.guru](https://crontab.guru)
- Set your desired time
- Copy the cron expression
- Paste into workflow file

### Cron Syntax Reference

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 6) (Sunday to Saturday)
│ │ │ │ │
│ │ │ │ │
* * * * *
```

**Common Values:**
- `*` = Any value
- `5,10,15` = Multiple specific values
- `0-5` = Range
- `*/15` = Every 15th value

---

## 5. Configuration Files

### Main Configuration: `.env.example`

The `.env.example` file documents all environment variables. When running on GitHub Actions:
- **GitHub Secrets** are injected as environment variables
- **GitHub Variables** (non-sensitive) are also injected
- **Precedence:** `Secrets > Variables > Defaults in .env.example`

### Key Configuration Sections

#### A. Stock Selection
```bash
STOCK_LIST=600519,AAPL,00700.HK,2330.TW
# Format: Comma-separated stock codes
# China (Shanghai): 600xxx, 601xxx, 603xxx
# China (Shenzhen): 000xxx, 002xxx, 300xxx
# Hong Kong: 00700.HK (or hk00700)
# US: AAPL, TSLA
# Japan: 9999.T (or 9999T)
# Taiwan: 2330.TW (or 2330TW)
```

#### B. AI Model Settings (in `.env.example`)
```bash
# Pick ONE of these:
GEMINI_API_KEY=your_key_here              # Google Gemini (FREE)
DEEPSEEK_API_KEY=sk-xxx                  # DeepSeek
OPENAI_API_KEY=sk-xxx                    # OpenAI (Paid)
ANTHROPIC_API_KEY=sk-ant-xxx             # Claude

# Optional: Fine-tune model selection
LITELLM_MODEL=gemini/gemini-2.5-flash    # Specify exact model
LLM_TEMPERATURE=0.7                       # Creativity (0=strict, 2=creative)
```

#### C. Data Sources (optional)
```bash
TUSHARE_TOKEN=your_token_here             # A-share historical data
TICKFLOW_API_KEY=your_key_here            # Real-time quotes
LONGBRIDGE_OAUTH_CLIENT_ID=your_id        # HK/US stocks
```

#### D. News Search (optional)
```bash
TAVILY_API_KEYS=your_key_here
SERPAPI_API_KEYS=your_key_here
BOCHA_API_KEYS=your_key_here
```

#### E. Notifications (required to get results)
```bash
# Telegram example:
TELEGRAM_BOT_TOKEN=123456:ABCDefg
TELEGRAM_CHAT_ID=-1001234567890

# Email example:
EMAIL_SENDER=your_email@gmail.com
EMAIL_PASSWORD=your_app_password
EMAIL_RECEIVERS=recipient@example.com

# WeChat/Feishu/Discord: See full .env.example
```

#### F. Report Settings
```bash
REPORT_TYPE=simple                        # simple or detailed
REPORT_LANGUAGE=en                        # Leave empty for auto-detect
MARKET_REVIEW_ENABLED=true               # Include market review
MARKET_REVIEW_REGION=cn                  # cn/us/hk
```

---

## 6. Quick Start Checklist

### ✅ Step 1: Fork the Repository
```
https://github.com/ZhuLinsen/daily_stock_analysis
Click "Fork" button → Select your account
```

### ✅ Step 2: Enable GitHub Actions
```
Repository → Settings → Actions → General
Enable Actions if disabled
Allow all actions and reusable workflows
```

### ✅ Step 3: Add Required Secrets
```
Settings → Secrets and variables → Actions → New repository secret

Add:
Name: STOCK_LIST
Value: 600519,AAPL

Name: GEMINI_API_KEY  (or DEEPSEEK_API_KEY, OPENAI_API_KEY, etc.)
Value: your_actual_api_key_here

Name: TELEGRAM_BOT_TOKEN (or your notification channel)
Value: your_bot_token

Name: TELEGRAM_CHAT_ID
Value: your_chat_id
```

### ✅ Step 4: Test Manual Trigger
```
Actions → Daily Stock Analysis → Run workflow
Select mode: "full" or "stocks-only"
Click "Run workflow"
Wait 2-5 minutes for execution
Check "Reports" artifact for output
```

### ✅ Step 5: Verify Notifications
```
Check your notification channel (Telegram/Email/WeChat)
You should receive the analysis report
```

### ✅ Step 6: Schedule Automatic Runs (Optional)
```
Edit .github/workflows/00-daily-analysis.yml
Change cron schedule to your preferred time
Commit changes
Workflow will run automatically at scheduled time
```

---

## 📊 Complete Workflow Execution Flow

```
┌─────────────────────────────────────────────────────┐
│ GitHub Actions Trigger                              │
│ (Cron: 10:00 UTC Mon-Fri OR Manual via UI)          │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 1: Environment Setup                           │
│ - Python 3.11 installation                          │
│ - Dependency installation (pip install -r req.txt)  │
│ - Directory creation (data/, logs/, reports/)       │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 2: Configuration Check                         │
│ - Validate GitHub Secrets/Variables                 │
│ - Test AI model connectivity                        │
│ - Verify notification channels                      │
│ - Print configuration summary to logs               │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 3: Data Fetching                               │
│ - Fetch OHLCV data from stock exchanges             │
│ - Retrieve real-time prices                         │
│ - Fetch news and sentiment data                     │
│ - Calculate technical indicators                    │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 4: AI Analysis                                 │
│ - Send data to LLM (Gemini/GPT/DeepSeek)           │
│ - Generate analysis report                          │
│ - Generate market review                            │
│ - Format as Markdown/JSON                           │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Step 5: Notifications                               │
│ - Send to Telegram/Email/WeChat/Discord            │
│ - Log results to reports/                           │
│ - Upload artifacts (30-day retention)              │
└──────────────────┬──────────────────────────────────┘
                   ▼
┌─────────────────────────────────────────────────────┐
│ Completion: Results Available                       │
│ - Notifications received in your channel            │
│ - Reports saved in GitHub Artifacts                │
│ - Logs available for debugging                      │
└─────────────────────────────────────────────────────┘
```

---

## 🔧 Troubleshooting

### Issue: Workflow fails with "API key not configured"
**Solution:** 
- Check Settings → Secrets and variables → Actions
- Ensure `GEMINI_API_KEY` (or your AI key) is added
- Verify secret name matches exactly (case-sensitive)

### Issue: No notification received
**Solution:**
- Verify notification secret is configured (TELEGRAM_BOT_TOKEN, EMAIL_SENDER, etc.)
- Check workflow logs for notification errors
- Test notification channel manually

### Issue: Workflow takes too long / times out
**Solution:**
- Reduce STOCK_LIST (fewer stocks = faster)
- Disable news fetching in report settings
- Reduce REPORT_TYPE to "simple"
- Increase `ANALYSIS_TIMEOUT_MINUTES` variable (default 30)

### Issue: Stock data not found
**Solution:**
- Verify STOCK_LIST format is correct (see Configuration section)
- Check if stock market is open (non-trading days skip automatically)
- Set `force_run: true` to skip trading day check

### Issue: "Repository not found" or "Workflow file not found"
**Solution:**
- Ensure you forked the repository correctly
- Workflow file is `.github/workflows/00-daily-analysis.yml`
- Repository must be public or Actions enabled

---

## 📈 Advanced: Customize Analysis Output

### Modify Report Format

Edit `.github/workflows/00-daily-analysis.yml` environment variables:

```yaml
REPORT_TYPE: simple              # Change to 'detailed'
REPORT_LANGUAGE: en              # English output
MARKET_REVIEW_COLOR_SCHEME: green_up  # Color scheme
SINGLE_STOCK_NOTIFY: false      # Notify per stock
ANALYSIS_DELAY: 0               # Delay in seconds
```

### Add Custom Stock List Based on Conditions

```yaml
# In workflow file, before "Execute Stock Analysis" step:
- name: Generate Dynamic Stock List
  run: |
    echo "STOCK_LIST=600519,300750,000858" >> $GITHUB_ENV
```

### Integrate with External Services

GitHub Actions supports webhooks - you can POST analysis results to your backend:

```yaml
- name: Send to External API
  run: |
    curl -X POST https://your-api.example.com/analysis \
      -H "Content-Type: application/json" \
      -d @reports/analysis_latest.json
```

---

## 📚 Additional Resources

- **Original Repository:** [ZhuLinsen/daily_stock_analysis](https://github.com/ZhuLinsen/daily_stock_analysis)
- **Full Documentation:** [docs/full-guide.md](https://github.com/ZhuLinsen/daily_stock_analysis/tree/main/docs)
- **LLM Configuration:** [docs/LLM_CONFIG_GUIDE.md](https://github.com/ZhuLinsen/daily_stock_analysis/tree/main/docs)
- **Cron Syntax Helper:** [crontab.guru](https://crontab.guru)
- **GitHub Actions Docs:** [docs.github.com/en/actions](https://docs.github.com/en/actions)

---

## ⚡ Getting Started NOW

**5-Minute Quick Start:**

1. Fork: https://github.com/ZhuLinsen/daily_stock_analysis/fork
2. Add Secret `STOCK_LIST` = `600519,AAPL`
3. Add Secret `GEMINI_API_KEY` = (from https://aistudio.google.com)
4. Add Secret `TELEGRAM_BOT_TOKEN` & `TELEGRAM_CHAT_ID`
5. Go to Actions → Run workflow → Run workflow
6. Check Telegram in 2-5 minutes! ✅

---

**Last Updated:** September 2026 | **For:** ZhuLinsen/daily_stock_analysis workflow
