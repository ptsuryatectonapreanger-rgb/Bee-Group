# Performance Optimization Guide

Panduan lengkap untuk mengoptimalkan performa Bee Group Platform.

---

## 📊 Performance Metrics

### Key Performance Indicators (KPIs)

```
Frontend:
  - First Contentful Paint (FCP): < 2 seconds
  - Largest Contentful Paint (LCP): < 2.5 seconds
  - Cumulative Layout Shift (CLS): < 0.1
  - Time to Interactive (TTI): < 3.8 seconds
  - Page Load Time: < 3 seconds

Backend:
  - API Response Time: < 200ms (95th percentile)
  - Database Query Time: < 50ms (average)
  - Cache Hit Rate: > 80%
  - Error Rate: < 0.1%
  - Throughput: > 1000 requests/second

Database:
  - Query Execution Time: < 100ms
  - Connection Pool Utilization: < 80%
  - Replication Lag: < 1 second
  - Backup Time: < 5 minutes
```

---

## 🚀 Frontend Performance Optimization

### Code Splitting & Lazy Loading

```javascript
// App.jsx - Code splitting with React.lazy
import { Suspense, lazy } from 'react'
import Loading from './components/Loading'

const Dashboard = lazy(() => import('./pages/Dashboard'))
const Companies = lazy(() => import('./pages/Companies'))
const Portfolio = lazy(() => import('./pages/Portfolio'))
const Reports = lazy(() => import('./pages/Reports'))

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/companies" element={<Companies />} />
        <Route path="/portfolio" element={<Portfolio />} />
        <Route path="/reports" element={<Reports />} />
      </Routes>
    </Suspense>
  )
}
```

### Bundle Size Optimization

```javascript
// vite.config.js - Optimize build
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui-lib': ['@reduxjs/toolkit', 'react-redux'],
          'charts': ['recharts'],
          'icons': ['lucide-react']
        }
      }
    },
    // Minimize bundle size
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true
      }
    }
  },
  // Tree shaking
  ssr: {
    noExternal: ['recharts']
  }
})
```

### Image Optimization

```javascript
// ImageOptimizer component
import { useState, useEffect } from 'react'

function OptimizedImage({ src, alt, width, height }) {
  const [imageSrc, setImageSrc] = useState(null)
  const [isLoading, setIsLoading] = useState(true)

  useEffect(() => {
    // Use Image API to detect support for modern formats
    const img = new Image()
    
    // Try WebP first (modern browsers)
    img.onload = () => setImageSrc(src.replace(/\.\w+$/, '.webp'))
    img.onerror = () => setImageSrc(src) // Fallback to original
    
    img.src = src
    setIsLoading(false)
  }, [src])

  return (
    <img
      src={imageSrc}
      alt={alt}
      width={width}
      height={height}
      loading="lazy"
      decoding="async"
      className="optimized-image"
    />
  )
}

export default OptimizedImage
```

### Caching Strategy

```javascript
// utils/cache.js
export class CacheManager {
  constructor() {
    this.cache = new Map()
    this.ttl = new Map()
  }

  set(key, value, ttlSeconds = 300) {
    this.cache.set(key, value)
    
    // Set TTL
    const timeout = setTimeout(() => {
      this.cache.delete(key)
      this.ttl.delete(key)
    }, ttlSeconds * 1000)
    
    this.ttl.set(key, timeout)
  }

  get(key) {
    return this.cache.get(key)
  }

  has(key) {
    return this.cache.has(key)
  }

  clear() {
    this.cache.clear()
    this.ttl.forEach(timeout => clearTimeout(timeout))
    this.ttl.clear()
  }
}

// Usage in Redux
import { createAsyncThunk, createSlice } from '@reduxjs/toolkit'

const cache = new CacheManager()

export const fetchCompanies = createAsyncThunk(
  'companies/fetchCompanies',
  async (params, { rejectWithValue }) => {
    const cacheKey = `companies_${JSON.stringify(params)}`
    
    // Check cache
    if (cache.has(cacheKey)) {
      return cache.get(cacheKey)
    }
    
    try {
      const response = await fetch(`/api/companies?${new URLSearchParams(params)}`)
      const data = await response.json()
      
      // Cache for 5 minutes
      cache.set(cacheKey, data, 300)
      
      return data
    } catch (error) {
      return rejectWithValue(error.message)
    }
  }
)
```

### Virtual Scrolling for Large Lists

```javascript
// components/VirtualList.jsx
import { FixedSizeList } from 'react-window'

function CompanyList({ companies }) {
  const Row = ({ index, style }) => (
    <div style={style} className="company-row">
      <span>{companies[index].name}</span>
      <span>{companies[index].code}</span>
      <span>{companies[index].status}</span>
    </div>
  )

  return (
    <FixedSizeList
      height={600}
      itemCount={companies.length}
      itemSize={50}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  )
}

export default CompanyList
```

### Service Worker for Offline Support

