# Plan: Replace Chat with Claude Agent SDK Trading Assistant

## Context

The current `chat` command uses a keyword-based REPL (`TradingOrchestrator` + `InteractiveSession`) that counts keyword matches to route queries. It has no LLM reasoning, no conversation memory between queries, and can't handle ambiguous questions.

Replace it with the **Claude Agent SDK for Python** (`claude-agent-sdk` pip package, v0.1.31+). The existing analysis agents become in-process MCP tools that Claude can call. The old keyword system is removed entirely (no fallback — require `ANTHROPIC_API_KEY`).

## Architecture

```
User types query in terminal REPL
  -> TradingChatEngine.process_query(user_input)
     -> client.query(user_input)        # sends to Claude via Agent SDK
     -> async for msg in client.receive_response():
          AssistantMessage  -> stream text to Rich console
          StreamEvent       -> real-time token-by-token display
          ResultMessage     -> show cost, end of turn
     -> Claude calls in-process MCP tools as needed:
          mcp__trading__get_technical_analysis
          mcp__trading__analyze_options
          mcp__trading__scan_market
          mcp__trading__get_news_sentiment
          mcp__trading__assess_risk
          mcp__trading__get_quote
          mcp__trading__get_economic_indicators
     -> Claude synthesizes results into coherent response
  -> Conversation history maintained automatically by ClaudeSDKClient
  -> User asks follow-up -> Claude has full context from all prior turns
```

## Step 1: NEW `src/chat/__init__.py`

Barrel export:
```python
from src.chat.engine import TradingChatEngine
```

## Step 2: NEW `src/chat/engine.py` — Claude-powered chat engine

### Imports (confirmed from SDK docs)

```python
from claude_agent_sdk import (
    ClaudeSDKClient,
    ClaudeAgentOptions,
    create_sdk_mcp_server,
    tool,
    AssistantMessage,
    TextBlock,
    StreamEvent,
    ResultMessage,
)
```

### Tool definitions (7 tools)

Each tool is an `async def` decorated with `@tool(name, description, input_schema)`.
Each receives `args: dict[str, Any]` and returns `{"content": [{"type": "text", "text": "..."}]}`.
Tools share a module-level `_resources` dict initialized by the engine.

| Tool | Wraps | Input Schema |
|------|-------|-------------|
| `get_technical_analysis` | `TechnicalAnalystAgent.full_analysis()` | `{"symbol": str, "timeframes": list}` — timeframes defaults to `["1d"]` |
| `analyze_options` | `OptionsStrategistAgent.recommend_strategy()` | `{"symbol": str, "outlook": str, "timeframe_days": int}` |
| `scan_market` | `ScannerAgent.scan()` | `{"preset": str, "min_price": float, "max_price": float, "limit": int}` |
| `get_news_sentiment` | `SentimentAnalystAgent.analyze()` | `{"symbol": str, "hours_back": int}` |
| `assess_risk` | `RiskAssessorAgent.assess_trade()` | `{"entry": float, "stop": float, "target": float, "symbol": str}` |
| `get_quote` | `MarketDataAggregator.get_quote()` | `{"symbol": str}` |
| `get_economic_indicators` | `FREDClient.get_economic_data()` | `{}` (no params) |

Example tool implementation pattern:
```python
@tool(
    "get_technical_analysis",
    "Multi-timeframe technical analysis with trend, signals, support/resistance levels",
    {"symbol": str, "timeframes": list},
)
async def get_technical_analysis(args: dict[str, Any]) -> dict[str, Any]:
    symbol = args["symbol"].upper().strip()
    timeframes = args.get("timeframes", ["1d"])
    try:
        agent = _resources["technical_agent"]
        report = await agent.full_analysis(symbol, timeframes=timeframes)
        # Reuse orchestrator's formatting logic
        text = _format_technical_report(report)
        return {"content": [{"type": "text", "text": text}]}
    except Exception as e:
        return {"content": [{"type": "text", "text": f"Error analyzing {symbol}: {e}"}]}
```

The formatting functions (`_format_technical_report`, `_format_options_recommendations`, etc.) are extracted from `TradingOrchestrator`'s private methods and reused as module-level functions in `engine.py`.

### MCP server creation

```python
def create_trading_server():
    return create_sdk_mcp_server(
        name="trading",
        tools=[
            get_technical_analysis,
            analyze_options,
            scan_market,
            get_news_sentiment,
            assess_risk,
            get_quote,
            get_economic_indicators,
        ],
    )
```

### System prompt

