# Architecture Decision Records (ADR)

Dokumentasi keputusan arsitektur dan alasan di balik setiap keputusan teknis dalam Bee Group Platform.

---

## ADR-001: Monolithic Backend Architecture

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih antara monolithic vs microservices architecture

### Decision
Menggunakan monolithic architecture dengan Node.js + Express untuk backend.

### Rationale
1. **Simplicity**: Lebih mudah untuk dikembangkan dan di-deploy pada awal project
2. **Development Speed**: Tim dapat bergerak lebih cepat tanpa overhead komunikasi antar service
3. **Easier Debugging**: Stack trace dan debugging lebih straightforward
4. **Lower Operational Complexity**: Tidak perlu service discovery, distributed transactions, dll
5. **Cost Efficient**: Infrastructure cost lebih rendah dibanding microservices

### Consequences
- (+) Faster initial development
- (+) Simpler deployment pipeline
- (+) Easier cross-cutting concerns (auth, logging)
- (-) Potential scalability limitations di masa depan
- (-) Cannot scale individual components independently
- (-) Risk of technology lock-in

### Future Migration Path
Jika diperlukan, dapat di-refactor menjadi microservices dengan:
- Backend for Frontend (BFF) pattern
- API gateway
- Event-driven architecture dengan message queue

---

## ADR-002: PostgreSQL as Primary Database

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih database relational vs NoSQL

### Decision
Menggunakan PostgreSQL sebagai primary database.

### Rationale
1. **Strong ACID Compliance**: Konsistensi data dijamin
2. **Complex Queries**: Support JOIN kompleks untuk portfolio analysis
3. **Transaction Support**: ACID transactions untuk financial operations
4. **Array & JSON Support**: Flexible data types untuk berbagai use case
5. **Mature Ecosystem**: Tool dan library yang sudah mature
6. **Cost**: Open source dan tidak ada licensing cost

### Data Model Benefits
- Relational model sesuai dengan hierarchical company structure
- Foreign keys untuk referential integrity
- Constraints untuk business rules
- Built-in audit trail dengan triggers

### Consequences
- (+) Strong consistency guarantee
- (+) Excellent for reporting and analytics
- (+) Good relationship handling
- (-) Vertical scaling limitations
- (-) Complex distributed transactions
- (-) Not ideal for unstructured data

### Backup & Recovery
```sql
-- Point-in-time recovery dengan WAL archiving
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/wal/%f'
```

---

## ADR-003: Prisma ORM Selection

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih ORM untuk Node.js

### Decision
Menggunakan Prisma sebagai ORM.

### Rationale
1. **Type Safety**: TypeScript support built-in (future-proofing)
2. **Migration Management**: Prisma Migrate untuk version control database
3. **Developer Experience**: Modern API, auto-completion
4. **Query Optimization**: Automatic N+1 prevention dengan include/select
5. **Introspection**: Dapat generate schema dari existing database
6. **Community**: Growing community dan ecosystem

### Schema-Driven Development
```prisma
// Schema mendefinisikan source of truth untuk database
model Company {
  id String @id @default(cuid())
  name String
  code String @unique
}
```

### Consequences
- (+) Excellent developer experience
- (+) Built-in migrations
- (+) Type safety with TypeScript
- (+) Good performance characteristics
- (-) Less flexibility untuk complex queries
- (-) ORM overhead untuk performance-critical operations

### Alternative Approaches
- Raw SQL untuk performance-critical queries
- Query builder untuk complex WHERE clauses
- Database views untuk complex aggregations

---

## ADR-004: JWT for Authentication

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih authentication mechanism

### Decision
Menggunakan JWT (JSON Web Tokens) dengan refresh token pattern.

### Rationale
1. **Stateless**: Tidak perlu session storage di server
2. **Scalable**: Cocok untuk distributed systems
3. **Mobile-Friendly**: Well-supported di mobile apps
4. **Standard**: Widely adopted dan well-understood
5. **Granular Permissions**: Dapat embed claims dalam token

### Token Strategy
```javascript
// Access Token (short-lived): 15 minutes
// Refresh Token (long-lived): 30 days
// CSRF Token untuk protected mutations

// Token Payload
{
  sub: "user-id",
  email: "user@example.com",
  role: "MANAGER",
  permissions: ["read:companies", "write:companies"],
  iat: 1624617600,
  exp: 1624618500
}
```

### Security Considerations
- Tokens stored in HTTP-only cookies
- HTTPS enforcement
- Token rotation on refresh
- Token blacklisting untuk logout
- Rate limiting pada token refresh

