# Advanced Multi-URL Crawling with Dispatchers

> **Heads Up**: Crawl4AI supports advanced dispatchers for **parallel** or **throttled** crawling, providing dynamic rate limiting and memory usage checks. The built-in `arun_many()` function uses these dispatchers to handle concurrency efficiently.

## 1. Introduction

When crawling many URLs:
- **Basic**: Use `arun()` in a loop (simple but less efficient)
- **Better**: Use `arun_many()`, which efficiently handles multiple URLs with proper concurrency control
- **Best**: Customize dispatcher behavior for your specific needs (memory management, rate limits, etc.)

**Why Dispatchers?**  
- **Adaptive**: Memory-based dispatchers can pause or slow down based on system resources
- **Rate-limiting**: Built-in rate limiting with exponential backoff for 429/503 responses
- **Real-time Monitoring**: Live dashboard of ongoing tasks, memory usage, and performance
- **Flexibility**: Choose between memory-adaptive or semaphore-based concurrency

## 2. Core Components

### 2.1 Rate Limiter

```python
class RateLimiter:
    def __init__(
        base_delay: Tuple[float, float] = (1.0, 3.0),  # Random delay range between requests
        max_delay: float = 60.0,                        # Maximum backoff delay
        max_retries: int = 3,                          # Retries before giving up
        rate_limit_codes: List[int] = [429, 503]       # Status codes triggering backoff
    )
```

The RateLimiter provides:
- Random delays between requests
- Exponential backoff on rate limit responses
- Domain-specific rate limiting
- Automatic retry handling

### 2.2 Crawler Monitor

The CrawlerMonitor provides real-time visibility into crawling operations:

```python
monitor = CrawlerMonitor(
    max_visible_rows=15,           # Maximum rows in live display
    display_mode=DisplayMode.DETAILED  # DETAILED or AGGREGATED view
)
```

**Display Modes**:
1. **DETAILED**: Shows individual task status, memory usage, and timing
2. **AGGREGATED**: Displays summary statistics and overall progress

## 3. Available Dispatchers

### 3.1 MemoryAdaptiveDispatcher (Default)

Automatically manages concurrency based on system memory usage:

```python
dispatcher = MemoryAdaptiveDispatcher(
    memory_threshold_percent=90.0,  # Pause if memory exceeds this
    check_interval=1.0,             # How often to check memory
    max_session_permit=10,          # Maximum concurrent tasks
    rate_limiter=RateLimiter(       # Optional rate limiting
        base_delay=(1.0, 2.0),
        max_delay=30.0,
        max_retries=2
    ),
    monitor=CrawlerMonitor(         # Optional monitoring
        max_visible_rows=15,
        display_mode=DisplayMode.DETAILED
    )
)
```

### 3.2 SemaphoreDispatcher

Provides simple concurrency control with a fixed limit:

```python
dispatcher = SemaphoreDispatcher(
    max_session_permit=5,             # Fixed concurrent tasks
    rate_limiter=RateLimiter(      # Optional rate limiting
        base_delay=(0.5, 1.0),
        max_delay=10.0
    ),
    monitor=CrawlerMonitor(        # Optional monitoring
        max_visible_rows=15,
        display_mode=DisplayMode.DETAILED
    )
)
```

## 4. Usage Examples

### 4.1 Batch Processing (Default)

```python
async def crawl_batch():
    browser_config = BrowserConfig(headless=True, verbose=False)
    run_config = CrawlerRunConfig(
        cache_mode=CacheMode.BYPASS,
        stream=False  # Default: get all results at once
    )
    
    dispatcher = MemoryAdaptiveDispatcher(
        memory_threshold_percent=70.0,
        check_interval=1.0,
        max_session_permit=10,
        monitor=CrawlerMonitor(
            display_mode=DisplayMode.DETAILED
        )
    )

    async with AsyncWebCrawler(config=browser_config) as crawler:
        # Get all results at once
        results = await crawler.arun_many(
            urls=urls,
            config=run_config,
            dispatcher=dispatcher
        )
        
        # Process all results after completion
        for result in results:
            if result.success:
                await process_result(result)
            else:
                print(f"Failed to crawl {result.url}: {result.error_message}")
```

