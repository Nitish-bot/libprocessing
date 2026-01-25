# Codebase Analysis Documentation

This directory contains comprehensive documentation of the libprocessing codebase, generated through systematic analysis of the source code.

## Main Documents

**[CODEBASE_KNOWLEDGE.md](./CODEBASE_KNOWLEDGE.md)** - Complete technical reference covering:

- **High-Level Overview** - What the library is, its purpose, features, and tech stack
- **System Architecture** - Component overview, data flow, architectural decisions
- **Feature-by-Feature Analysis** - Detailed breakdown of each major feature
- **Nuances, Subtleties & Gotchas** - Critical design decisions, performance considerations, security implications
- **Technical Reference & Glossary** - Domain terms, key classes, APIs, ECS schema
- **State Block** - File index, open questions, known risks

**[GEOMETRY_PYO3_GUIDE.md](./GEOMETRY_PYO3_GUIDE.md)** - Comprehensive guide for:
- Understanding the geometry module architecture
- Exposing geometry functionality to PyO3
- Adding new primitive shapes (sphere, cube, cylinder, etc.)
- Best practices and implementation patterns

## Usage

This documentation is designed for:

1. **New Contributors** - Understand the codebase structure and design decisions
2. **Feature Implementers** - Know where to make changes and what to consider
3. **Bug Fixers** - Understand edge cases and gotchas
4. **Refactorers** - Understand architectural constraints and patterns

## Quick Navigation

### Understanding the Architecture
- Start with [High-Level Overview](./CODEBASE_KNOWLEDGE.md#high-level-overview)
- Then read [System Architecture](./CODEBASE_KNOWLEDGE.md#system-architecture)

### Implementing a Feature
1. Check [Feature-by-Feature Analysis](./CODEBASE_KNOWLEDGE.md#feature-by-feature-analysis) for similar features
2. Review [Nuances & Gotchas](./CODEBASE_KNOWLEDGE.md#nuances-subtleties--gotchas) for constraints
3. Consult [Technical Reference](./CODEBASE_KNOWLEDGE.md#technical-reference--glossary) for APIs

### Adding Geometry Primitives
1. See [Geometry & PyO3 Guide](./GEOMETRY_PYO3_GUIDE.md) for step-by-step instructions
2. Follow the pattern established by `create_box()` function
3. Use Bevy's `Meshable` primitives (Sphere, Cylinder, Torus, etc.)

### Fixing a Bug
1. Check [Nuances & Gotchas](./CODEBASE_KNOWLEDGE.md#nuances-subtleties--gotchas) for known issues
2. Review relevant feature section in [Feature-by-Feature Analysis](./CODEBASE_KNOWLEDGE.md#feature-by-feature-analysis)
3. Check [State Block](./CODEBASE_KNOWLEDGE.md#state-block) for known risks

## Document Structure

The knowledge document follows a structured approach:

1. **Phase 1** - Initial context scan (what, why, how)
2. **Phase 2** - System architecture deep dive
3. **Phase 3** - Feature-by-feature analysis
4. **Phase 4** - Nuances and gotchas
5. **Phase 5** - Technical reference
6. **Phase 6** - Final assembly with state tracking

## Assets Directory

The `assets/` directory is reserved for:
- Architecture diagrams (if needed)
- Schema diagrams (if needed)
- Additional visual documentation

## Maintenance

This documentation should be updated when:
- Major architectural changes occur
- New features are added
- Critical gotchas are discovered
- API changes are made

## Related Documentation

- [Project Principles](../docs/principles.md) - Design principles and rationale
- [API Documentation](../docs/api.md) - API object documentation
- [README](../README.md) - Project overview and getting started

---

**Last Updated:** 2026-01-25  
**Document Version:** 1.0
