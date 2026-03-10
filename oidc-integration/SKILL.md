---
name: oidc-integration
description: Guidelines for integrating OIDC providers (e.g., Authing, IdentityServer, Keycloak) into frontend and backend applications. Use when implementing authentication, token refresh, or multi-provider auth systems.
---

# OIDC Provider Integration Guide

## Overview

This skill covers integrating OpenID Connect (OIDC) providers into web applications with:
- Frontend (React/TypeScript) authentication flow
- Backend (Java/Spring Boot) token validation
- Dual-provider support (e.g., admin vs user auth)
- Token refresh mechanism
- PKCE support for public clients

## When to Use

- Adding OIDC authentication to an application
- Integrating multiple OIDC providers
- Implementing token refresh with offline_access
- Handling 401 responses with auto re-login
- Setting up protected routes

## Frontend Integration (React + TypeScript)

### 1. OIDC Configuration File

Create a config file for each OIDC provider:

```typescript
// src/config/userOidc.ts
export const USER_OIDC_CONFIG = {
  authority: import.meta.env.VITE_USER_OIDC_AUTHORITY, // e.g., https://idp.example.com
  clientId: import.meta.env.VITE_USER_OIDC_CLIENT_ID,
  redirectUri: `${window.location.origin}/auth/callback`,
  scope: import.meta.env.VITE_USER_OIDC_SCOPE || 'openid profile email offline_access',
  responseType: 'code',
};

// PKCE utilities
export function generateCodeVerifier(): string {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

export async function generateCodeChallenge(verifier: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const digest = await crypto.subtle.digest('SHA-256', data);
  return base64UrlEncode(new Uint8Array(digest));
}

function base64UrlEncode(buffer: Uint8Array): string {
  return btoa(String.fromCharCode(...buffer))
    .replace(/\+/g, '-')
    .replace(/\//g, '_')
    .replace(/=+$/, '');
}

// Build authorization URL
export async function getUserAuthorizationUrl(): Promise<string> {
  const codeVerifier = generateCodeVerifier();
  const codeChallenge = await generateCodeChallenge(codeVerifier);
  
  // Store verifier for token exchange
  sessionStorage.setItem('oidc_code_verifier', codeVerifier);
  sessionStorage.setItem('oidc_redirect_uri', USER_OIDC_CONFIG.redirectUri);

  const params = new URLSearchParams({
    client_id: USER_OIDC_CONFIG.clientId,
    redirect_uri: USER_OIDC_CONFIG.redirectUri,
    response_type: 'code',
    scope: USER_OIDC_CONFIG.scope,
    code_challenge: codeChallenge,
    code_challenge_method: 'S256',
    prompt: 'login', // Force login page display
  });

  return `${USER_OIDC_CONFIG.authority}/authorize?${params.toString()}`;
}
```

### 2. Auth Context

```typescript
// src/contexts/AuthContext.tsx
import React, { createContext, useContext, useState, useEffect } from 'react';

interface AuthContextType {
  isAuthenticated: boolean;
  user: User | null;
  accessToken: string | null;
  login: () => Promise<void>;
  logout: () => void;
  refreshToken: () => Promise<boolean>;
}

const AuthContext = createContext<AuthContextType | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [accessToken, setAccessToken] = useState<string | null>(null);

  useEffect(() => {
    // Restore from localStorage on mount
    const savedToken = localStorage.getItem('access_token');
    const savedUser = localStorage.getItem('user');
    if (savedToken && savedUser) {
      setAccessToken(savedToken);
      setUser(JSON.parse(savedUser));
    }
  }, []);

  const login = async () => {
    const url = await getUserAuthorizationUrl();
    window.location.href = url;
  };

  const logout = () => {
    localStorage.removeItem('access_token');
    localStorage.removeItem('refresh_token');
    localStorage.removeItem('user');
    setAccessToken(null);
    setUser(null);
  };

  const refreshToken = async (): Promise<boolean> => {
    const refresh = localStorage.getItem('refresh_token');
    if (!refresh) return false;

    try {
      const response = await fetch(`${USER_OIDC_CONFIG.authority}/token`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
        body: new URLSearchParams({
          grant_type: 'refresh_token',
          client_id: USER_OIDC_CONFIG.clientId,
          refresh_token: refresh,
        }),
      });

      if (!response.ok) return false;

      const data = await response.json();
      localStorage.setItem('access_token', data.access_token);
      if (data.refresh_token) {
        localStorage.setItem('refresh_token', data.refresh_token);
      }
      setAccessToken(data.access_token);
      return true;
    } catch {
      return false;
    }
  };

  return (
    <AuthContext.Provider value={{
      isAuthenticated: !!accessToken,
      user,
      accessToken,
      login,
      logout,
      refreshToken,
    }}>
      {children}
    </AuthContext.Provider>
  );
}

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
};
```

