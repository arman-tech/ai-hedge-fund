# User Rules 2.4 - Core Development Standards

_Stack: .NET 8+ | ASP.NET MVC | React 18+ | Docker | Azure | Git_

## 🧠 CRITICAL: Memory Bank First
**Always read ALL memory-bank/ files before ANY task:**
```
memory-bank/
├── projectbrief.md      # Project scope
├── productContext.md    # Business purpose
├── activeContext.md     # Current focus
├── systemPatterns.md    # Architecture
├── techContext.md       # Tech stack
└── progress.md          # Status/issues
```
Update when: patterns discovered, major changes, user says "update memory bank"

## 🚨 Pre-Action Checklist (MANDATORY)
- [ ] Search existing files before creating
- [ ] Use absolute imports (@paths) NEVER relative
- [ ] Use constants for URLs/timing NEVER hardcode
- [ ] Run linting after changes
- [ ] Use `git --no-pager` always
- [ ] ASK when unclear, AVOID assumptions

## 📂 Import Standards
```typescript
// ✅ CORRECT
import { API_ENDPOINTS } from '@shared/constants';
import { UserCard } from '@components/UserCard';

// ❌ WRONG: ./relative ../paths hardcoded-urls
```

## 🏗️ Architecture
**Clean Architecture**: Domain → Application → Infrastructure → Presentation  
**SOLID**: Single-responsibility | Open-closed | Liskov | Interface-segregation | Dependency-inversion  
**Principles**: Composition > inheritance | Methods <30 lines | Early returns | Self-documenting

## 🔧 .NET 8+ Standards
```csharp
// Async pattern
public async Task<Result<T>> GetAsync(int id, CancellationToken ct = default)

// Structured logging
_logger.LogInformation("Processing {UserId} at {Timestamp}", userId, DateTime.UtcNow);

// Records for DTOs
public record UserDto(int Id, string Name);
```

**Data**: EF Core (CRUD) | Dapper (performance) | DBUp (migrations)  
**Testing**: xUnit | NSubstitute | Shouldly | TestContainers | Custom mapping (no AutoMapper)  
**Error**: Global middleware | Correlation IDs | Retry policies

## ⚛️ React 18+ Standards
```typescript
// Functional components only
export const UserCard: FC<Props> = ({ user, onSelect }) => {
  const [loading, setLoading] = useState(false);
  return <button onClick={() => onSelect(user.id)}>{user.name}</button>;
};
```

**State**: useState/useEffect (local) | Context (shared) | Redux Toolkit (complex)  
**Performance**: React.memo | useMemo/useCallback | React.lazy  
**Style**: Tailwind CSS | Mobile-first | ARIA labels | Semantic HTML

## 🧪 Debug Workflow (TDD)
1. Run test: `dotnet test --filter "Name"`
2. Read logs in `TestLogs/` sequentially
3. Analyze: timestamps, patterns, divergence
4. Fix targeted, verify, document

## 🔒 Security
**Auth**: Azure AD B2C | JWT | RBAC | CSRF protection  
**Data**: Encryption (rest/transit) | HTTPS only | Input validation | OWASP

## 🎯 Implementation
1. **Analyze**: Entry points, dependencies, patterns
2. **Design Patterns**: Repository | Factory | Strategy | Observer | Decorator
3. **Ask**: Real problem? Improves maintainability? Simpler solution?
4. **Order**: Minimize conflicts → Enable validation → Log first → TDD

## 📚 Extensions (Load When Needed)
```
"Load the [extension] for [specific task]"
```
- **performance**: DB optimization, React performance
- **security-core**: STRIDE, auth patterns
- **security-compliance**: GDPR, SOC2, HIPAA
- **security-testing**: Penetration testing
- **testing**: E2E, performance testing
- **devops**: Docker, CI/CD
- **monitoring**: Application Insights
- **accessibility**: WCAG compliance
- **quality**: Refactoring, tech debt
- **lifecycle**: Sprint planning

## 🧠 Investigation Process
**Before solutions**: Evidence → Validate assumptions → Check scope → Verify solution  
**Git commands**: Always use `--no-pager` (status, log, diff, branch)

## 📖 Documentation
```csharp
/// <summary>Processes registration</summary>
/// <param name="model">Registration data</param>
/// <returns>Result with user ID</returns>
```
Project docs: README | API (Swagger) | CHANGELOG | Architecture decisions

---
_v2.4 - Optimized for clarity and context efficiency_