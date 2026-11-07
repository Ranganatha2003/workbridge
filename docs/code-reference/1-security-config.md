# WorkBridge — Security Configuration

## Stack
- Spring Security + JWT + BCrypt
- Roles: `ROLE_WORKER`, `ROLE_CONTRACTOR`, `ROLE_ADMIN`

---

## Components to Create

| Class | Purpose |
|---|---|
| `SecurityConfig` | HTTP security, filter chain, CORS, CSRF |
| `JwtAuthenticationFilter` | Validate JWT on every request |
| `JwtService` | Generate & validate tokens |
| `CustomUserDetailsService` | Load user by email |

---

## SecurityConfig

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http, JwtAuthenticationFilter jwtFilter) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(Customizer.withDefaults())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/services/**", "/services/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

## JwtService

```java
@Service
public class JwtService {
    // Store secret in application.properties: jwt.secret=...
    @Value("${jwt.secret}")
    private String secret;

    // Access token: 24h, Refresh token: 7 days
    private static final long ACCESS_EXPIRY  = 86_400_000L;
    private static final long REFRESH_EXPIRY = 604_800_000L;

    public String generateAccessToken(UserDetails user)  { /* sign with HMAC-SHA256 */ }
    public String generateRefreshToken(UserDetails user) { /* sign with HMAC-SHA256 */ }
    public String extractUsername(String token)          { /* parse subject */ }
    public boolean isTokenValid(String token, UserDetails user) { /* verify sig + expiry */ }
}
```

---

## JwtAuthenticationFilter

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {

        String header = req.getHeader("Authorization");
        if (header == null || !header.startsWith("Bearer ")) { chain.doFilter(req, res); return; }

        String token = header.substring(7);
        String email = jwtService.extractUsername(token);

        if (email != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(email);
            if (jwtService.isTokenValid(token, userDetails)) {
                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                auth.setDetails(new WebAuthenticationDetailsSource().buildDetails(req));
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }
        chain.doFilter(req, res);
    }
}
```

---

## CustomUserDetailsService

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String email) throws UsernameNotFoundException {
        User user = userRepository.findByEmail(email)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + email));
        return org.springframework.security.core.userdetails.User.builder()
            .username(user.getEmail())
            .password(user.getPasswordHash())
            .roles(user.getRole().name().replace("ROLE_", ""))
            .build();
    }
}
```

---

## Role-Based Access Matrix

| Endpoint group | WORKER | CONTRACTOR | ADMIN |
|---|---|---|---|
| `/api/auth/**` | public | public | public |
| `/api/services` (GET) | ✓ | ✓ | ✓ |
| `POST /api/services` | ✗ | ✓ | ✗ |
| `/api/applications/my-applications` | ✓ | ✗ | ✗ |
| `/api/services/{id}/applications` (GET) | ✗ | owner only | ✓ |
| accept / reject application | ✗ | owner only | ✓ |
| cancel application | owner only | ✗ | ✗ |
| `/api/notifications/**` | ✓ | ✓ | ✓ |
| `/api/payments/**` | ✗ | ✓ | ✓ |
| block / unblock contractor | ✗ | ✗ | ✓ |

### Method-level `@PreAuthorize` examples

```java
// Service update / cancel / complete
@PreAuthorize("hasRole('ADMIN') or @serviceSecurity.isServiceOwner(#id, authentication.name)")

// Accept / reject application
@PreAuthorize("hasRole('ADMIN') or @applicationSecurity.isServiceOwnerOfApplication(#id, authentication.name)")

// Cancel application
@PreAuthorize("@applicationSecurity.isApplicationOwner(#id, authentication.name)")

// Process payout
@PreAuthorize("hasRole('ADMIN') or @applicationSecurity.isServiceOwnerOfApplication(#applicationId, authentication.name)")
```

---

## Contractor Blocking Check
After JWT validation, also check contractor account status:

```java
// In JwtAuthenticationFilter or a custom AuthenticationProvider
if (user.getRole() == Role.ROLE_CONTRACTOR) {
    ContractorProfile cp = contractorProfileRepository.findByUserId(user.getId())...;
    if (cp.getStatus() == ContractorStatus.BLOCKED) {
        throw new AccessDeniedException("Contractor account is blocked");
    }
}
```

---

## application.properties keys required

```properties
jwt.secret=<strong-random-secret>
jwt.access.expiration=86400000
jwt.refresh.expiration=604800000
```
