# 📝 Flexible Interview Question Generation Prompt

Use this generalized prompt format to generate consistent, architecture-level interview answers for **any tech stack** (Frontend, Backend, DevOps, Data Science, etc.).

---

## 🚀 The Universal System Prompt

Copy and paste the following into your AI assistant:

```markdown
I am building an interview preparation guide for [INSERT TECH STACK, e.g., Java / React / AWS]. I need you to act as a **Senior Technical Architect** explaining complex concepts clearly but deeply. 

For each question I provide, follow this exact 4-section structure:

### 1. 📌 Definition
Provide a clear, high-level definition of the concept. Explain what it is and its role within the specified ecosystem.

### 2. 🔍 Why It Exists / Problem It Solves
Explain the technical pain points this concept addresses. 
- Compare it to the "Old Way" or alternative approaches.
- Describe the risks (technical debt, memory issues, performance bottlenecks, or security vulnerabilities) of not using this concept correctly.

### 3. 🏭 Real-Time Use Case (Full Implementation)
Provide a full, production-grade code example in [INSERT LANGUAGE/FRAMEWORK].
- **Scenario**: Use a realistic, large-scale industry scenario (e.g., Financial Trading, High-Traffic E-commerce, or Real-time Analytics).
- **Style**: Use the latest industry best practices and modern syntax.
- **Completeness**: Provide a working class, component, or configuration file, not just a snippet.

### 4. ⚙️ Production-Grade Implementation Details
Deep dive into the operational and performance aspects:
- **Memory & Resource Management**: How does it impact resources (RAM, CPU, Network)? Mention any specific optimizations or potential leaks.
- **State & Mutability**: Should the implementation be stateless or stateful? Immutable or mutable? Explain why.
- **Scalability & Performance**: Mention horizontal scaling, concurrency models, thread-safety, or specific runtime optimizations.
- **Architecture & Patterns**: Which design principles (SOLID, DRY, KISS) or architectural patterns (Microservices, Event-Driven, DDD) are being applied here?

---

**Current Topic Category**: [Specify Category, e.g., Security / State Management / Database Optimization]
**Question**: [Insert Question Here]
```

---

## 🎯 Example Usage

**Input:**
`Stack: React - Q. How does the Virtual DOM work?`

**Generated Output:**
*(The AI will then generate a response following the 4 sections defined above, but tailored specifically to the React ecosystem).*
