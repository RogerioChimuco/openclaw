# GitHub Copilot OAuth Integration Analysis

## Overview

This directory contains a comprehensive technical analysis of how OpenClaw integrates with GitHub Copilot using OAuth authentication and token exchange mechanisms.

## Documents

### [GITHUB_COPILOT_OAUTH_ANALISE.md](./GITHUB_COPILOT_OAUTH_ANALISE.md) (Portuguese)

A complete technical report covering:

1. **Architecture Overview**
   - Main components of the integration
   - Data flow diagrams
   - System architecture

2. **OAuth Authentication Flow**
   - OAuth 2.0 Device Flow implementation
   - Step-by-step authentication process
   - Token storage and security

3. **Copilot Token Exchange**
   - GitHub token to Copilot token conversion
   - Token format and parsing
   - Base URL extraction
   - Token caching strategy

4. **Model Discovery and Listing**
   - Available models by subscription
   - Model definition structure
   - Subscription plan differences

5. **Usage Tracking and Limits**
   - Quota monitoring API
   - Usage statistics
   - Plan-based limitations

6. **Python/Django Implementation Guide**
   - Complete code examples
   - Django models, views, and APIs
   - OAuth flow implementation in Python
   - Frontend integration examples
   - Security recommendations
   - Production deployment guidelines

## Key Findings

### How OpenClaw Gets GitHub Copilot OAuth Signature

OpenClaw uses the **OAuth 2.0 Device Authorization Grant** (RFC 8628):

1. Requests a device code from GitHub
2. User authorizes via browser
3. Polls for access token
4. Exchanges GitHub token for Copilot-specific token
5. Uses Copilot token to access AI models

### How It Accesses Models Without API Keys

Instead of traditional API keys, OpenClaw:

- Leverages the user's existing GitHub Copilot subscription
- Uses OAuth tokens that automatically expire for security
- Exchanges tokens at runtime for model access
- Caches tokens to minimize API calls

### How It Discovers Available Models

The implementation:

- Maintains a static list of known Copilot models
- Model availability depends on user's subscription plan
- Attempts to use requested models
- Handles errors when models are unavailable
- Does NOT query a models list API (no such endpoint exists)

### Models by Subscription

- **Free**: Limited basic models
- **Individual**: gpt-4o, claude-sonnet, etc.
- **Business**: Advanced models
- **Enterprise**: All models including o1, o3-mini

### Usage Tracking

OpenClaw monitors usage via:

- `https://api.github.com/copilot_internal/user` endpoint
- Tracks premium interactions quota
- Tracks chat quota
- Reports subscription plan information

## Implementation Files in OpenClaw

- `src/providers/github-copilot-auth.ts` - OAuth Device Flow
- `src/providers/github-copilot-token.ts` - Token exchange and caching
- `src/providers/github-copilot-models.ts` - Model definitions
- `src/infra/provider-usage.fetch.copilot.ts` - Usage monitoring
- `extensions/copilot-proxy/` - Local proxy plugin

## For Developers

The analysis includes a complete Python/Django implementation guide with:

- OAuth Device Flow in Python
- Copilot API client
- Django models for auth profiles and token caching
- API endpoints for chat completions
- Frontend JavaScript examples
- Security best practices

See the [full analysis](./GITHUB_COPILOT_OAUTH_ANALISE.md) for detailed code examples and implementation instructions.

## Security Considerations

⚠️ Important security notes:

1. Store tokens encrypted at rest
2. Use HTTPS in production
3. Implement rate limiting
4. Log all API requests for audit
5. Rotate tokens automatically
6. The `copilot_internal` API is unofficial and may change

## References

- [RFC 8628 - OAuth 2.0 Device Authorization Grant](https://datatracker.ietf.org/doc/html/rfc8628)
- [GitHub OAuth Apps](https://docs.github.com/en/developers/apps/building-oauth-apps)
- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [Django Documentation](https://docs.djangoproject.com/)

## Questions?

For questions about this analysis or implementation details, please refer to the comprehensive documentation in [GITHUB_COPILOT_OAUTH_ANALISE.md](./GITHUB_COPILOT_OAUTH_ANALISE.md).

---

**Created**: February 11, 2026  
**Source**: Analysis of OpenClaw codebase  
**Language**: Portuguese (main document) / English (this README)