### Consequences
- (+) Stateless authentication
- (+) Easy to scale
- (+) Good for APIs
- (-) Token revocation complex
- (-) Token size constraints
- (-) Cannot change permissions until token expires

### Token Revocation Strategy
```javascript
// Redis blacklist untuk revoked tokens
// Short TTL matching token expiration
redis.setex(`token:${tokenJTI}`, 900, 'revoked')
```

---

## ADR-005: Redis for Caching & Sessions

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih in-memory cache layer

### Decision
Menggunakan Redis untuk caching dan session management.

### Rationale
1. **Performance**: Sub-millisecond response time
2. **Data Structures**: Support multiple data types (strings, hashes, sets, etc)
3. **Persistence**: RDB dan AOF untuk durability
4. **Pub/Sub**: Real-time message queue capabilities
5. **Distributed Lock**: Untuk preventing race conditions

### Use Cases dalam Project
```javascript
// 1. Query Result Caching
cache.set('companies:list', companiesData, 300) // 5 minutes

// 2. Session Storage
session.set(sessionId, userData, 1800) // 30 minutes

// 3. Rate Limiting
limiter.increment(`requests:${userId}`, 3600) // 1 hour window

// 4. Task Queue
queue.push('reports:generate', taskData)

// 5. Pub/Sub untuk real-time updates
redis.publish('companies:updated', companyData)
```

### Cache Invalidation
```javascript
// Event-driven cache invalidation
after UpdateCompany():
  - Invalidate company detail cache
  - Invalidate companies list cache
  - Invalidate portfolio cache
  - Publish event untuk subscriber updates
```

### Consequences
- (+) Dramatically improved performance
- (+) Flexible data structures
- (+) Pub/Sub capabilities
- (-) Additional infrastructure to maintain
- (-) Memory limitations
- (-) Data consistency challenges

### Fallback Strategy
```javascript
// Graceful degradation jika Redis down
try {
  data = await redis.get(key)
} catch (error) {
  logger.warn('Redis unavailable, fetching from database')
  data = await database.query(...)
}
```

---

## ADR-006: React + Vite for Frontend

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih frontend framework dan build tool

### Decision
Menggunakan React 18 dengan Vite sebagai build tool.

### Rationale
1. **React**: Component-based, large ecosystem, developer experience
2. **Vite**: Fast build, instant HMR, optimized production builds
3. **Performance**: Sub-second dev startup, millisecond HMR
4. **ESM Native**: Modern module system
5. **Rollup Integration**: Excellent code splitting

### Project Structure
```
frontend/
├── src/
│   ├── components/    # Reusable components
│   ├── pages/         # Page components
│   ├── store/         # Redux store
│   ├── services/      # API services
│   ├── hooks/         # Custom hooks
│   └── utils/         # Utility functions
```

### Code Splitting Strategy
```javascript
// Lazy load routes
const routes = [
  { path: '/', component: lazy(() => import('./pages/Dashboard')) },
  { path: '/companies', component: lazy(() => import('./pages/Companies')) }
]

// Chunk split configuration
manualChunks: {
  'vendor': ['react', 'react-dom'],
  'ui-lib': ['tailwindcss', 'lucide-react'],
  'charts': ['recharts']
}
```

### State Management
- Redux Toolkit untuk global state
- Local component state untuk UI state
- Custom hooks untuk logic reuse

### Consequences
- (+) Modern DX with Vite
- (+) Excellent performance
- (+) React ecosystem resources
- (+) Component reusability
- (-) JavaScript framework overhead
- (-) Bundle size concerns for low-bandwidth users

---

## ADR-007: Docker Containerization

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih containerization strategy

### Decision
Menggunakan Docker untuk containerization dan Docker Compose untuk development.

### Rationale
1. **Consistency**: "Works on my machine" tidak jadi masalah
2. **Isolation**: Services berjalan dalam isolated environments
3. **Easy Scaling**: Dengan orchestrators seperti Kubernetes
4. **Development Parity**: Dev environment mirip production
5. **CI/CD Integration**: Native support di semua CI/CD platforms

### Multi-Stage Build
```dockerfile
# Development stage
FROM node:20-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm ci

# Builder stage
FROM development AS builder
COPY . .
RUN npm run build

# Production stage
FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm ci --production
```

### Image Optimization
- Alpine Linux untuk smaller images
- Multi-stage builds untuk reduced final size
- Non-root user untuk security
- Health checks untuk container orchestration