```python
SYSTEM_PROMPT = """\
You are an expert stock and options trading analyst. You help traders analyze \
market conditions, evaluate strategies, and manage risk. You have access to \
real-time market data and analysis tools.

RULES:
- This is an ADVISORY system only. You provide analysis, never trade execution.
- Always state your confidence level when making assessments.
- When presenting numerical data, cite which tool provided it.
- If asked about something outside your tools' capabilities, say so clearly.
- Never fabricate price data, indicators, or statistics. Only report what the \
tools return.
- Include brief risk disclaimers when recommending specific strategies.
- Lead with your conclusion, then present supporting data.

You can use multiple tools in a single response to build comprehensive analysis. \
For example, if asked "Should I buy NVDA?", you might check technical analysis, \
news sentiment, and options flow together.

Available tools: get_technical_analysis, analyze_options, scan_market, \
get_news_sentiment, assess_risk, get_quote, get_economic_indicators.\
"""
```

### TradingChatEngine class

```python
class TradingChatEngine:
    def __init__(self, model: str = "sonnet") -> None:
        self._model = model
        self._client: ClaudeSDKClient | None = None
        self._resources: dict[str, Any] = {}

    async def initialize(self) -> None:
        """Init DataCache, aggregator, analyzers, agents. Store in _resources."""
        cache = DataCache()
        await cache.initialize()
        self._resources["cache"] = cache

        aggregator = MarketDataAggregator(cache=cache)
        self._resources["aggregator"] = aggregator

        # Create analyzers and agents (same pattern as TradingOrchestrator.__init__)
        tech_analyzer = TechnicalAnalyzer(data_client=aggregator)
        self._resources["technical_agent"] = TechnicalAnalystAgent(analyzer=tech_analyzer)

        options_calc = OptionsCalculator()
        strategy_analyzer = StrategyAnalyzer()
        self._resources["options_agent"] = OptionsStrategistAgent(
            data_client=aggregator, calculator=options_calc,
            strategy_analyzer=strategy_analyzer,
        )

        self._resources["scanner"] = ScannerAgent(data_client=aggregator)

        config = TradingConfig()
        self._resources["risk_agent"] = RiskAssessorAgent(config=config)

        # Sentiment agent is lazily created (needs FINNHUB_API_KEY)
        self._resources["sentiment_agent"] = None

        # Wire module-level _resources for tool functions
        global _resources
        _resources = self._resources

        # Create SDK client
        trading_server = create_trading_server()
        options = ClaudeAgentOptions(
            model=self._model,
            system_prompt=SYSTEM_PROMPT,
            max_turns=25,
            permission_mode="bypassPermissions",
            mcp_servers={"trading": trading_server},
            allowed_tools=[
                "mcp__trading__get_technical_analysis",
                "mcp__trading__analyze_options",
                "mcp__trading__scan_market",
                "mcp__trading__get_news_sentiment",
                "mcp__trading__assess_risk",
                "mcp__trading__get_quote",
                "mcp__trading__get_economic_indicators",
            ],
            include_partial_messages=True,  # for streaming
        )
        self._client = ClaudeSDKClient(options=options)

    async def process_query(self, user_input: str) -> AsyncIterator[Any]:
        """Send a query and yield response messages for the REPL to display."""
        await self._client.query(user_input)
        async for msg in self._client.receive_response():
            yield msg

    async def close(self) -> None:
        """Cleanup: close aggregator, cache."""
        if "aggregator" in self._resources:
            await self._resources["aggregator"].close()
        if "cache" in self._resources:
            await self._resources["cache"].close()

    async def __aenter__(self):
        await self._client.__aenter__()
        return self

    async def __aexit__(self, *args):
        await self._client.__aexit__(*args)
        await self.close()
```

## Step 3: MODIFY `src/main.py` — rewrite `chat` command

Replace the `chat()` function body (lines 309-353). Keep the command decorator and epilog.

