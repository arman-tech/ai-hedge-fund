# Database Repository Layer - Execution Plan

## 🎯 Core Strategy: Zero-Impact Abstraction Layer

**Key Insight**: Agents should never know if data comes from cache, database, or API. This eliminates migration complexity and risk.

## 🏗️ Architecture Overview

```
Current: Agent → API Functions → Financial Datasets API
Target:  Agent → API Functions → Data Service → [Cache|DB|API]
```

**Critical Design Decision**: Keep existing `src/tools/api.py` function signatures identical. Agents require zero changes.

## 📋 Implementation Phases

### Phase 1: Service Abstraction (Days 1-7)

**Goal**: Create the abstraction without changing any agent code

**Week 1 Tasks:**

1. **Day 1-2**: Create `IFinancialDataService` interface
2. **Day 3-4**: Implement `DirectApiDataService` (wraps current behavior)
3. **Day 5-6**: Update `src/tools/api.py` to use service pattern
4. **Day 7**: Test all agents work identically

**Validation**: Run full agent test suite - should be 100% identical results

### Phase 2: Database Infrastructure (Days 8-14)

**Goal**: Add PostgreSQL with repository pattern

**Week 2 Tasks:**

1. **Day 8-9**: Docker PostgreSQL + SQLAlchemy setup
2. **Day 10-11**: Create database models matching Pydantic models
3. **Day 12-13**: Implement repository classes
4. **Day 14**: Database migration scripts

**Validation**: Database can store/retrieve all data types correctly

### Phase 3: Hybrid Implementation (Days 15-21)

**Goal**: Implement 3-layer caching with smart fallback

**Week 3 Tasks:**

1. **Day 15-16**: Create `HybridDataService` with L1→L2→L3 flow logic

   - Enhance existing L1 cache with timestamp tracking
   - Implement freshness checking for each layer
   - Create cache population flow (L3→L2→L1)

2. **Day 17-18**: Implement data freshness management

   - TTL configuration per data type
   - Cache invalidation strategies
   - Background refresh for near-expiry data

3. **Day 19-20**: Add comprehensive error handling and API fallback

   - Database failure → skip L2, go L1→L3
   - API failure → serve stale data with warnings
   - Cache corruption → rebuild from available layers

4. **Day 21**: Performance testing and optimization
   - Measure cache hit ratios at each layer
   - Optimize database queries and connection pooling
   - Validate 80%+ API call reduction

**Validation**: 80%+ API call reduction while maintaining identical results

### Phase 4: Production Deployment (Days 22-28)

**Goal**: Deploy with monitoring and backup

**Week 4 Tasks:**

1. **Day 22-23**: Configuration management and environment setup
2. **Day 24-25**: Azure backup integration
3. **Day 26-27**: Monitoring, logging, and alerting
4. **Day 28**: Documentation and operational procedures

**Validation**: Production-ready system with full observability

## 🔧 Technical Implementation

### **Three-Layer Cache Flow (Detailed Implementation)**

**Critical**: We must preserve and enhance the existing L1 cache, not replace it.

```python
# Detailed cache flow logic
async def get_financial_metrics(ticker: str, end_date: str, period: str = "ttm", limit: int = 10):
    cache_key = f"{ticker}_{period}_{end_date}_{limit}"

    # L1 CHECK: In-memory cache (existing cache.py)
    if l1_data := _memory_cache.get_financial_metrics(cache_key):
        if is_data_fresh(l1_data, FINANCIAL_METRICS_TTL):  # 7 days
            return l1_data

    # L2 CHECK: PostgreSQL database
    if l2_data := await _database_repo.get_financial_metrics(ticker, end_date, period, limit):
        if is_data_fresh(l2_data, FINANCIAL_METRICS_TTL):
            # Populate L1 cache for next request
            _memory_cache.set_financial_metrics(cache_key, l2_data)
            return l2_data

    # L3 FETCH: Financial Datasets API (expensive)
    l3_data = await _api_client.get_financial_metrics(ticker, end_date, period, limit)

    # POPULATE CACHE HIERARCHY (data flows up)
    await _database_repo.save_financial_metrics(l3_data)  # L2
    _memory_cache.set_financial_metrics(cache_key, l3_data)  # L1

    return l3_data
```

### **Cache Freshness Strategy by Layer**

| Data Type         | L1 TTL  | L2 TTL | L3 Source    | Logic    |
| ----------------- | ------- | ------ | ------------ | -------- |
| Financial Metrics | Session | 7 days | Always fresh | L1→L2→L3 |
| Price Data        | Session | 15 min | Always fresh | L1→L2→L3 |
| Company News      | Session | 1 hour | Always fresh | L1→L2→L3 |
| Insider Trades    | Session | 1 day  | Always fresh | L1→L2→L3 |
| Line Items        | Session | 7 days | Always fresh | L1→L2→L3 |