### 4.2 Streaming Mode

```python
async def crawl_streaming():
    browser_config = BrowserConfig(headless=True, verbose=False)
    run_config = CrawlerRunConfig(
        cache_mode=CacheMode.BYPASS,
        stream=True  # Enable streaming mode
    )
    
    dispatcher = MemoryAdaptiveDispatcher(
        memory_threshold_percent=70.0,
        check_interval=1.0,
        max_session_permit=10,
        monitor=CrawlerMonitor(
            display_mode=DisplayMode.DETAILED
        )
    )

    async with AsyncWebCrawler(config=browser_config) as crawler:
        # Process results as they become available
        async for result in await crawler.arun_many(
            urls=urls,
            config=run_config,
            dispatcher=dispatcher
        ):
            if result.success:
                # Process each result immediately
                await process_result(result)
            else:
                print(f"Failed to crawl {result.url}: {result.error_message}")
```

### 4.3 Semaphore-based Crawling

```python
async def crawl_with_semaphore(urls):
    browser_config = BrowserConfig(headless=True, verbose=False)
    run_config = CrawlerRunConfig(cache_mode=CacheMode.BYPASS)
    
    dispatcher = SemaphoreDispatcher(
        semaphore_count=5,
        rate_limiter=RateLimiter(
            base_delay=(0.5, 1.0),
            max_delay=10.0
        ),
        monitor=CrawlerMonitor(
            max_visible_rows=15,
            display_mode=DisplayMode.DETAILED
        )
    )
    
    async with AsyncWebCrawler(config=browser_config) as crawler:
        results = await crawler.arun_many(
            urls, 
            config=run_config,
            dispatcher=dispatcher
        )
        return results
```

### 4.4 Robots.txt Consideration

```python
import asyncio
from crawl4ai import AsyncWebCrawler, CrawlerRunConfig, CacheMode

async def main():
    urls = [
        "https://example1.com",
        "https://example2.com",
        "https://example3.com"
    ]
    
    config = CrawlerRunConfig(
        cache_mode=CacheMode.ENABLED,
        check_robots_txt=True,  # Will respect robots.txt for each URL
        semaphore_count=3      # Max concurrent requests
    )
    
    async with AsyncWebCrawler() as crawler:
        async for result in crawler.arun_many(urls, config=config):
            if result.success:
                print(f"Successfully crawled {result.url}")
            elif result.status_code == 403 and "robots.txt" in result.error_message:
                print(f"Skipped {result.url} - blocked by robots.txt")
            else:
                print(f"Failed to crawl {result.url}: {result.error_message}")

if __name__ == "__main__":
    asyncio.run(main())
```

**Key Points**:
- When `check_robots_txt=True`, each URL's robots.txt is checked before crawling
- Robots.txt files are cached for efficiency
- Failed robots.txt checks return 403 status code
- Dispatcher handles robots.txt checks automatically for each URL

## 5. Dispatch Results

Each crawl result includes dispatch information:

```python
@dataclass
class DispatchResult:
    task_id: str
    memory_usage: float
    peak_memory: float
    start_time: datetime
    end_time: datetime
    error_message: str = ""
```

Access via `result.dispatch_result`:

```python
for result in results:
    if result.success:
        dr = result.dispatch_result
        print(f"URL: {result.url}")
        print(f"Memory: {dr.memory_usage:.1f}MB")
        print(f"Duration: {dr.end_time - dr.start_time}")
```

## 6. Summary

1. **Two Dispatcher Types**:
   - MemoryAdaptiveDispatcher (default): Dynamic concurrency based on memory
   - SemaphoreDispatcher: Fixed concurrency limit

2. **Optional Components**:
   - RateLimiter: Smart request pacing and backoff
   - CrawlerMonitor: Real-time progress visualization

3. **Key Benefits**:
   - Automatic memory management
   - Built-in rate limiting
   - Live progress monitoring
   - Flexible concurrency control

Choose the dispatcher that best fits your needs:
- **MemoryAdaptiveDispatcher**: For large crawls or limited resources
- **SemaphoreDispatcher**: For simple, fixed-concurrency scenarios
