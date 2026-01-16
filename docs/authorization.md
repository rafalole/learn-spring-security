# Spring Security Authorization Process

## Overview
Spring Security authorization determines what authenticated users can access based on their roles and permissions.

## Configuration Analysis

### User Setup
```java
/**
 * AuthenticationManagerBuilder configures authentication providers.
 * InMemoryUserDetailsManagerConfigurer handles in-memory user storage.
 * UserDetailsBuilder creates user with encoded password and roles.
 */
AuthenticationManagerBuilder auth = // injected parameter
auth.inMemoryAuthentication()
    .withUser("user").password(passwordEncoder.encode("pass"))
    .roles("USER");
```
- Creates user "user" with role "USER"
- Password is encoded for security

### Authorization Rules
```java
/**
 * HttpSecurity configures request-based authorization rules.
 * AuthorizeHttpRequestsConfigurer handles URL pattern matching.
 * RequestMatcher implementations check URL patterns.
 * AuthorityAuthorizationManager validates user authorities.
 */
HttpSecurity http = // injected parameter
http.authorizeHttpRequests((requests) -> requests
    .requestMatchers("/delete/**").hasRole("ADMIN")
    .anyRequest().authenticated())
```

## Authorization Alternatives

### Method-Level Security

**Configuration**
```java
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
```

**Usage Patterns**
```java
/**
 * Validates user has ADMIN role before method execution.
 * Processed by ExpressionBasedPreInvocationAdvice which evaluates SpEL expression.
 * Throws AccessDeniedException if user lacks ROLE_ADMIN authority.
 * Use for preventing unauthorized method calls entirely.
 */
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) {}

/**
 * Validates authorization after method execution based on return value.
 * Method executes first, then compares returnObject.owner with current user.
 * If check fails, AccessDeniedException thrown and return value discarded.
 * Use for data-dependent authorization where user can only access own resources.
 */
@PostAuthorize("returnObject.owner == authentication.name")
public Document getDocument(Long id) {}

/**
 * Legacy annotation for simple role-based authorization.
 * Requires exact authority match - no automatic ROLE_ prefix added.
 * Less flexible than @PreAuthorize as it doesn't support SpEL expressions.
 * Use for straightforward role checks without complex logic.
 */
@Secured("ROLE_USER")
public List<Document> getUserDocuments() {}
```

**Internal Classes**
- `MethodSecurityInterceptor`: AOP interceptor for method calls
- `PrePostAnnotationSecurityMetadataSource`: Parses annotations
- `ExpressionBasedPreInvocationAdvice`: Evaluates SpEL expressions

### Custom AuthorizationManager

**Implementation**
```java
public class BusinessRuleAuthorizationManager implements AuthorizationManager<RequestAuthorizationContext> {
    
    @Override
    public AuthorizationDecision check(Supplier<Authentication> authentication, 
                                     RequestAuthorizationContext context) {
        String userId = context.getRequest().getParameter("userId");
        String currentUser = authentication.get().getName();
        
        // Custom business logic
        boolean canAccess = businessService.canUserAccessResource(currentUser, userId);
        return new AuthorizationDecision(canAccess);
    }
}
```

**Integration**
```java
/**
 * HttpSecurity integrates custom AuthorizationManager.
 * AuthorizeHttpRequestsConfigurer.AuthorizationManagerRequestMatcherRegistry
 * handles request matcher to authorization manager mapping.
 * RequestAuthorizationContext provides HTTP request details to manager.
 */
HttpSecurity http = // injected parameter
http.authorizeHttpRequests(requests -> requests
    .requestMatchers("/api/users/**")
    .access(new BusinessRuleAuthorizationManager())
);
```

**Extension Points**
- `AuthorizationManager<T>`: Core interface for custom authorization
- `RequestAuthorizationContext`: Provides HTTP request details
- `AuthorizationDecision`: Encapsulates access decision with optional reason

## Authorization Flow