```javascript
// public/sw.js - Service Worker
const CACHE_NAME = 'bee-group-v1'
const urlsToCache = [
  '/',
  '/index.html',
  '/app.js',
  '/app.css'
]

// Install event
self.addEventListener('install', event => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(urlsToCache))
  )
})

// Fetch event
self.addEventListener('fetch', event => {
  event.respondWith(
    caches.match(event.request)
      .then(response => response || fetch(event.request))
      .catch(() => caches.match('/offline.html'))
  )
})
```

---

## ⚡ Backend Performance Optimization

### Database Query Optimization

#### N+1 Query Prevention

```javascript
// ❌ N+1 problem
const companies = await prisma.company.findMany()
for (const company of companies) {
  company.assets = await prisma.asset.findMany({
    where: { companyId: company.id }
  })
}

// ✅ Use include (single query)
const companies = await prisma.company.findMany({
  include: {
    assets: true,
    portfolioItems: true,
    financialRecords: {
      take: 10 // Limit related records
    }
  }
})
```

#### Query Indexing

```javascript
// schema.prisma - Add indexes
model Company {
  id        String     @id @default(cuid())
  name      String
  code      String     @unique
  status    String
  createdAt DateTime   @default(now())

  // Composite index for common queries
  @@index([status, createdAt])
  @@index([code])
}

model FinancialRecord {
  id        String     @id @default(cuid())
  companyId String
  type      String
  amount    BigInt
  date      DateTime

  company   Company    @relation(fields: [companyId], references: [id])

  // Index for filtering
  @@index([companyId])
  @@index([companyId, type, date])
}

model ActivityLog {
  id        String     @id @default(cuid())
  userId    String
  action    String
  createdAt DateTime   @default(now())

  user      User       @relation(fields: [userId], references: [id])

  // Index for audit queries
  @@index([userId])
  @@index([createdAt])
  @@index([userId, createdAt])
}
```

#### Query Pagination

```javascript
// controllers/companyController.js
export const getCompanies = async (req, res) => {
  const page = parseInt(req.query.page) || 1
  const limit = Math.min(parseInt(req.query.limit) || 10, 100)
  const skip = (page - 1) * limit

  // Get total count (can be cached)
  const total = await prisma.company.count()

  // Get paginated results
  const companies = await prisma.company.findMany({
    skip,
    take: limit,
    select: {
      id: true,
      name: true,
      code: true,
      status: true,
      // Don't select large fields
      _count: {
        select: { assets: true }
      }
    },
    orderBy: { createdAt: 'desc' }
  })

  res.json({
    data: companies,
    pagination: {
      page,
      limit,
      total,
      pages: Math.ceil(total / limit)
    }
  })
}
```

### Caching Strategy

#### Redis Integration

```javascript
// middleware/cache.js
import redis from './redis'

export const cacheMiddleware = (durationSeconds) => {
  return async (req, res, next) => {
    // Only cache GET requests
    if (req.method !== 'GET') {
      return next()
    }

    const key = `${req.originalUrl}`
    
    try {
      const cachedData = await redis.get(key)
      if (cachedData) {
        res.setHeader('X-Cache', 'HIT')
        return res.json(JSON.parse(cachedData))
      }
    } catch (error) {
      console.error('Cache error:', error)
    }

    // Intercept response
    const originalJson = res.json
    res.json = function(data) {
      try {
        redis.setex(key, durationSeconds, JSON.stringify(data))
        res.setHeader('X-Cache', 'MISS')
      } catch (error) {
        console.error('Cache write error:', error)
      }
      return originalJson.call(this, data)
    }

    next()
  }
}

// Apply to endpoints
router.get('/companies', cacheMiddleware(300), getCompanies)
router.get('/portfolio', cacheMiddleware(600), getPortfolio)
```

#### Cache Invalidation

```javascript
// services/companyService.js
export const updateCompany = async (id, data) => {
  const company = await prisma.company.update({
    where: { id },
    data
  })

  // Invalidate related caches
  await redis.del(`/api/companies`)
  await redis.del(`/api/companies/${id}`)
  await redis.del(`/api/portfolio`)
  
  // Publish event for other services
  await redis.publish('company:updated', JSON.stringify({
    id,
    changes: data
  }))

  return company
}
```

### Connection Pooling

```javascript
// config/database.js
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL
    }
  }
})

// Connection pool configuration
const prismaConfig = {
  // Maximum number of connections
  connection_limit: 20,
  // Idle timeout in seconds
  idle_timeout: 300,
  // Connection string
  url: process.env.DATABASE_URL
}

export default prisma
```

### Response Compression

```javascript
// middleware/compression.js
import compression from 'compression'

app.use(compression({
  level: 6, // Compression level (1-9)
  threshold: 1024, // Compress responses > 1KB
  filter: (req, res) => {
    // Don't compress if request has no-compression header
    if (req.headers['x-no-compression']) {
      return false
    }
    return compression.filter(req, res)
  }
}))
```

### Request/Response Optimization

