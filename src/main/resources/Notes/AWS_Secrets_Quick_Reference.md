# AWS Secrets Management - Quick Reference Guide

## Decision Flowchart: Which Service to Use?

```
Need to store secrets/config?
        ↓
   Is it sensitive data?
   (passwords, API keys, tokens)
   ├─ YES → Is rotation needed?
   │        ├─ YES → Use Secrets Manager
   │        └─ NO  → Parameter Store (SecureString)
   │
   └─ NO  → Is it configuration data?
            (URLs, settings, feature flags)
            ├─ YES  → Use Parameter Store (Standard/String)
            └─ NO   → Don't store it
```

---

## Quick Implementation (5 Minutes)

### Step 1: Create Secret (AWS Console)

```
1. Secrets Manager → Store a new secret
2. Name: prod/database/password
3. Value: your-password
4. Next → Disable automatic rotation (for now)
5. Store
```

### Step 2: Add IAM Permission

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:*:*:secret:prod/database/*"
    }
  ]
}
```

### Step 3: Add Spring Boot Dependency

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secretsmanager</artifactId>
    <version>3.0.0</version>
</dependency>
```

### Step 4: Use in Application

```java
@Value("${prod/database/password}")
private String dbPassword;
```

Done! Secret is automatically fetched and injected.

---

## Code Snippets

### Secrets Manager

#### Java - Get Secret
```java
SecretsManagerClient client = SecretsManagerClient.builder().build();
GetSecretValueRequest request = GetSecretValueRequest.builder()
    .secretId("prod/database/password")
    .build();
String secret = client.getSecretValue(request).secretString();
client.close();
```

#### Java - Get JSON Secret
```java
GetSecretValueResponse response = client.getSecretValue(request);
JSONObject json = new JSONObject(response.secretString());
String password = json.getString("password");
String username = json.getString("username");
```

#### CLI - Create Secret
```bash
aws secretsmanager create-secret \
  --name prod/database/password \
  --secret-string "mypassword123" \
  --kms-key-id alias/aws/secretsmanager
```

#### CLI - Get Secret
```bash
aws secretsmanager get-secret-value \
  --secret-id prod/database/password \
  --query SecretString \
  --output text
```

#### Terraform - Create Secret
```hcl
resource "aws_secretsmanager_secret" "db_password" {
  name = "prod/database/password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id      = aws_secretsmanager_secret.db_password.id
  secret_string  = "mypassword123"
}
```

#### Docker - Use Secret in Environment
```bash
# Fetch secret and set as environment variable
SECRET=$(aws secretsmanager get-secret-value \
  --secret-id prod/database/password \
  --query SecretString \
  --output text)

docker run -e DB_PASSWORD="$SECRET" myapp:latest
```

---

### Parameter Store

#### Java - Get Parameter
```java
SsmClient client = SsmClient.builder().build();
GetParameterRequest request = GetParameterRequest.builder()
    .name("/prod/database/host")
    .withDecryption(true)  // For SecureString
    .build();
String value = client.getParameter(request).parameter().value();
client.close();
```

#### Java - Get Multiple Parameters
```java
GetParametersRequest request = GetParametersRequest.builder()
    .names("/prod/database/host", "/prod/database/port", "/prod/database/user")
    .withDecryption(true)
    .build();

GetParametersResponse response = client.getParameters(request);
response.parameters().forEach(param -> 
    System.out.println(param.name() + " = " + param.value())
);
```

#### CLI - Create Parameter
```bash
# String parameter (free)
aws ssm put-parameter \
  --name "/prod/database/host" \
  --value "db.example.com" \
  --type "String"

# Encrypted parameter
aws ssm put-parameter \
  --name "/prod/database/password" \
  --value "mypassword123" \
  --type "SecureString"
```

#### CLI - Get Parameter
```bash
aws ssm get-parameter \
  --name "/prod/database/host"

# Get with decryption
aws ssm get-parameter \
  --name "/prod/database/password" \
  --with-decryption
```

#### Terraform - Create Parameter
```hcl
resource "aws_ssm_parameter" "db_host" {
  name  = "/prod/database/host"
  type  = "String"
  value = "db.example.com"
}

resource "aws_ssm_parameter" "db_password" {
  name  = "/prod/database/password"
  type  = "SecureString"
  value = "mypassword123"
  key_id = aws_kms_key.ssm.id
}
```

#### Spring Boot - Inject Parameters
```yaml
# application.yml
app:
  database:
    host: ${/prod/database/host}
    port: ${/prod/database/port:3306}
    password: ${/prod/database/password}
```

```java
@Configuration
@EnableConfigurationProperties
public class AppConfig {
    
    @Value("${app.database.host}")
    private String dbHost;
    
    @Value("${app.database.port}")
    private Integer dbPort;
}
```

---

## Caching Strategy (Critical for Performance)

### In-Memory Cache
```java
@Component
public class SecretCache {
    private static final long CACHE_TTL = 3600000; // 1 hour
    private final Map<String, CachedValue<?>> cache = new ConcurrentHashMap<>();
    
    public <T> T getOrFetch(String key, Class<T> type, Function<String, String> fetcher) {
        CachedValue<?> cached = cache.get(key);
        
        // Return if valid
        if (cached != null && !cached.isExpired()) {
            return (T) cached.value;
        }
        
        // Fetch fresh value
        String rawValue = fetcher.apply(key);
        T value = parseValue(rawValue, type);
        cache.put(key, new CachedValue<>(value, CACHE_TTL));
        return value;
    }
    
    private static class CachedValue<T> {
        T value;
        long createdAt;
        long ttl;
        
        CachedValue(T value, long ttl) {
            this.value = value;
            this.ttl = ttl;
            this.createdAt = System.currentTimeMillis();
        }
        
        boolean isExpired() {
            return System.currentTimeMillis() - createdAt > ttl;
        }
    }
}
```

### Caffeine Cache (Production)
```java
@Bean
public Cache<String, String> secretCache() {
    return Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(1, TimeUnit.HOURS)
        .build();
}

@Component
public class CachedSecretManager {
    private final Cache<String, String> cache;
    private final SecretsManagerClient client;
    
    public String getSecret(String name) {
        return cache.get(name, k -> fetchSecretFromAWS(k));
    }
    
    private String fetchSecretFromAWS(String name) {
        return client.getSecretValue(
            GetSecretValueRequest.builder()
                .secretId(name)
                .build()
        ).secretString();
    }
}
```

---

## IAM Policies

### Minimal Read-Only Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:*:secret:prod/*"
    }
  ]
}
```

### Parameter Store Read-Only Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:GetParameter",
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
      ],
      "Resource": "arn:aws:ssm:*:*:parameter/prod/*"
    },
    {
      "Effect": "Allow",
      "Action": "kms:Decrypt",
      "Resource": "arn:aws:kms:*:*:key/*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "ssm.*.amazonaws.com"
        }
      }
    }
  ]
}
```

### Admin Policy (Use Carefully)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:*",
      "Resource": "arn:aws:secretsmanager:*:*:secret:prod/*"
    }
  ]
}
```

---

## Cost Comparison

### Scenario: Production E-commerce Site

**Database Credentials + API Keys:**
- 5 database secrets
- 3 API keys
- 2 OAuth tokens

| Service | Monthly Cost | Features |
|---------|--------------|----------|
| **Secrets Manager Only** | $4.00 | Auto-rotation, audit trail |
| **Parameter Store Only** | $0.00 | Manual rotation, basic audit |
| **Hybrid (Recommended)** | $0.80 | Secrets Manager for credentials, Param Store for config |

---

## Monitoring & Alerts

### CloudWatch Alarm - Failed Secret Access
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "secrets-access-failed" \
  --alarm-actions arn:aws:sns:region:account:alerts \
  --metric-name "GetSecretValueFailures" \
  --namespace "AWS/SecretsManager" \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold
```

### CloudTrail Query - Who Accessed Secrets
```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
    AttributeKey=EventName,AttributeValue=GetSecretValue \
  --max-results 50 \
  --region us-east-1
```

---

## Common Mistakes to Avoid

### ❌ WRONG: Storing secrets in .env files
```bash
# .env
DB_PASSWORD=mypassword123  # Never do this!
API_KEY=secret123
```

### ✅ RIGHT: Use AWS Secrets Manager or Parameter Store
```java
@Value("${/prod/database/password}")
private String dbPassword;
```

---

### ❌ WRONG: No caching, API call per request
```java
@GetMapping("/data")
public Data getData() {
    String secret = secretsClient.getSecret("api-key"); // Slow!
    return fetchData(secret);
}
```

### ✅ RIGHT: Cache secrets with TTL
```java
public String getSecret(String name) {
    return cache.get(name, k -> secretsClient.getSecret(k));
}
```

---

### ❌ WRONG: Hardcoded credentials
```java
String password = "admin123";  // Never!
DataSource ds = createDataSource(password);
```

### ✅ RIGHT: External secret management
```java
@Value("${database.secret-name}")
String secretName;

String password = secretsClient.getSecret(secretName);
DataSource ds = createDataSource(password);
```

---

### ❌ WRONG: Logging secrets
```java
logger.info("Connecting with password: " + password);  // Never!
```

### ✅ RIGHT: Log only metadata
```java
logger.info("Connecting to database");  // Good
logger.debug("DB connection established with pool size: " + poolSize);
```

---

## Troubleshooting Checklist

| Problem | Cause | Fix |
|---------|-------|-----|
| **AccessDeniedException** | Missing IAM permission | Add `secretsmanager:GetSecretValue` to role policy |
| **ResourceNotFound** | Secret doesn't exist or wrong name | Check secret name spelling and region |
| **Timeout** | Network connectivity issue | Use VPC endpoint for faster access |
| **Stale values** | No caching | Implement application-level cache |
| **High costs** | Too many secrets | Consolidate into JSON secrets |
| **Failed rotation** | Lambda connectivity | Check Lambda VPC and DB firewall |
| **Slow API calls** | No caching + frequent retrieval | Cache secrets for 1 hour |

---

## Best Practices Checklist

- [ ] Use Secrets Manager for credentials requiring rotation
- [ ] Use Parameter Store for configuration and non-rotating secrets
- [ ] Implement application-level caching (1-hour TTL)
- [ ] Use customer-managed KMS keys (not AWS managed)
- [ ] Enable CloudTrail logging for audit trail
- [ ] Set up IAM least privilege access
- [ ] Never commit secrets to Git
- [ ] Use VPC endpoints for private access
- [ ] Test secret rotation in staging environment
- [ ] Monitor failed access attempts
- [ ] Document your secrets naming convention
- [ ] Review secret access quarterly
- [ ] Rotate critical secrets every 30-90 days
- [ ] Use environment variables, not config files

---

## One-Liner Commands

```bash
# Create secret
aws secretsmanager create-secret --name prod/db/password --secret-string "pass123"

# Get secret
aws secretsmanager get-secret-value --secret-id prod/db/password --query SecretString --output text

# List all secrets
aws secretsmanager list-secrets --region us-east-1

# Delete secret (schedule deletion for 7 days)
aws secretsmanager delete-secret --secret-id prod/db/password --recovery-window-in-days 7

# Create parameter
aws ssm put-parameter --name /prod/api/endpoint --value "https://api.com" --type String

# Get parameter
aws ssm get-parameter --name /prod/api/endpoint --query Parameter.Value --output text

# Get encrypted parameter
aws ssm get-parameter --name /prod/db/password --with-decryption --query Parameter.Value --output text

# Tag secret (for cost tracking)
aws secretsmanager tag-resource --secret-id prod/db/password --tags Key=Environment,Value=Production

# Rotate secret immediately
aws secretsmanager rotate-secret --secret-id prod/db/password --rotate-immediately
```

---

## Next Steps

1. **Audit current secrets** - Find all hardcoded credentials and .env files
2. **Create migration plan** - Move to Secrets Manager/Parameter Store
3. **Set up caching** - Implement application-level cache
4. **Enable monitoring** - Set up CloudTrail and CloudWatch alarms
5. **Test rotation** - Create test environment and test secret rotation
6. **Deploy to production** - Roll out gradually, monitor closely
7. **Decommission old method** - Remove hardcoded secrets and files
8. **Document process** - Create runbooks for common operations

---

**Resources:**
- AWS Secrets Manager: https://docs.aws.amazon.com/secretsmanager/
- Parameter Store: https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html
- Spring Cloud AWS: https://spring.cloud.io/spring-cloud-aws/
- IAM Best Practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
