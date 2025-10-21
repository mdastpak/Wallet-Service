# External Auth Provider Integration

## Overview

Integration with external OAuth 2.0 / OIDC providers (Keycloak, Auth0, Okta) for centralized authentication.

---

## Keycloak Integration

### Configuration

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.example.com/realms/wallet-service
          jwk-set-uri: https://keycloak.example.com/realms/wallet-service/protocol/openid-connect/certs
```

### JWT Token Validation

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            .build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(new KeycloakRoleConverter());
        return converter;
    }
}
```

### Extract Roles from Keycloak Token

```java
public class KeycloakRoleConverter implements Converter<Jwt, Collection<GrantedAuthority>> {

    @Override
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        Map<String, Object> realmAccess = jwt.getClaim("realm_access");
        if (realmAccess == null) {
            return Collections.emptyList();
        }

        List<String> roles = (List<String>) realmAccess.get("roles");
        return roles.stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
            .collect(Collectors.toList());
    }
}
```

---

## Auth0 Integration

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://your-tenant.auth0.com/
          audiences: https://wallet-api
```

---

## User Synchronization

```java
@Service
public class UserSyncService {

    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        Jwt jwt = (Jwt) event.getAuthentication().getPrincipal();

        String externalUserId = jwt.getSubject();
        String email = jwt.getClaim("email");
        String name = jwt.getClaim("name");

        // Sync to local database
        User user = userRepo.findByExternalId(externalUserId)
            .orElseGet(() -> {
                User newUser = new User();
                newUser.setExternalId(externalUserId);
                return newUser;
            });

        user.setEmail(email);
        user.setFullName(name);
        userRepo.save(user);
    }
}
```

---

## Token Caching

```java
@Service
public class TokenCacheService {

    private final Cache<String, Claims> tokenCache = Caffeine.newBuilder()
        .expireAfterWrite(15, TimeUnit.MINUTES)
        .maximumSize(10_000)
        .build();

    public Claims getClaims(String token) {
        return tokenCache.get(token, this::parseToken);
    }

    private Claims parseToken(String token) {
        return jwtDecoder.decode(token);
    }
}
```

---

See [Authentication](../03-api-specification/authentication.md) for complete auth flows.
