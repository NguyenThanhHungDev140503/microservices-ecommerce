# MCP Server Test Report: Authentication System Analysis

**Test Date:** February 16, 2026  
**Use Case:** Phân tích hệ thống authentication trong dự án ProgCoder Shop Microservices  
**Project:** .NET 8 Clean Architecture + DDD microservices

## Executive Summary

| Metric | code-graph-rag | relation-sematic-search |
|--------|----------------|-------------------------|
| **Overall Score** | ⭐⭐⭐⭐ (4/5) | ⭐⭐ (2/5) |
| **Indexing** | ✅ Fast & Accurate | ❌ Mixed wrong directories |
| **Natural Language Query** | ✅ Good results | ❌ Irrelevant results |
| **Entity Resolution** | ✅ Accurate | ⚠️ Partially working |
| **Relationship Analysis** | ⚠️ Limited for C# | ❌ Not working |
| **Code Retrieval** | ✅ Working | ❌ Wrong files |

---

## Test Results

### 1. code-graph-rag (Memgraph-based)

#### ✅ Strengths

1. **Natural Language Queries Work Well**
   - Query: "What classes and methods are responsible for JWT authentication?"
   - Found: 50 relevant entities including:
     - `AuthenticationExtensions` - main JWT setup
     - `IKeycloakApi` - interface across 4 services (Report, Notification, Inventory, JobOrchestrator)
     - `KeycloakAccessToken`, `KeycloakAccessTokenDto`, `KeycloakUser` models
     - `KeycloakService` implementations

2. **Cross-Service Analysis**
   - Correctly identified that each microservice has its own copy of `IKeycloakApi`
   - Found service-to-service authentication pattern via Keycloak access tokens

3. **Code Snippet Retrieval**
   - Successfully retrieved `KeycloakAccessToken` class definition with full source code

4. **Fast Indexing**
   - Completed repository indexing in seconds
   - No configuration required

#### ⚠️ Limitations

1. **C# Support Limited**
   - Could not find methods in `AuthenticationExtensions` class (returned 0 results)
   - Could not find classes implementing `IKeycloakApi` interface
   - Better suited for JavaScript/TypeScript codebases (as documented)

2. **Implementation Tracking**
   - Interface implementations not properly tracked for C#
   - Inheritance relationships partially missing

---

### 2. relation-sematic-search (Neo4j-based with Semantic Search)

#### ❌ Major Issues Found

1. **Wrong Indexing Directories**
   ```
   Indexed sources include:
   - /home/nguyen-thanh-hung/LightRAG/ (Python source code!)
   - /home/nguyen-thanh-hung/Documents/Code/progcoder-shop-microservices/ (correct)
   ```
   **Problem:** Server mixed multiple data sources, polluting the graph

2. **Semantic Search Completely Broken**
   Query: "How to get access token from Keycloak"
   
   Results returned:
   | # | File | Language | Similarity | Relevant? |
   |---|------|----------|------------|-----------|
   | 1 | `shared_storage.py` | Python | 0.44 | ❌ |
   | 2 | `ThumbnailUrl` property | C# | 0.44 | ❌ |
   | 3 | `CreateReservationMappings` | C# | 0.44 | ❌ |
   | 4 | React Table.tsx | TypeScript | 0.44 | ❌ |
   | 5 | Graph edge selection | Python | 0.44 | ❌ |
   
   **Problem:** No authentication-related code found despite correct indexing

3. **Entity Resolution Partial**
   - Could find `KeycloakService` entity but with 0 relationships
   - Could find `KeycloakAccessToken` but relationships not extracted

4. **Relationship Extraction Failed for C#**
   - `list_entity_relationships()` returned empty or unrelated entities
   - C# relationship types not properly mapped

---

## Architecture Discovered

### Authentication System in ProgCoder Shop

```
┌─────────────────────────────────────────────────────────────┐
│                   Authentication Flow                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  User Request ──▶ API Gateway (YARP)                        │
│                          │                                   │
│                          ▼                                   │
│                  JWT Bearer Token Validation                 │
│                          │                                   │
│                          ▼                                   │
│              ┌───────────────────────┐                      │
│              │  Keycloak Server      │                      │
│              │  (Identity Provider)  │                      │
│              └───────────────────────┘                      │
│                          │                                   │
│                          ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │         Service-to-Service Authentication            │   │
│  ├─────────────────────────────────────────────────────┤   │
│  │  Report Service     ──┐                              │   │
│  │  Notification Svc   ──┤    IKeycloakApi             │   │
│  │  Inventory Service  ──┼──▶ GetAccessTokenAsync()    │   │
│  │  JobOrchestrator    ──┘    KeycloakService          │   │
│  │                                                         │   │
│  │  Each service has its own copy of IKeycloakApi       │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Key Components Found

| Component | Location | Purpose |
|-----------|----------|---------|
| `AuthenticationExtensions` | Shared/BuildingBlocks | Main JWT setup configuration |
| `IKeycloakApi` | Each service's Infrastructure | HTTP client for Keycloak |
| `KeycloakService` | Service Infrastructure | Token acquisition logic |
| `KeycloakAccessToken` | Domain Models | Access token model |
| `AuthorizeRole` | Shared constants | Role-based authorization |
| `UnauthorizedException` | BuildingBlocks | Auth error handling |

---

## Recommendations

### For code-graph-rag
✅ **Recommended for this project** (C# microservices)

**Best Use Cases:**
- Finding classes and interfaces across the codebase
- Natural language queries about architecture
- Code snippet retrieval by qualified name
- Cross-service analysis

**Limitations to Note:**
- C# method-level relationships limited
- Interface implementations may not be tracked

### For relation-sematic-search
❌ **Not recommended for C# projects**

**Issues to Fix:**
1. **Indexing configuration** - Should allow specifying single directory
2. **Semantic search** - Needs training on C# code or better embeddings
3. **Relationship extraction** - C# parser needs improvement
4. **Directory isolation** - Prevent mixing multiple codebases

**When to Use:**
- JavaScript/TypeScript projects (as designed)
- When semantic search is working correctly
- For finding similar code patterns (if embeddings trained)

---

## Test Coverage

| Test Case | code-graph-rag | relation-sematic-search |
|-----------|----------------|-------------------------|
| Index repository | ✅ Pass | ❌ Fail (mixed sources) |
| Natural language query | ✅ Pass | ❌ Fail (irrelevant results) |
| Find authentication classes | ✅ Pass | ❌ Fail |
| Find entity by name | ✅ Pass | ⚠️ Partial |
| Get entity relationships | ⚠️ Partial (C# limited) | ❌ Fail |
| Retrieve code snippet | ✅ Pass | ❌ Fail (wrong files) |
| Cross-service analysis | ✅ Pass | ❌ Fail |

---

## Conclusion

**code-graph-rag** is the clear winner for analyzing C# microservices projects. It provides reliable natural language querying, accurate entity resolution, and useful cross-service analysis despite some C# limitations.

**relation-sematic-search** requires significant improvements before being useful for C# projects, particularly in:
1. Directory configuration and isolation
2. Semantic search accuracy for C# code
3. C# relationship extraction

**Verdict:** Use **code-graph-rag** for authentication system analysis in this .NET project.