### **Enhanced L1 Cache Integration**

```python
# src/data/enhanced_cache.py - Extends existing cache.py
class EnhancedCache(Cache):  # Inherits from existing Cache
    def __init__(self):
        super().__init__()
        self._cache_timestamps: dict[str, datetime] = {}

    def is_fresh(self, cache_key: str, ttl_seconds: int) -> bool:
        """Check if cached data is still fresh"""
        if cache_key not in self._cache_timestamps:
            return False

        age = datetime.now() - self._cache_timestamps[cache_key]
        return age.total_seconds() < ttl_seconds

    def set_with_timestamp(self, cache_key: str, data: any):
        """Set data with timestamp for freshness checking"""
        # Use existing set methods from parent class
        # Add timestamp tracking
        self._cache_timestamps[cache_key] = datetime.now()
```

### Service Interface Design

```python
# src/services/interfaces.py
class IFinancialDataService(ABC):
    def get_financial_metrics(self, ticker: str, end_date: str, period: str = "ttm", limit: int = 10):
        pass
    def get_prices(self, ticker: str, start_date: str, end_date: str):
        pass
    # ... other methods matching current API
```

### Updated API Layer

```python
# src/tools/api.py - ZERO CHANGES to function signatures
from src.services.factory import get_data_service

def get_financial_metrics(ticker: str, end_date: str, period: str = "ttm", limit: int = 10):
    service = get_data_service()  # Returns configured implementation
    return service.get_financial_metrics(ticker, end_date, period, limit)
```

### Configuration-Driven Switching

```python
# Environment variable controls implementation
USE_DATABASE_CACHE=true  → HybridDataService (cache + DB + API)
USE_DATABASE_CACHE=false → DirectApiDataService (current behavior)
```

## 📊 Data Freshness Strategy

| Data Type         | Cache Duration | Rationale                                     |
| ----------------- | -------------- | --------------------------------------------- |
| Financial Metrics | 7 days         | Quarterly reports change predictably          |
| Price Data        | 15 minutes     | Balance real-time needs with API limits       |
| Company News      | 1 hour         | News sentiment doesn't need immediate updates |
| Insider Trades    | 1 day          | SEC filing delays                             |
| Line Items        | 7 days         | Financial statements are static               |

## 🗄️ Database Schema (Simplified)

```sql
-- Core tables mirror Pydantic models exactly
CREATE TABLE financial_metrics (
    ticker VARCHAR(10),
    report_period DATE,
    period VARCHAR(20),
    -- All FinancialMetrics fields as DECIMAL/VARCHAR
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(ticker, report_period, period)
);

-- Similar pattern for: price_data, line_items, insider_trades, company_news
```

## 🚀 Docker Configuration

```yaml
# docker/docker-compose.db.yml
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: hedge_fund_data
      POSTGRES_USER: hedge_fund_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 30s
      timeout: 10s
      retries: 3
```

## 📈 Success Metrics

**Primary Goals:**

- 80%+ reduction in API calls to financialdatasets.ai
- Zero changes required to any of the 21 agents
- <100ms response time for cached data
- Graceful degradation if database unavailable

**Monitoring:**

- API call count per analysis session
- Cache hit ratios by data type
- Database query performance
- System availability metrics

## 🎯 Risk Mitigation

**Technical Risks:**

- Database failure → Automatic fallback to API-only mode
- Performance issues → Connection pooling and query optimization
- Data inconsistency → Transaction management and validation

**Implementation Risks:**

- Agent compatibility → Maintain identical function signatures
- Migration complexity → Gradual rollout with configuration switches
- Testing coverage → Comprehensive integration tests

## 🔒 Azure Backup Strategy

```bash
# Daily automated backup to Azure
pg_dump hedge_fund_data | az storage blob upload \
  --account-name hedgefundstorage \
  --container-name database-backups \
  --name daily_backup_$(date +%Y%m%d).sql
```

## 📋 Deliverables

**Code:**

- Service abstraction layer
- PostgreSQL repository implementation
- Hybrid caching service
- Docker configuration
- Migration scripts

**Documentation:**

- Architecture decision records
- Configuration guide
- Operational procedures
- Performance benchmarks

**Infrastructure:**

- PostgreSQL database with optimized schema
- Azure backup integration
- Monitoring and alerting setup

## 🎯 Next Steps

1. **Approval**: Confirm approach and timeline
2. **Environment Setup**: Prepare development environment
3. **Phase 1 Start**: Begin service abstraction implementation
4. **Weekly Reviews**: Progress check and course correction

**Estimated Timeline: 4 weeks**
**Risk Level: Low** (backward compatible, proven patterns)
**Impact: High** (80% API cost reduction, zero agent changes)

Ready to proceed when approved! 🚀