```javascript
// Reduce payload size
export const getCompanies = async (req, res) => {
  const companies = await prisma.company.findMany({
    // Only select needed fields
    select: {
      id: true,
      name: true,
      code: true,
      status: true
      // Skip large fields like 'description'
    },
    take: 50 // Limit results
  })

  // Remove null/undefined values
  const cleanData = companies.map(c => {
    const cleaned = {}
    Object.keys(c).forEach(key => {
      if (c[key] != null) {
        cleaned[key] = c[key]
      }
    })
    return cleaned
  })

  res.json(cleanData)
}
```

---

## 🗄️ Database Performance Optimization

### Query Analysis

```sql
-- Analyze query performance
EXPLAIN ANALYZE SELECT * FROM companies 
WHERE status = 'ACTIVE' AND created_at > NOW() - INTERVAL '30 days';

-- Check missing indexes
SELECT * FROM pg_stat_user_indexes 
WHERE idx_scan = 0 
ORDER BY pg_relation_size(indexrelid) DESC;
```

### Connection Pool Tuning

```yaml
# PostgreSQL configuration
max_connections: 200
shared_buffers: 256MB
effective_cache_size: 1GB
work_mem: 16MB
checkpoint_completion_target: 0.9
wal_buffers: 16MB
```

### Replication Setup

```yaml
# Primary database configuration
wal_level: replica
max_wal_senders: 10
max_replication_slots: 10

# Standby database
primary_conninfo: 'host=primary_host user=replication password=password'
restore_command: 'cp /wal_archive/%f %p'
```

---

## 🔍 Monitoring & Profiling

### Application Performance Monitoring (APM)

```javascript
// middleware/apm.js
import apm from 'elastic-apm-node'

apm.start({
  serviceName: 'bee-group-api',
  serverUrl: process.env.APM_SERVER_URL
})

// Capture transactions
app.use((req, res, next) => {
  const transaction = apm.startTransaction(`${req.method} ${req.path}`)
  
  res.on('finish', () => {
    transaction.result = res.statusCode
    transaction.end()
  })
  
  next()
})

// Capture database queries
const captureQuery = apm.captureAsyncSpan('database.query', async () => {
  return await prisma.company.findMany()
})
```

### Performance Benchmarking

```javascript
// utils/benchmark.js
export class Benchmark {
  static async measure(name, fn) {
    const start = performance.now()
    const result = await fn()
    const duration = performance.now() - start
    
    console.log(`${name}: ${duration.toFixed(2)}ms`)
    
    return { result, duration }
  }

  static measureSync(name, fn) {
    const start = performance.now()
    const result = fn()
    const duration = performance.now() - start
    
    console.log(`${name}: ${duration.toFixed(2)}ms`)
    
    return { result, duration }
  }
}

// Usage
await Benchmark.measure('Fetch companies', async () => {
  return await prisma.company.findMany()
})
```

### Load Testing

```bash
# Using Apache Bench
ab -n 10000 -c 100 http://localhost:3000/api/companies

# Using wrk
wrk -t12 -c400 -d30s http://localhost:3000/api/companies

# Using Artillery
artillery quick --count 100 --num 1000 http://localhost:3000/api/health
```

---

## 📈 Scalability Strategies

### Horizontal Scaling

```yaml
# Kubernetes HPA (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: bee-group-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bee-group-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Vertical Scaling

```yaml
# Increase resource limits
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bee-group-api
spec:
  template:
    spec:
      containers:
      - name: api
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
```

### Database Sharding Strategy

```javascript
// Get shard ID based on company ID
function getShardId(companyId, totalShards) {
  const hash = companyId
    .split('')
    .reduce((a, b) => a + b.charCodeAt(0), 0)
  return hash % totalShards
}

// Route to correct shard
const shard = getShardId(companyId, 4)
const connection = shardConnections[shard]
const company = await connection.query('SELECT * FROM companies WHERE id = ?', [companyId])
```

---

## 🎯 Frontend Performance Checklist

- [ ] Code splitting implemented
- [ ] Lazy loading for routes
- [ ] Images optimized (WebP format)
- [ ] Bundle size < 300KB
- [ ] Caching enabled
- [ ] Service Worker configured
- [ ] CSS minified
- [ ] JavaScript minified
- [ ] Tree shaking enabled
- [ ] Virtual scrolling for lists

## 🎯 Backend Performance Checklist

- [ ] Database indexes optimized
- [ ] Query N+1 problems fixed
- [ ] Pagination implemented
- [ ] Redis caching enabled
- [ ] Connection pooling configured
- [ ] Response compression enabled
- [ ] Rate limiting configured
- [ ] API response time < 200ms
- [ ] Error handling optimized
- [ ] Logging optimized

## 🎯 Database Performance Checklist

- [ ] Indexes created for common queries
- [ ] Query analysis completed
- [ ] Replication configured
- [ ] Backups automated
- [ ] Connection limits tuned
- [ ] Slow query log enabled
- [ ] Statistics updated
- [ ] Vacuum/ANALYZE scheduled
- [ ] Archive strategy implemented
- [ ] Monitoring enabled

---

**Last Updated**: June 25, 2024
**Version**: 1.0.0