1. **Request Received**: User attempts to access a URL
2. **Authentication Check**: Verify user is logged in
3. **Role Evaluation**: Check if user has required role for the endpoint
4. **Access Decision**: 
   - `/delete/**` → Requires ADMIN role
   - All other URLs → Requires any authenticated user

## Spring Security Internal Classes

### Core Authorization Components

**SecurityFilterChain**
- `DefaultSecurityFilterChain`: Contains ordered list of security filters
- `AuthorizationFilter`: Main filter that handles authorization decisions

**Authentication Storage**
- `SecurityContextHolder`: Thread-local storage for authentication
- `SecurityContext`: Contains `Authentication` object
- `Authentication`: Holds user details and authorities

**Authorization Decision Making**
- `AuthorizationManager<RequestAuthorizationContext>`: Makes access decisions
- `RequestMatcherDelegatingAuthorizationManager`: Delegates based on URL patterns
- `AuthorityAuthorizationManager`: Checks roles/authorities

**User Details**
- `UserDetailsService`: Interface for loading user information
- `UserDetails`: Interface representing user information

### Authorization Process Flow

1. **Filter Chain Execution**
   - `AuthorizationFilter` intercepts requests
   - Extracts `Authentication` from `SecurityContextHolder`

2. **Request Matching**
   - `RequestMatcher` implementations check URL patterns
   - `AntPathRequestMatcher` handles `/delete/**` patterns

3. **Authority Checking**
   - `AuthorityAuthorizationManager.hasRole("ADMIN")` checks authorities
   - Converts role "ADMIN" to authority "ROLE_ADMIN"
   - Compares against user's `GrantedAuthority` collection

4. **Access Decision**
   - `AuthorizationDecision`: GRANTED or DENIED
   - If DENIED, throws `AccessDeniedException`

### Key Classes in Action

```java
// Behind the scenes when .hasRole("ADMIN") is called:
AuthorityAuthorizationManager.hasRole("ADMIN")
  → checks for "ROLE_ADMIN" in user's authorities
  → user has ["ROLE_USER"] 
  → returns DENIED
```

### OAuth2/JWT Authorization

**JWT-Based Authorization**
```java
/**
 * HttpSecurity configures OAuth2 scope-based authorization.
 * AuthorizeHttpRequestsConfigurer handles scope validation.
 * OAuth2 scopes are treated as authorities with SCOPE_ prefix.
 */
HttpSecurity http = // injected parameter
http.authorizeHttpRequests(requests -> requests
    .requestMatchers("/delete/**")
    .hasAuthority("SCOPE_admin")
    .anyRequest().hasAuthority("SCOPE_read")
);
```

**Key Integration Points**
- `OAuth2AuthorizedClientService`: Manages OAuth2 tokens
- `ReactiveOAuth2AuthorizedClientManager`: For reactive applications
- `OAuth2AuthenticationToken`: Contains OAuth2 user details and authorities

## Production Considerations

### Performance Trade-offs
- **Method Security**: AOP overhead for annotated methods
- **Custom Managers**: Business logic complexity impacts performance

### Security Best Practices
- **Role Hierarchy**: Use `RoleHierarchy` for role inheritance
- **Method Security**: Combine with URL-based for defense in depth

### Common Pitfalls
- **Role vs Authority**: `hasRole("USER")` expects `ROLE_USER` authority
- **Method Security**: Requires `@EnableMethodSecurity` annotation
- **Custom Managers**: Must handle null authentication gracefully

## Current Configuration Issue

**Problem**: User has "USER" role but `/delete/**` requires "ADMIN" role.

**Result**: Authenticated user cannot access delete endpoints.

**Solutions**:
- Add ADMIN user: `.withUser("admin").password(...).roles("ADMIN")`
- Change rule to: `.requestMatchers("/delete/**").hasRole("USER")`
- Give user both roles: `.roles("USER", "ADMIN")`