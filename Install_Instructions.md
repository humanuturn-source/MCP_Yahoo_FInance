# Complete Guide: Local Yahoo Finance MCP Server Setup

This guide walks you through setting up a local Model Context Protocol (MCP) server that links **Claude Desktop** or **Cursor** directly to your local stock portfolio files and live market data.

---

### 📥 Prerequisites & System Requirements

Before starting, ensure your local environment has the following components configured:

* **Operating System**: macOS (Intel or Apple Silicon)
* **Core Runtime**: Python 3.10 or higher installed
* **AI Client Application**: Claude Desktop Client or Cursor AI Code Editor installed

---

### 🛠️ Step 1: Initialize Your Project Directory

Create a dedicated project folder to house your python server file alongside your asset configuration tables.

1. Open your terminal app and execute the following commands to initialize your workspace:
   ```bash
   mkdir -p ~/Developer/mcp-portfolio-tracker
   cd ~/Developer/mcp-portfolio-tracker
   ````

	1.	Generate your asset files inside this folder:
•	portfolio.json: This structure records your current asset properties. Create it with the following baseline formatting:

```json
{
  "AAPL": {
    "Quantity": 50,
    "Cost": 300.00
  },
  "GOOGL": {
    "Quantity": 50,
    "Cost": 3500.00
  }
}

```
•	watchlist.json: This structure tracks assets you are monitoring for entry zones:


```json
{
  "NVDA": {
    "Planned_Quantity": 20,
    "Target_Buy_Price": 110.00
  }
}

```


### 📦 Step 2: Install Required Dependencies
You must install the high-level FastMCP server framework along with the Yahoo Finance client data scraper and pandas array processing library.
Run the following command to download the packages into your local system environment:


```bash
pip3 install fastmcp yfinance pandas


```

### 📄 Step 3: Create the Server Script

Save your production server logic exactly as finance_server.py inside your ~/Developer/mcp-portfolio-tracker/ directory. This version enforces absolute path lookups to prevent Cursor or Claude from checking empty background extension directories.

```python
# finance_server.py
from fastmcp import FastMCP
import yfinance as yf
import json
import os

# Initialize the FastMCP server
mcp = FastMCP("Local Portfolio Tracker")

# Force the script to look in its own exact directory on your Mac,
# regardless of what internal system folder Cursor launches the process from.
SCRIPT_DIR = os.path.dirname(os.path.abspath(__file__))
PORTFOLIO_FILE = os.path.join(SCRIPT_DIR, "portfolio.json")
WATCHLIST_FILE = os.path.join(SCRIPT_DIR, "watchlist.json")

def load_json_file(filepath: str) -> dict:
    """Helper function to safely read local JSON files."""
    if not os.path.exists(filepath):
        return {}
    try:
        with open(filepath, "r") as f:
            return json.load(f)
    except Exception:
        return {}

def resolve_to_ticker(input_string: str) -> str:
    """
    Resolves company names like 'Google' to ticker symbols like 'GOOGL'.
    Handles both old and new yfinance search return types safely.
    """
    clean_input = input_string.strip().upper()
    if len(clean_input) <= 5 and clean_input.isalpha():
        return clean_input
    try:
        search_results = yf.Search(input_string, max_results=1).tickers
        if search_results:
            first_match = search_results[0]
            # Handle case where result is a dictionary vs a raw string symbol
            if isinstance(first_match, dict) and 'symbol' in first_match:
                return first_match['symbol'].upper()
            elif isinstance(first_match, str):
                return first_match.upper()
    except Exception:
        pass
    return clean_input