### Consequences
- (+) Easy deployment
- (+) Environment consistency
- (+) Excellent for microservices
- (-) Container overhead
- (-) Image size management needed
- (-) Security concerns if not done right

---

## ADR-008: Kubernetes for Production Orchestration

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih container orchestration platform untuk production

### Decision
Menggunakan Kubernetes untuk production workload orchestration.

### Rationale
1. **Industry Standard**: De facto standard untuk container orchestration
2. **Scalability**: Auto-scaling berdasarkan metrics
3. **High Availability**: Built-in redundancy dan health checking
4. **Rolling Updates**: Zero-downtime deployments
5. **Resource Management**: Efficient utilization dengan resource limits
6. **Ecosystem**: Rich ecosystem of tools and extensions

### Deployment Strategy
```yaml
# Blue-Green Deployment
- Deploy green service alongside blue
- Test green thoroughly
- Switch traffic to green
- Keep blue for quick rollback
```

### Resource Management
```yaml
# CPU dan memory limits
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"
```

### Consequences
- (+) Production-grade orchestration
- (+) Auto-scaling capabilities
- (+) Self-healing
- (+) Advanced deployment strategies
- (-) Operational complexity
- (-) Steep learning curve
- (-) Cost for small deployments

---

## ADR-009: GitHub Actions for CI/CD

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih CI/CD platform

### Decision
Menggunakan GitHub Actions untuk automation pipeline.

### Rationale
1. **Native Integration**: Built-in dengan GitHub
2. **No Additional Accounts**: Tidak perlu platform pihak ketiga
3. **Generous Free Tier**: Cocok untuk open source
4. **Flexible**: Highly customizable dengan actions marketplace
5. **Good Documentation**: Well-documented dan active community

### Pipeline Stages
```yaml
1. Trigger: Push to main atau pull request
2. Test: Run unit tests, lint, type checking
3. Build: Build Docker images
4. Push: Push ke container registry
5. Deploy: Deploy ke staging/production
6. Verify: Run smoke tests
7. Notify: Slack notification
```

### Consequences
- (+) No additional infrastructure
- (+) Good integration with GitHub
- (+) Community actions available
- (-) Limited to GitHub
- (-) Can be slower than dedicated CI/CD
- (-) Action marketplace quality varies

---

## ADR-010: Tailwind CSS for Styling

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih CSS framework/approach

### Decision
Menggunakan Tailwind CSS untuk styling dengan utility-first approach.

### Rationale
1. **Utility-First**: Tidak perlu menulis custom CSS
2. **Small Bundle**: PurgeCSS removes unused styles
3. **Responsive**: Built-in responsive utilities
4. **Dark Mode**: Built-in dark mode support
5. **Customization**: Extensive customization via config
6. **Performance**: No CSS-in-JS overhead

### Customization
```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      colors: {
        primary: '#3b82f6',
        secondary: '#10b981'
      }
    }
  }
}
```

### Consequences
- (+) Fast development
- (+) Consistent design system
- (+) Small production bundle
- (+) Easy responsive design
- (-) HTML verbosity
- (-) Learning curve
- (-) Class naming conventions needed

---

## ADR-011: API Versioning Strategy

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih versioning strategy untuk API

### Decision
Menggunakan URL path versioning (`/api/v1/`) untuk backward compatibility.

### Rationale
1. **Clarity**: Version terlihat jelas di URL
2. **Easy Testing**: Dapat test multiple versions
3. **Explicit**: No ambiguity tentang API version
4. **Migration**: Easy untuk migrate clients

### Versioning Policy
```javascript
// Current version: v1
GET /api/v1/companies

// Future versions
GET /api/v2/companies

// Support multiple versions untuk smooth migration
// v1: Maintained untuk 12 months
// v2: New version, new features
```

### Deprecation Path
```javascript
// Add deprecation header
res.setHeader('Deprecation', 'true')
res.setHeader('Sunset', 'Sun, 25 Dec 2025 23:59:59 GMT')
```

### Consequences
- (+) Easy versioning
- (+) Clear deprecation path
- (+) Backward compatibility
- (-) URL duplication
- (-) Increased maintenance
- (-) Version management overhead

---

## ADR-012: Logging & Monitoring Stack

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih logging dan monitoring solution

### Decision
Menggunakan:
- Winston untuk application logging
- ELK Stack (Elasticsearch, Logstash, Kibana) untuk centralized logging
- Prometheus + Grafana untuk metrics dan monitoring

