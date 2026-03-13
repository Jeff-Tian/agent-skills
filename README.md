# Agent Skills

A collection of agent skills for GitHub Copilot, Claude, and other AI coding assistants.

This repository currently includes an OIDC integration skill focused on React or TypeScript frontends and Java or Spring Boot backends.

## Skills

| Skill | Description | Use Cases |
|-------|-------------|-----------|
| [oidc-integration](./oidc-integration/) | Plan and implement OIDC and OAuth 2.0 integration for React or TypeScript frontends and Java or Spring Boot backends. | Add login with Keycloak or Auth0, wire up PKCE callback handling, validate JWTs with JWKS, support multi-provider auth, or add protected routes and token refresh. |

## Repository Layout

- Main skill file: [oidc-integration/SKILL.md](./oidc-integration/SKILL.md)
- Example references:
	- [oidc-integration/references/react-spa.md](./oidc-integration/references/react-spa.md)
	- [oidc-integration/references/spring-resource-server.md](./oidc-integration/references/spring-resource-server.md)
	- [oidc-integration/references/multi-provider.md](./oidc-integration/references/multi-provider.md)
- License: [LICENSE](./LICENSE)

## Installation

Install all skills from this repository:

```bash
npx skills add Jeff-Tian/agent-skills
```

Install a specific skill:

```bash
npx skills add Jeff-Tian/agent-skills --skill oidc-integration
```

## License

MIT