@mcp.tool(name="check_portfolio_and_watch_list")
def check_portfolio_and_watch_list(ticker_or_company: str) -> str:
    """
    CRITICAL PORTFOLIO TOOL: Use this tool to look up stock data. 
    It checks local portfolio.json and watchlist.json properties.
    
    Args:
        ticker_or_company (str): Ticker symbol or company name (e.g. 'AAPL', 'Google').
    """
    try:
        lookup_ticker = resolve_to_ticker(ticker_or_company)
        
        # Download 5 days of history to ensure data exists over weekends/after hours
        df = yf.download(lookup_ticker, period="5d", progress=False)
        
        if df.empty:
            return f"🚨 MCP SERVER ERROR: No market data returned for Ticker: {lookup_ticker}."
            
        # Flatten the index layout so dates become normal data columns
        df_flat = df.reset_index()
        
        # Locate the exact column tracking the closing prices
        close_col = None
        for col in df_flat.columns:
            col_str = str(col).lower()
            if "close" in col_str:
                close_col = col
                break
                
        if close_col is None:
            return f"🚨 MCP SERVER ERROR: Could not find Close price properties. Options: {list(df_flat.columns)}"
            
        # Extract the closing price values directly as a primitive Python list
        price_list = [float(val) for val in df_flat[close_col].dropna().tolist()]
        
        if not price_list:
            return f"🚨 MCP SERVER ERROR: Extraction completed but pricing list came back empty."
            
        current_price = round(price_list[-1], 2)
        
        portfolio = load_json_file(PORTFOLIO_FILE)
        watchlist = load_json_file(WATCHLIST_FILE)
        
        # SMART TICKER ALIGNMENT: Match GOOG vs GOOGL interchangeably across data structures
        matched_portfolio_key = None
        if lookup_ticker in portfolio:
            matched_portfolio_key = lookup_ticker
        elif lookup_ticker == "GOOGL" and "GOOG" in portfolio:
            matched_portfolio_key = "GOOG"
        elif lookup_ticker == "GOOG" and "GOOGL" in portfolio:
            matched_portfolio_key = "GOOGL"

        matched_watchlist_key = None
        if lookup_ticker in watchlist:
            matched_watchlist_key = lookup_ticker
        elif lookup_ticker == "GOOGL" and "GOOG" in watchlist:
            matched_watchlist_key = "GOOG"
        elif lookup_ticker == "GOOG" and "GOOGL" in watchlist:
            matched_watchlist_key = "GOOGL"
        
        # Build the clean, updated Markdown string response
        output = f"### Custom Portfolio Update for {lookup_ticker}\n"
        output += f"* **Current Market Price**: ${current_price} USD\n\n"
        
        # 1. CONDITION: Portfolio Match
        if matched_portfolio_key:
            owned_data = portfolio[matched_portfolio_key]
            qty = owned_data.get("Quantity", 0)
            total_cost = owned_data.get("Cost", 0.0)
            
            current_value = current_price * qty
            profit_loss = round(current_value - total_cost, 2)
            
            output += f"**Portfolio Status**: Owned\n"
            output += f"* **Quantity Owned**: {qty} shares\n"
            output += f"* **Purchased Cost Basis**: ${total_cost:.2f}\n"
            output += f"* **Current Market Value**: ${current_value:.2f}\n"
            output += f"* **Exact Amount of Profit/Loss**: {profit_loss:+.2f} USD\n"
            
        # 2. CONDITION: Watchlist Match
        elif matched_watchlist_key:
            watch_data = watchlist[matched_watchlist_key]
            target_qty = watch_data.get("Planned_Quantity", 0)
            target_price = watch_data.get("Target_Buy_Price", 0.0)
            
            # Total amount required to purchase target shares at target price
            total_required = round(target_price * target_qty, 2)
            
            output += f"**Watchlist Status**: Monitored\n"
            output += f"* **Planned Quantity**: {target_qty} shares\n"
            output += f"* **Target Buy Price**: ${target_price:.2f}\n"
            output += f"* **Total Amount Required to Purchase**: ${total_required:.2f} USD\n"
            
            # Highlight buy zone metrics
            if current_price <= target_price:
                output += f"* **Alert**: 🟢 Stock is ready to buy! Current price is at or below target.\n"
            else:
                output += f"* **Alert**: 🔴 Price is currently higher than your target by ${(current_price - target_price):.2f}.\n"
                
        # 3. CONDITION: Neither List Match
        else:
            output += f"**Status**: This stock is not in your portfolio or watch list.\n"
            
        return output
        
    except Exception as e:
        return f"🚨 Python Runtime Exception inside MCP Server: {str(e)}"

if __name__ == "__main__":
    mcp.run()
```

### 🔍 Step 4: Previewing with MCP Inspector (Optional Development Check)
Before modifying your client applications, verify that the tool handles communication correctly by loading the graphical testing interface:

Save your production server logic exactly as finance_server.py inside your ~/Developer/mcp-portfolio-tracker/ directory. 

This version enforces absolute path lookups to prevent Cursor or Claude from checking empty background extension directories.

```bash
npx @modelcontextprotocol/inspector python3 finance_server.py

```

Click "List Tools", type in an entry like AAPL, and click "Run Tool" to confirm everything functions correctly.

### ⚙️ Step 5: Inject Server Configuration into AI Clients
To allow your LLM interface to query your custom tracker tool on-demand, register your python file path into the system configuration schemas below.

Option A: Claude Desktop Configuration
	1.	Open or create the local configuration file at the following location:
•	macOS: ~/Library/Application Support/Claude/claude_desktop_config.json
	2.	Update the mcpServers tracking object to include your absolute system directories:


```json
{
  "mcpServers": {
    "portfolio-tracker": {
      "command": "python3",
      "args": [
        "/Users/sethrajaram/Developer/mcp-portfolio-tracker/finance_server.py"
      ]
    }
  }
}

```

Option B: Cursor Editor Configuration
	1.	Navigate to Cursor Settings -> Features -> MCP.
	2.	Click "+ Add New MCP Server".
	3.	Input the configuration details:
•	Name: portfolio-tracker
•	Type: stdio
•	Command:

```bash
python3 /Users/sethrajaram/Developer/mcp-portfolio-tracker/finance_server.py

```
	4.	Click Save.


🔄 Step 6: Clear Application Cache and Run Prompts
⚠️ CRITICAL TROUBLESHOOTING STEP: If you modify code statements, clicking "Refresh" within the UI panels often fails to clear old background Python processes. You must Completely Quit and Restart Cursor or Claude Desktop to load your modifications.
Once restarted, test your deployment using natural language. You no longer need to reference the tool name explicitly:
•	"Check AAPL"
•	"How is Google looking?"
•	"Give me a status update on NVDA"
The LLM will automatically query your local server, compute your investment metrics, and present the formatted report directly in your chat interface.




