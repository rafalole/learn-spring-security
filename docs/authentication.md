# Spring Security Authentication Process

## Overview
Spring Security authentication verifies user identity through credentials and establishes security context for subsequent authorization decisions.

## Authentication Configuration

### In-Memory Authentication
```java
/**
 * AuthenticationManagerBuilder configures authentication providers.
 * InMemoryUserDetailsManagerConfigurer handles in-memory user storage.
 * PasswordEncoder ensures secure password storage.
 */
@Autowired
public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMemoryAuthentication().passwordEncoder(passwordEncoder)
        .withUser("user").password(passwordEncoder.encode("pass"))
        .roles("USER");
}
```

## Authentication Alternatives

### Database-Backed User Stores

**JPA Implementation**
```java
/**
 * Custom UserDetailsService implementation loading users from JPA repository.
 * Requires database query per authentication request.
 * Use when you need complex user entity relationships.
 */
@Bean
public UserDetailsService userDetailsService() {
    return new JpaUserDetailsService(userRepository);
}
```

**JDBC Implementation**
```java
/**
 * Built-in JDBC implementation using standard schema.
 * Requires 'users' and 'authorities' tables.
 * Faster than JPA due to direct SQL queries.
 */
@Bean
public UserDetailsService users(DataSource dataSource) {
    return new JdbcUserDetailsManager(dataSource);
}
```

### OAuth2/JWT Authentication

**Resource Server Configuration**
```java
/**
 * Configures application as OAuth2 resource server.
 * HttpSecurity instance provides fluent API for security configuration.
 * OAuth2ResourceServerConfigurer configures JWT token validation.
 * JwtDecoder validates JWT token structure and signature.
 * BearerTokenAuthenticationFilter extracts tokens from Authorization header.
 */
HttpSecurity http = // injected parameter
http.oauth2ResourceServer(oauth2 -> oauth2
    .jwt(jwt -> jwt.decoder(jwtDecoder()))
);
```

**Custom JWT Authority Mapping**
```java
/**
 * Converts JWT claims to Spring Security authorities.
 * JwtGrantedAuthoritiesConverter extracts roles from JWT claims.
 * Authority prefix and claim name are configurable.
 */
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
    authoritiesConverter.setAuthorityPrefix("ROLE_");
    authoritiesConverter.setAuthoritiesClaimName("roles");
    
    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
    return converter;
}
```

**OAuth2 Client Integration**
```java
/**
 * HttpSecurity configures OAuth2 login with custom authority mapping.
 * OAuth2LoginConfigurer handles OAuth2 authentication flow.
 * UserInfoEndpointConfig configures user info endpoint processing.
 * GrantedAuthoritiesMapper converts OAuth2 authorities to application roles.
 * OAuth2AuthenticationToken contains user details and authorities.
 */
HttpSecurity http = // injected parameter
http.oauth2Login(oauth2 -> oauth2
    .userInfoEndpoint(userInfo -> userInfo
        .userAuthoritiesMapper(authoritiesMapper())
    )
);

@Bean
public GrantedAuthoritiesMapper authoritiesMapper() {
    return authorities -> {
        Set<GrantedAuthority> mappedAuthorities = new HashSet<>();
        authorities.forEach(authority -> {
            if (authority.getAuthority().equals("OIDC_USER")) {
                mappedAuthorities.add(new SimpleGrantedAuthority("ROLE_USER"));
            }
        });
        return mappedAuthorities;
    };
}
```

## Spring Security Internal Classes

### Core Authentication Components

**Authentication Storage**
- `SecurityContextHolder`: Thread-local storage for authentication
- `SecurityContext`: Contains `Authentication` object
- `Authentication`: Holds user details and authorities

**User Details Management**
- `InMemoryUserDetailsManager`: Stores users in memory
- `JdbcUserDetailsManager`: Database-backed user storage
- `UserDetails`: Interface representing user information
- `User`: Default implementation with username, password, authorities

**Authentication Processing**
- `AuthenticationManager`: Coordinates authentication providers
- `ProviderManager`: **Default implementation** managing multiple providers
- `DaoAuthenticationProvider`: Username/password authentication
- `JwtAuthenticationProvider`: JWT token validation

### Authentication Process Flow

1. **Credential Extraction**
   - `UsernamePasswordAuthenticationFilter` extracts login credentials
   - `BearerTokenAuthenticationFilter` extracts JWT tokens
   - Creates `Authentication` object (unauthenticated)

2. **Authentication Manager Processing**
   - `ProviderManager` delegates to appropriate `AuthenticationProvider`
   - `DaoAuthenticationProvider` loads user via `UserDetailsService`
   - Password validation using `PasswordEncoder`

3. **Security Context Establishment**
   - Successful authentication creates authenticated `Authentication` object
   - `SecurityContextHolder` stores authentication for current thread
   - `SecurityContextPersistenceFilter` manages context across requests

4. **Authority Population**
   - `UserDetails.getAuthorities()` provides user roles/permissions
   - Authorities converted to `GrantedAuthority` objects
   - Available for subsequent authorization decisions

### Key Classes in Action

```java
// Behind the scenes during form login:
UsernamePasswordAuthenticationFilter
  → creates UsernamePasswordAuthenticationToken (unauthenticated)
  → ProviderManager.authenticate()
  → DaoAuthenticationProvider.authenticate()
  → UserDetailsService.loadUserByUsername()
  → PasswordEncoder.matches()
  → returns UsernamePasswordAuthenticationToken (authenticated)
  → SecurityContextHolder.setContext()
```

## Production Considerations

### Performance Trade-offs
- **In-Memory**: Fastest, limited scalability
- **JDBC**: Good performance, requires connection pooling
- **JPA**: Flexible but slower, consider caching with `@Cacheable`
- **JWT**: Stateless, requires token validation overhead

### Security Best Practices
- **Password Encoding**: Always use `BCryptPasswordEncoder` in production
- **JWT Security**: Validate issuer, audience, and expiration claims
- **Session Management**: Configure session fixation protection
- **CSRF Protection**: Enable for stateful applications

### Common Pitfalls
- **Password Storage**: Never store plain text passwords
- **JWT Claims**: Authority mapping must match expected format
- **Session Handling**: Understand stateless vs stateful implications
- **Authority Format**: `hasRole("USER")` expects `ROLE_USER` authority