### 3. Auth Callback Handler

```typescript
// src/pages/AuthCallback.tsx
import { useEffect, useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { USER_OIDC_CONFIG } from '../config/userOidc';

export function AuthCallback() {
  const navigate = useNavigate();
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const handleCallback = async () => {
      const params = new URLSearchParams(window.location.search);
      const code = params.get('code');
      const errorParam = params.get('error');

      if (errorParam) {
        setError(params.get('error_description') || errorParam);
        return;
      }

      if (!code) {
        setError('No authorization code received');
        return;
      }

      try {
        // Exchange code for tokens
        const codeVerifier = sessionStorage.getItem('oidc_code_verifier');
        const redirectUri = sessionStorage.getItem('oidc_redirect_uri');

        const response = await fetch(`${USER_OIDC_CONFIG.authority}/token`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
          body: new URLSearchParams({
            grant_type: 'authorization_code',
            client_id: USER_OIDC_CONFIG.clientId,
            code,
            redirect_uri: redirectUri || USER_OIDC_CONFIG.redirectUri,
            code_verifier: codeVerifier || '',
          }),
        });

        if (!response.ok) {
          const errorData = await response.json();
          throw new Error(errorData.error_description || 'Token exchange failed');
        }

        const tokens = await response.json();
        
        // Store tokens
        localStorage.setItem('access_token', tokens.access_token);
        if (tokens.refresh_token) {
          localStorage.setItem('refresh_token', tokens.refresh_token);
        }

        // Cleanup
        sessionStorage.removeItem('oidc_code_verifier');
        sessionStorage.removeItem('oidc_redirect_uri');

        // Get user info (optional)
        const userInfoResponse = await fetch(`${USER_OIDC_CONFIG.authority}/userinfo`, {
          headers: { Authorization: `Bearer ${tokens.access_token}` },
        });
        
        if (userInfoResponse.ok) {
          const userInfo = await userInfoResponse.json();
          localStorage.setItem('user', JSON.stringify(userInfo));
        }

        // Redirect to original location or home
        const returnUrl = sessionStorage.getItem('auth_return_url') || '/';
        sessionStorage.removeItem('auth_return_url');
        navigate(returnUrl, { replace: true });
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Authentication failed');
      }
    };

    handleCallback();
  }, [navigate]);

  if (error) {
    return <div>Error: {error}</div>;
  }

  return <div>Processing authentication...</div>;
}
```

### 4. API Client with Token Refresh

```typescript
// src/services/api.ts
const API_BASE = import.meta.env.VITE_API_URL;

// Check if token is expiring soon (within 5 minutes)
function isTokenExpiringSoon(token: string): boolean {
  try {
    const payload = JSON.parse(atob(token.split('.')[1]));
    const expiresAt = payload.exp * 1000;
    const fiveMinutes = 5 * 60 * 1000;
    return Date.now() > expiresAt - fiveMinutes;
  } catch {
    return false;
  }
}

// Create fetch wrapper with auth
export async function apiFetch(
  endpoint: string,
  options: RequestInit = {}
): Promise<Response> {
  let token = localStorage.getItem('access_token');

  // Pre-request: refresh if expiring soon
  if (token && isTokenExpiringSoon(token)) {
    const refreshed = await refreshUserToken();
    if (refreshed) {
      token = localStorage.getItem('access_token');
    }
  }

  const response = await fetch(`${API_BASE}${endpoint}`, {
    ...options,
    headers: {
      ...options.headers,
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
    },
  });

  // Handle 401 - try refresh once
  if (response.status === 401) {
    const refreshed = await refreshUserToken();
    if (refreshed) {
      token = localStorage.getItem('access_token');
      return fetch(`${API_BASE}${endpoint}`, {
        ...options,
        headers: {
          ...options.headers,
          Authorization: `Bearer ${token}`,
        },
      });
    } else {
      // Dispatch event for auth context to handle
      window.dispatchEvent(new CustomEvent('auth:required'));
    }
  }

  return response;
}
```