### Rationale
1. **Observability**: Complete visibility into system
2. **Debugging**: Easy to trace issues
3. **Alerting**: Proactive problem detection
4. **Performance Analysis**: Identify bottlenecks
5. **Compliance**: Audit trail untuk regulatory requirements

### Logging Levels
```javascript
{
  error: 0,    // Critical errors
  warn: 1,     // Warnings
  info: 2,     // General information
  debug: 3,    // Debug information
  trace: 4     // Detailed trace
}
```

### Metrics Collection
```javascript
// Request metrics
- Response time
- Error rate
- Throughput

// Database metrics
- Query execution time
- Connection pool utilization
- Slow queries

// Business metrics
- Companies created/updated
- Portfolio value
- User activity
```

### Consequences
- (+) Complete observability
- (+) Easy debugging
- (+) Performance insights
- (+) Proactive alerting
- (-) Infrastructure overhead
- (-) Data retention costs
- (-) Complexity in setup

---

## ADR-013: Testing Strategy

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih testing approach dan tools

### Decision
Menggunakan:
- **Unit Tests**: Jest untuk backend dan frontend
- **Integration Tests**: Supertest untuk API endpoints
- **E2E Tests**: Playwright untuk critical user flows
- **Coverage Target**: 80% code coverage

### Testing Pyramid
```
          E2E Tests (few)
       /
      / Integration Tests (moderate)
     /
    / Unit Tests (many)
   /______________________
```

### Test Organization
```javascript
// Backend structure
backend/
├── src/
├── __tests__/
│   ├── unit/
│   ├── integration/
│   └── fixtures/

// Frontend structure
frontend/
├── src/
├── __tests__/
│   ├── unit/
│   ├── integration/
│   └── mocks/
```

### CI/CD Integration
```yaml
# Tests run on every push/PR
- npm run test:unit
- npm run test:integration
- npm run test:coverage
- npm run test:e2e (only for merge to main)
```

### Consequences
- (+) Confidence in code changes
- (+) Easy refactoring
- (+) Documentation through tests
- (+) Regression prevention
- (-) Time investment for testing
- (-) Test maintenance overhead
- (-) False positives possible

---

## ADR-014: Configuration Management

**Date**: 2024-06-25
**Status**: Accepted
**Context**: Memilih configuration strategy

### Decision
Menggunakan environment variables (.env files) dengan dotenv package.

### Rationale
1. **Security**: Sensitive data tidak di-commit ke repo
2. **Environment Parity**: Sama config format untuk dev/staging/prod
3. **Simplicity**: Tidak perlu config management tools untuk MVP
4. **Flexibility**: Easy untuk override per environment

### Configuration Hierarchy
```
1. .env.local (personal overrides, gitignored)
2. .env.{ENVIRONMENT} (environment specific)
3. .env (defaults)
4. Environment variables
```

### Validation
```javascript
// Validate required env vars on startup
const schema = z.object({
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  API_PORT: z.string().default('3000').transform(Number)
})

const config = schema.parse(process.env)
```

### Consequences
- (+) Simple and straightforward
- (+) Good for small-medium projects
- (+) Easy local development
- (-) Not ideal for complex configurations
- (-) Limited to environment variables
- (-) Scaling challenges

### Future Enhancement
- Migrate to HashiCorp Vault untuk secrets management
- Implement dynamic configuration reloading
- Configuration versioning in production

---

## Decision Log Summary

| ADR | Decision | Status |
|-----|----------|--------|
| 001 | Monolithic Backend | Accepted |
| 002 | PostgreSQL Database | Accepted |
| 003 | Prisma ORM | Accepted |
| 004 | JWT Authentication | Accepted |
| 005 | Redis Caching | Accepted |
| 006 | React + Vite | Accepted |
| 007 | Docker Containerization | Accepted |
| 008 | Kubernetes Orchestration | Accepted |
| 009 | GitHub Actions CI/CD | Accepted |
| 010 | Tailwind CSS | Accepted |
| 011 | API Versioning | Accepted |
| 012 | ELK + Prometheus | Accepted |
| 013 | Jest Testing | Accepted |
| 014 | Environment Variables | Accepted |

---

## How to Add New ADR

1. Use next sequential number (ADR-015, etc)
2. Follow ADR template:
   - Date
   - Status (Proposed, Accepted, Deprecated, Superseded)
   - Context
   - Decision
   - Rationale
   - Consequences
   - Alternatives Considered

3. Commit with message: `docs(adr): ADR-XXX - Title`

4. Update Decision Log

---

**Last Updated**: June 25, 2024
**Version**: 1.0.0