```python
def chat(model: str = ...) -> None:
    import os
    logger = get_logger("cli")

    # Check for API key
    if not os.environ.get("ANTHROPIC_API_KEY"):
        console.print(
            "[red]Error:[/red] ANTHROPIC_API_KEY not set.\n"
            "Set it in your .env file or environment to use the chat feature."
        )
        raise typer.Exit(1)

    async def run_chat() -> None:
        from src.chat.engine import TradingChatEngine

        engine = TradingChatEngine(model=model)
        await engine.initialize()

        console.print(Panel(
            "[bold]Stock Trader Chat[/bold]\n\n"
            "Ask about any stock, options strategy, or market condition.\n"
            "Type [bold]help[/bold] for tips, [bold]exit[/bold] to quit.",
            title="Trading Assistant",
            border_style="cyan",
        ))

        try:
            async with engine:
                while True:
                    try:
                        user_input = Prompt.ask("\n[bold cyan]You[/bold cyan]")
                        user_input = user_input.strip()
                        if not user_input:
                            continue
                        if user_input.lower() in ("exit", "quit", "q"):
                            console.print("\n[dim]Goodbye! Happy trading.[/dim]")
                            break
                        if user_input.lower() == "help":
                            _show_chat_help()
                            continue
                        if user_input.lower() == "clear":
                            console.clear()
                            continue

                        console.print()
                        async for msg in engine.process_query(user_input):
                            if isinstance(msg, StreamEvent):
                                event = msg.event
                                if event.get("type") == "content_block_delta":
                                    delta = event.get("delta", {})
                                    if delta.get("type") == "text_delta":
                                        console.print(
                                            delta.get("text", ""),
                                            end="",
                                        )
                            elif isinstance(msg, AssistantMessage):
                                # Tool use blocks show status
                                for block in msg.content:
                                    if hasattr(block, "name"):
                                        console.print(
                                            f"\n[dim]Using {block.name}...[/dim]"
                                        )
                            elif isinstance(msg, ResultMessage):
                                console.print()  # newline after streamed text
                                if hasattr(msg, "total_cost_usd"):
                                    console.print(
                                        f"[dim]Cost: ${msg.total_cost_usd:.4f}[/dim]"
                                    )

                        # Post-response disclaimer (not generated by Claude)
                        console.print(
                            "\n[dim italic]This is AI-generated analysis for "
                            "informational purposes only. Not financial advice.[/dim italic]"
                        )

                    except KeyboardInterrupt:
                        console.print("\n[dim]Press Ctrl+C again or type 'exit' to quit.[/dim]")
                    except Exception as e:
                        console.print(f"\n[red]Error:[/red] {e}")
                        logger.exception("Error in chat")

        except Exception as e:
            console.print(f"\n[red]Failed to start chat engine:[/red] {e}")
            logger.exception("Chat engine failed")

    asyncio.run(run_chat())
```

Add a `_show_chat_help()` function displaying the same help content as the current `InteractiveSession.HELP_MESSAGE`, updated to reflect that this is now an AI-powered conversation.

## Step 4: MODIFY `flake.nix` — add dependency

Add to `pipPackages`:
```
          # Claude Agent SDK (for AI-powered chat)
          claude-agent-sdk>=0.1.30
```

## Step 5: No changes to `src/agents/orchestrator.py`

`TradingOrchestrator` and `InteractiveSession` remain untouched. They're still used by the MCP server path and referenced from `src/agents/__init__.py`. The `chat` command simply stops importing them.

## Step 6: NEW `tests/test_chat_engine.py`

Tests using `mock_aggregator` fixture from `conftest.py`:

1. **`test_system_prompt_contains_safety_rules`** — verify SYSTEM_PROMPT contains "ADVISORY", "never trade execution", "Never fabricate"
2. **`test_engine_initialize_creates_resources`** — mock DataCache/aggregator, verify all agents created
3. **`test_engine_close_cleans_up`** — verify aggregator.close() and cache.close() called
4. **`test_tool_get_quote`** — call the tool function directly with mock aggregator, verify return format
5. **`test_tool_get_technical_analysis`** — mock TechnicalAnalystAgent.full_analysis(), verify tool returns formatted text
6. **`test_tool_assess_risk`** — call with entry/stop/target, verify RiskAssessorAgent called
7. **`test_missing_api_key_exits`** — verify chat command exits with error when ANTHROPIC_API_KEY is unset

## Safety & Security

- **Advisory invariant preserved**: All 7 tools are read-only wrappers around existing agents. No write/execute capabilities exposed to Claude.
- **Risk cap preserved**: `RiskAssessorAgent` enforces 10% max via `Settings.validate_risk()`.
- **API key handling**: `ANTHROPIC_API_KEY` checked at startup, never logged or displayed. Exits cleanly with instructions if missing.
- **Tool restriction**: `allowed_tools` explicitly lists only the 7 trading tools. Claude cannot access file system, bash, or any other tools.
- **`permission_mode="bypassPermissions"`**: Since the only available tools are our read-only MCP tools, this prevents the SDK from prompting for tool approval interactively.
- **Post-processed disclaimer**: After every Claude response, a hardcoded disclaimer is printed. This is NOT generated by Claude and cannot be suppressed by prompt injection.
- **No PII risk**: Only market tickers and prices flow through Claude. No personal financial data.
- **Cost transparency**: `ResultMessage.total_cost_usd` displayed after each response.

## Dependency

- `claude-agent-sdk>=0.1.30` added to `pipPackages` in `flake.nix`
- Requires `ANTHROPIC_API_KEY` environment variable
- The SDK bundles the Claude CLI subprocess internally

## Verification

1. `pytest tests/test_chat_engine.py` — unit tests for tools, engine lifecycle, safety
2. `ruff check src/ tests/ && mypy src/` — lint and type check
3. Manual integration test:
   ```
   python -m src.main chat
   > What's the technical analysis for NVDA?
   > How about the options setup?     (tests context retention)
   > exit
   ```
4. `python -m src.main analyze NVDA` and `python -m src.main scan --preset oversold` — verify unaffected