### 5. Protected Route Component

```typescript
// src/components/ProtectedRoute.tsx
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  const location = useLocation();

  if (!isAuthenticated) {
    // Save current location for post-login redirect
    sessionStorage.setItem('auth_return_url', location.pathname + location.search);
    return <Navigate to="/login" replace />;
  }

  return <>{children}</>;
}
```

### 6. Environment Variables

```bash
# .env
VITE_API_URL=https://api.example.com
VITE_USER_OIDC_AUTHORITY=https://idp.example.com
VITE_USER_OIDC_CLIENT_ID=your-client-id
VITE_USER_OIDC_SCOPE=openid profile email offline_access
```

## Backend Integration (Java Spring Boot)

### 1. OIDC Configuration

```java
// config/OidcConfig.java
@Configuration
@ConfigurationProperties(prefix = "oidc")
@Data
public class OidcConfig {
    private String jwksUri;
    private String issuer;
    private String audience; // client_id
}
```

### 2. Application Properties

```yaml
# application.yml
oidc:
  jwks-uri: https://idp.example.com/.well-known/openid-configuration/jwks
  issuer: https://idp.example.com
  audience: your-client-id
```

### 3. JWT Token Validator

```java
// service/TokenValidationService.java
@Service
@Slf4j
public class TokenValidationService {
    
    private final OidcConfig oidcConfig;
    private JwkProvider jwkProvider;
    
    public TokenValidationService(OidcConfig oidcConfig) {
        this.oidcConfig = oidcConfig;
        try {
            URL jwksUrl = new URL(oidcConfig.getJwksUri());
            this.jwkProvider = new JwkProviderBuilder(jwksUrl)
                .cached(10, 24, TimeUnit.HOURS)
                .rateLimited(10, 1, TimeUnit.MINUTES)
                .build();
        } catch (MalformedURLException e) {
            throw new RuntimeException("Invalid JWKS URL", e);
        }
    }
    
    public DecodedJWT validateToken(String token) {
        try {
            DecodedJWT jwt = JWT.decode(token);
            
            // Get the key from JWKS
            Jwk jwk = jwkProvider.get(jwt.getKeyId());
            Algorithm algorithm = Algorithm.RSA256((RSAPublicKey) jwk.getPublicKey(), null);
            
            // Verify token
            JWTVerifier verifier = JWT.require(algorithm)
                .withIssuer(oidcConfig.getIssuer())
                .withAudience(oidcConfig.getAudience())
                .build();
            
            return verifier.verify(token);
        } catch (JWTVerificationException | JwkException e) {
            log.warn("Token validation failed: {}", e.getMessage());
            return null;
        }
    }
    
    public String extractUserId(DecodedJWT jwt) {
        // Most OIDC providers use "sub" claim
        return jwt.getSubject();
    }
    
    public String extractEmail(DecodedJWT jwt) {
        return jwt.getClaim("email").asString();
    }
}
```

### 4. Auth Interceptor

```java
// config/AuthInterceptor.java
@Component
@Slf4j
public class AuthInterceptor implements HandlerInterceptor {
    
    private final TokenValidationService tokenService;
    
    // Paths that don't require authentication
    private static final List<String> PUBLIC_PATHS = Arrays.asList(
        "/api/health",
        "/api/public/"
    );
    
    public AuthInterceptor(TokenValidationService tokenService) {
        this.tokenService = tokenService;
    }
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        String path = request.getRequestURI();
        
        // Skip auth for public paths
        if (isPublicPath(path)) {
            return true;
        }
        
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }
        
        String token = authHeader.substring(7);
        DecodedJWT jwt = tokenService.validateToken(token);
        
        if (jwt == null) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }
        
        // Store user info in request for controllers
        request.setAttribute("userId", tokenService.extractUserId(jwt));
        request.setAttribute("email", tokenService.extractEmail(jwt));
        
        return true;
    }
    
    private boolean isPublicPath(String path) {
        return PUBLIC_PATHS.stream().anyMatch(path::startsWith);
    }
}
```

