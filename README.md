# Little Trader — Documentation & Learning Resources

Complete documentation and learning materials for the **Little Trader** advanced trading system.

## Overview

This repository contains:

- **📚 [docs/](./docs)** — Complete technical documentation
- **🎓 [learn/](./learn)** — Learning materials, courses, and guides

## Quick Links

### Documentation

- [Architecture Guide](./docs/ARCHITECTURE.md) — System design and components
- [Getting Started](./docs/README.md) — Project overview
- [Implementation Guide](./docs/IMPLEMENTATION.md) — How to build and deploy
- [DevOps & Infrastructure](./docs/DEVOPS.md) — Cloud setup and operations
- [Authentication](./docs/AUTH_ARCHITECTURE.md) — Auth system design

### Learning

- [Learning Path](./learn/little-trader-learning-path.md) — Recommended learning sequence
- [Course Curriculum](./learn/course-curriculum.md) — Full course structure
- [REPL First Development](./learn/repl-first.md) — Interactive development guide

## Documentation Structure

### By Topic

- **Architecture** — System design, data models, component interactions
- **Features** — Trading features, signal detection, execution
- **Implementation** — Setup guides, deployment, status tracking
- **DevOps** — Cloud infrastructure, operations, monitoring
- **Development** — Coding practices, testing, MCP integration
- **Strategy** — Quantitative analysis, research, trading strategies
- **Learning** — Courses, tutorials, onboarding materials

### Complete Doc Index

| Category | Documents |
|----------|-----------|
| Getting Started | README, Features, Implementation Guide |
| Architecture | Architecture, Auth, Frontend, C4 Diagrams |
| Strategy & Research | ML, AI Trading Papers, Strategy Engine |
| Implementation | Status, Project Status, Executor Platform |
| DevOps & Infrastructure | DevOps, Cloud Setup, Cloud MCP, SRE |
| Development | Claude AI Guide, MCP, AI Integration, Testing |
| Learning | Course, Curriculum, Onboarding, Options 101 |

## Technology Stack

**Backend**
- Clojure, Ring+Reitit, HTTP-Kit
- Datomic Pro (PostgreSQL)
- Pathom3, Mount

**Frontend**
- ClojureScript, Fulcro 3.7.10
- Fulcro RAD, Semantic UI
- shadow-cljs

**Rules & Execution**
- Clara Rules 0.21.1
- Custom execution engine
- Kafka/Pulsar integration

**Infrastructure**
- Cloud Run, Cloud SQL
- Kubernetes (in progress)
- GitHub Actions CI/CD

## Getting Started

### For Learning

1. Start with [Learning Path](./learn/little-trader-learning-path.md)
2. Follow the [Course Curriculum](./learn/course-curriculum.md)
3. Reference specific [Documentation](./docs) as needed

### For Development

1. Read [Architecture Guide](./docs/ARCHITECTURE.md)
2. Follow [Implementation Guide](./docs/IMPLEMENTATION.md)
3. Check [DevOps Guide](./docs/DEVOPS.md) for deployment
4. See [CLAUDE.md](./docs/CLAUDE.md) for AI-assisted development

### For Operations

1. Review [DevOps Guide](./docs/DEVOPS.md)
2. Check [SRE Operations](./docs/SRE_OPERATIONS.md)
3. Follow [Cloud Deploy](./docs/CLOUD_DEPLOY.md)

## Key Features

### Trading System

- **Renko Chart Analysis** — Brick pattern detection and signal generation
- **Multi-Strategy Support** — Backtest, paper trading, live execution
- **Real-Time Execution** — Direct market integration via fs-worker
- **Comprehensive Backtesting** — Historical analysis with detailed metrics
- **Risk Management** — Dynamic position sizing, stop losses, profit targets

### Platform

- **Web UI** — Full-featured Fulcro frontend with real-time updates
- **API-First** — EQL-based GraphQL for all operations
- **REPL-Driven** — Interactive development with live code reloading
- **Cloud-Native** — Kubernetes ready, scalable architecture
- **Learn Tab** — Integrated course materials and documentation

## Contributing

This repository contains documentation and learning materials. To contribute:

1. Make changes in the appropriate directory (`docs/` or `learn/`)
2. Follow markdown formatting conventions
3. Link to related documents
4. Submit PR to main branch

## Resources

- **GitHub Repository** — [github.com/4coders-com-br/little-trader](https://github.com/4coders-com-br/little-trader)
- **4coders Website** — https://4coders.com.br
- **Contact** — contato@4coders.com.br

## License

All documentation and learning materials are provided as-is for the Little Trader project.

---

**Last Updated:** April 2026  
**Maintained by:** 4coders Team  
**Version:** 1.0
