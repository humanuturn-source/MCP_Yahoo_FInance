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


📦 Step 2: Install Required Dependencies
You must install the high-level FastMCP server framework along with the Yahoo Finance client data scraper and pandas array processing library.
Run the following command to download the packages into your local system environment:


📦 Step 2: Install Required Dependencies
You must install the high-level FastMCP server framework along with the Yahoo Finance client data scraper and pandas array processing library.
Run the following command to download the packages into your local system environment:

pip3 install fastmcp yfinance pandas


📄 Step 3: Create the Server Script

Save your production server logic exactly as finance_server.py inside your ~/Developer/mcp-portfolio-tracker/ directory. This version enforces absolute path lookups to prevent Cursor or Claude from checking empty background extension directories.
🔍 Step 4: Previewing with MCP Inspector (Optional Development Check)
Before modifying your client applications, verify that the tool handles communication correctly by loading the graphical testing interface:

Save your production server logic exactly as finance_server.py inside your ~/Developer/mcp-portfolio-tracker/ directory. This version enforces absolute path lookups to prevent Cursor or Claude from checking empty background extension directories.
🔍 Step 4: Previewing with MCP Inspector (Optional Development Check)
Before modifying your client applications, verify that the tool handles communication correctly by loading the graphical testing interface:


Save your production server logic exactly as finance_server.py inside your ~/Developer/mcp-portfolio-tracker/ directory. This version enforces absolute path lookups to prevent Cursor or Claude from checking empty background extension directories.
🔍 Step 4: Previewing with MCP Inspector (Optional Development Check)
Before modifying your client applications, verify that the tool handles communication correctly by loading the graphical testing interface:


npx @modelcontextprotocol/inspector python3 finance_server.py


Click "List Tools", type in an entry like AAPL, and click "Run Tool" to confirm everything functions correctly.
⚙️ Step 5: Inject Server Configuration into AI Clients
To allow your LLM interface to query your custom tracker tool on-demand, register your python file path into the system configuration schemas below.
Option A: Claude Desktop Configuration
	1.	Open or create the local configuration file at the following location:
•	macOS: ~/Library/Application Support/Claude/claude_desktop_config.json
	2.	Update the mcpServers tracking object to include your absolute system directories:


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

Option B: Cursor Editor Configuration
	1.	Navigate to Cursor Settings -> Features -> MCP.
	2.	Click "+ Add New MCP Server".
	3.	Input the configuration details:
•	Name: portfolio-tracker
•	Type: stdio
•	Command: python3 /Users/sethrajaram/Developer/mcp-portfolio-tracker/finance_server.py
	4.	Click Save.


🔄 Step 6: Clear Application Cache and Run Prompts
⚠️ CRITICAL TROUBLESHOOTING STEP: If you modify code statements, clicking "Refresh" within the UI panels often fails to clear old background Python processes. You must Completely Quit and Restart Cursor or Claude Desktop to load your modifications.
Once restarted, test your deployment using natural language. You no longer need to reference the tool name explicitly:
•	"Check AAPL"
•	"How is Google looking?"
•	"Give me a status update on NVDA"
The LLM will automatically query your local server, compute your investment metrics, and present the formatted report directly in your chat interface.