### 5. Web Configuration

```java
// config/WebConfig.java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    private final AuthInterceptor authInterceptor;
    
    public WebConfig(AuthInterceptor authInterceptor) {
        this.authInterceptor = authInterceptor;
    }
    
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(authInterceptor)
            .addPathPatterns("/api/**")
            .excludePathPatterns(
                "/api/health",
                "/api/public/**"
            );
    }
    
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:5173", "https://your-frontend.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true);
    }
}
```

### 6. Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- JWT validation -->
    <dependency>
        <groupId>com.auth0</groupId>
        <artifactId>java-jwt</artifactId>
        <version>4.4.0</version>
    </dependency>
    <dependency>
        <groupId>com.auth0</groupId>
        <artifactId>jwks-rsa</artifactId>
        <version>0.22.1</version>
    </dependency>
</dependencies>
```

## Dual Provider Support

For applications needing different auth providers (e.g., admin vs user):

### Frontend

```typescript
// Determine which provider to use based on route/context
export function getAuthProvider(isAdmin: boolean) {
  return isAdmin ? ADMIN_OIDC_CONFIG : USER_OIDC_CONFIG;
}

// Store which provider was used
localStorage.setItem('auth_provider', isAdmin ? 'admin' : 'user');
```

### Backend

```java
// Support multiple issuers
@Data
public class OidcConfig {
    private List<ProviderConfig> providers;
}

public class ProviderConfig {
    private String name;
    private String jwksUri;
    private String issuer;
    private String audience;
}

// In TokenValidationService, try each provider
public DecodedJWT validateToken(String token) {
    for (ProviderConfig provider : oidcConfig.getProviders()) {
        DecodedJWT result = validateWithProvider(token, provider);
        if (result != null) {
            return result;
        }
    }
    return null;
}
```

## Common Issues & Solutions

### 1. JWKS Endpoint Discovery

Some providers use different JWKS URLs:
- Standard: `/.well-known/openid-configuration/jwks`
- Auth0: `/.well-known/jwks.json`
- Authing: `/.well-known/jwks`

Fetch the OpenID configuration first to discover the correct URL:
```typescript
const config = await fetch(`${authority}/.well-known/openid-configuration`).then(r => r.json());
const jwksUri = config.jwks_uri;
```

### 2. PKCE Required

Many providers now require PKCE for public clients:
- Always use `code_challenge` and `code_challenge_method=S256`
- Store `code_verifier` in sessionStorage (not localStorage)
- Include `code_verifier` in token exchange request

### 3. Token Refresh

To enable token refresh:
1. Include `offline_access` in scope
2. Store `refresh_token` securely
3. Refresh proactively before expiration

### 4. 401 Handling

Best practice for 401 responses:
1. Try token refresh first
2. If refresh fails, redirect to login
3. Save current URL for post-login redirect
4. Dispatch event to notify other components

### 5. CORS Issues

Ensure backend allows:
- Frontend origin(s) in `Access-Control-Allow-Origin`
- Credentials with `Access-Control-Allow-Credentials: true`
- Authorization header in `Access-Control-Allow-Headers`

## Checklist

Before going live:

- [ ] Configure environment variables for each environment
- [ ] Register redirect URIs with OIDC provider
- [ ] Test PKCE flow end-to-end
- [ ] Implement token refresh
- [ ] Handle 401 responses gracefully
- [ ] Set up proper CORS configuration
- [ ] Validate tokens on backend with JWKS
- [ ] Test with expired tokens
- [ ] Implement logout (including provider logout if needed)
- [ ] Add login/logout events for analytics
