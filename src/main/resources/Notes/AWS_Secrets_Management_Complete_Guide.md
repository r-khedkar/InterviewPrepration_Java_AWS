# AWS Secrets Management - Complete Guide 2025

## Quick Decision Matrix

| Use Case | Secrets Manager | Parameter Store |
|----------|-----------------|-----------------|
| **Database passwords** | ✅ Yes (auto-rotation) | ✅ Yes (manual) |
| **API Keys/Tokens** | ✅ Yes (recommended) | ✅ Yes (cost-effective) |
| **Auto-rotation needed** | ✅ YES | ❌ NO |
| **Cross-account access** | ✅ YES (built-in) | ❌ NO (needs custom) |
| **Cost-sensitive** | ❌ High cost | ✅ Low/Free |
| **Compliance needs** | ✅ YES (audit trail) | ⚠️ Limited |
| **Simple config values** | ❌ Overkill | ✅ Perfect |
| **Hierarchical config** | ❌ Flat structure | ✅ YES |

---

## **PART 1: AWS Secrets Manager**

### What is Secrets Manager?

<cite index="7-1">AWS Secrets Manager specializes in protecting access to applications, services, and IT resources by managing the lifecycle of secrets, helping you rotate, manage, and retrieve credentials for databases, API keys, OAuth tokens, JSON Web Tokens (JWTs) and other secrets</cite>.

### Key Features

1. **Automatic Rotation** - <cite index="8-1">Provides full key rotation integration with AWS services like RDS, Redshift, and DocumentDB, with ability to write custom key rotation logic using AWS Lambda functions</cite>
2. **Encryption by Default** - <cite index="7-1">Secrets Manager integrates with AWS Key Management Service (AWS KMS) to encrypt secrets with a KMS key you own and control—this encryption of secrets cannot be disabled</cite>
3. **Cross-Region Replication** - Replicate secrets to multiple regions for DR
4. **Cross-Account Access** - Share secrets with other AWS accounts
5. **Audit Logging** - Full CloudTrail integration

### Cost

```
Price: $0.40 per secret per month
       + $0.05 per 10,000 API calls
       
Example for 50 secrets:
  $0.40 × 50 = $20/month (storage)
  + API calls based on usage
  
Total: ~$20-50/month for most applications
```

### How It Works

```
1. Store Secret
   ↓
2. Secret encrypted with KMS
   ↓
3. Application requests secret
   ↓
4. Secrets Manager validates IAM role
   ↓
5. Returns decrypted value
   ↓
6. Optional: Trigger automatic rotation
```

---

### Creating a Secret

#### Via AWS Console

```
1. Go to Secrets Manager
2. Click "Store a new secret"
3. Choose secret type:
   - Database credentials
   - API key/other
4. Enter secret name: prod/rds/mysql-password
5. Enter secret value
6. Click "Next"
7. Choose rotation (optional)
8. Click "Store"
```

#### Via AWS CLI

```bash
# Simple secret
aws secretsmanager create-secret \
  --name prod/database/password \
  --secret-string "mypassword123"

# JSON secret
aws secretsmanager create-secret \
  --name prod/api/credentials \
  --secret-string '{
    "username": "admin",
    "password": "secret123",
    "api_key": "key123"
  }'

# Retrieve secret
aws secretsmanager get-secret-value \
  --secret-id prod/database/password

# Output:
# {
#   "ARN": "arn:aws:secretsmanager:...",
#   "Name": "prod/database/password",
#   "VersionId": "abc123",
#   "SecretString": "mypassword123"
# }
```

#### Via Terraform

```hcl
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "prod/database/password"
  recovery_window_in_days = 7  # Allow recovery for 7 days
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id      = aws_secretsmanager_secret.db_password.id
  secret_string  = "mypassword123"
}

# Add rotation
resource "aws_secretsmanager_secret_rotation" "db_password" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_enabled    = true
  rotation_lambda_arn = aws_lambda_function.rotation.arn

  rotation_rules {
    automatically_after_days = 30  # Rotate every 30 days
  }
}
```

---

### Retrieving Secrets in Java/Spring Boot

#### Method 1: Using AWS SDK Directly

```java
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.*;
import com.fasterxml.jackson.databind.ObjectMapper;

@Component
public class SecretManager {
    
    private final SecretsManagerClient client;
    private final ObjectMapper objectMapper;
    
    public SecretManager() {
        this.client = SecretsManagerClient.builder().build();
        this.objectMapper = new ObjectMapper();
    }
    
    public String getSecret(String secretName) {
        try {
            GetSecretValueRequest request = GetSecretValueRequest.builder()
                .secretId(secretName)
                .build();
            
            GetSecretValueResponse response = client.getSecretValue(request);
            return response.secretString();
            
        } catch (ResourceNotFoundException e) {
            System.err.println("Secret not found: " + secretName);
            return null;
        } finally {
            client.close();
        }
    }
    
    // Get JSON secret
    public DatabaseCredentials getDatabaseCredentials(String secretName) {
        try {
            String secret = getSecret(secretName);
            return objectMapper.readValue(secret, DatabaseCredentials.class);
        } catch (Exception e) {
            throw new RuntimeException("Failed to parse secret", e);
        }
    }
}

class DatabaseCredentials {
    public String username;
    public String password;
    public String host;
    public String port;
}
```

#### Method 2: Spring Cloud AWS (Recommended)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-secretsmanager</artifactId>
    <version>3.0.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  cloud:
    aws:
      secretsmanager:
        enabled: true
        region: us-east-1
```

```java
@Configuration
public class SecretsConfig {
    
    @Bean
    public SecretsManagerTemplate secretsManagerTemplate(
            SecretsManagerClient secretsManagerClient) {
        return new SecretsManagerTemplate(secretsManagerClient);
    }
}

@Component
public class DatabaseConfig {
    
    @Value("${prod/database/password}")
    private String dbPassword;
    
    @Value("${prod/database/username}")
    private String dbUsername;
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername(dbUsername);
        config.setPassword(dbPassword);
        config.setMaximumPoolSize(20);
        
        return new HikariDataSource(config);
    }
}
```

#### Method 3: Using AWS Lambda Extension (Most Efficient)

```java
// For Lambda or containerized apps
public class SecretsLambdaExtension {
    
    // Lambda extension automatically caches secrets
    // Available at: http://localhost:2773/secretsmanager/get?secretId=...
    
    public static void main(String[] args) throws Exception {
        // Install extension layer: arn:aws:lambda:region:...
        // Secrets are cached for up to 1 hour
        
        String secretJson = callExtension("prod/database/password");
        // Much faster than calling Secrets Manager API directly
    }
}
```

---

### Automatic Secret Rotation

#### For RDS (Built-in)

```bash
# Enable automatic rotation for RDS
aws secretsmanager rotate-secret \
  --secret-id prod/rds/password \
  --rotation-lambda-arn arn:aws:lambda:region:account:function:SecretsManager-... \
  --rotation-rules AutomaticallyAfterDays=30
```

#### For Custom Services (Lambda-based)

```python
# Lambda rotation function (Python)
import boto3
import pymysql
import json

def lambda_handler(event, context):
    service_client = boto3.client('secretsmanager')
    
    # Get secret metadata
    secret_id = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']
    
    metadata = service_client.describe_secret(SecretId=secret_id)
    
    if step == "create":
        # Step 1: Generate new password
        new_password = service_client.get_random_password(PasswordLength=32)
        secret_dict = json.loads(service_client.get_secret_value(SecretId=secret_id)['SecretString'])
        secret_dict['password'] = new_password['RandomPassword']
        
        # Step 2: Set secret version with new password
        service_client.put_secret_value(
            SecretId=secret_id,
            ClientRequestToken=token,
            Secret=json.dumps(secret_dict),
            VersionStages=['AWSPENDING']
        )
    
    elif step == "set":
        # Step 3: Update database password
        secret_value = service_client.get_secret_value(SecretId=secret_id, VersionId=token)
        credentials = json.loads(secret_value['SecretString'])
        
        # Connect to database with old credentials
        old_credentials = json.loads(
            service_client.get_secret_value(SecretId=secret_id)['SecretString']
        )
        
        conn = pymysql.connect(
            host=old_credentials['host'],
            user=old_credentials['username'],
            password=old_credentials['password'],
            database='mysql'
        )
        
        # Update password
        cursor = conn.cursor()
        cursor.execute(
            f"ALTER USER '{credentials['username']}'@'%' IDENTIFIED BY %s",
            (credentials['password'],)
        )
        cursor.close()
        conn.commit()
        conn.close()
    
    elif step == "finish":
        # Step 4: Finalize rotation
        service_client.update_secret_version_stage(
            SecretId=secret_id,
            VersionStage='AWSCURRENT',
            MoveToVersionId=token,
            RemoveFromVersionId=metadata['VersionIdsToStages'][0][0]
        )
    
    return {"statusCode": 200}
```

---

## **PART 2: AWS Systems Manager Parameter Store**

### What is Parameter Store?

<cite index="10-1">Parameter Store enables you to securely store, organize, and retrieve simple configuration data at scale, designed to simplify configuration management across environments, allowing teams to standardize how applications access critical data without hardcoding values or relying on fragmented storage solutions</cite>.

### Key Features

1. **Hierarchical Organization** - `/prod/database/host`, `/prod/database/port`
2. **Three Parameter Types**:
   - **String**: Plain text (free)
   - **StringList**: Comma-separated values
   - **SecureString**: <cite index="12-1">Encrypted with AWS KMS key, can be AWS managed or customer managed for better access control</cite>
3. **Standard & Advanced Tiers**
4. **Free for Standard Tier** (10,000 parameters max)
5. **No Rotation Built-in**

### Pricing

```
Standard Tier (Free):
  - Up to 10,000 parameters
  - No charge

Advanced Tier:
  - $0.04 per parameter per month
  - Up to 1 million parameters
  
API Calls:
  - Free tier: $0
  - Standard: $0 (included)
  
Example:
  100 basic configs = $0/month (Standard Tier)
  100 database secrets = $4/month (Advanced Tier, 100 × $0.04)
```

### Parameter Types & Tiers

```yaml
Standard Tier (Free):
  - Max parameters: 10,000
  - Max size per parameter: 4KB
  - Parameter policies: NO
  - Parameter history: 100 versions

Advanced Tier ($0.04/param/month):
  - Max parameters: 1,000,000
  - Max size per parameter: 8KB
  - Parameter policies: YES (TTL, expiration)
  - Parameter history: unlimited
```

---

### Creating Parameters

#### Via AWS Console

```
1. Systems Manager → Parameter Store
2. Click "Create parameter"
3. Name: /prod/database/host
4. Type: String or SecureString
5. Value: your-db-host.rds.amazonaws.com
6. Click "Create"
```

#### Via AWS CLI

```bash
# Simple string parameter
aws ssm put-parameter \
  --name "/prod/database/host" \
  --value "my-db.rds.amazonaws.com" \
  --type "String"

# Encrypted parameter
aws ssm put-parameter \
  --name "/prod/database/password" \
  --value "mypassword123" \
  --type "SecureString" \
  --key-id "arn:aws:kms:region:account:key/key-id"

# Retrieve parameter
aws ssm get-parameter \
  --name "/prod/database/host"

# Get encrypted value decrypted
aws ssm get-parameter \
  --name "/prod/database/password" \
  --with-decryption

# Get multiple parameters
aws ssm get-parameters \
  --names "/prod/database/host" "/prod/database/port" \
  --with-decryption
```

#### Via Terraform

```hcl
# Simple parameter
resource "aws_ssm_parameter" "db_host" {
  name  = "/prod/database/host"
  type  = "String"
  value = "my-db.rds.amazonaws.com"
  tags = {
    Environment = "production"
  }
}

# Encrypted parameter
resource "aws_ssm_parameter" "db_password" {
  name  = "/prod/database/password"
  type  = "SecureString"
  value = "mypassword123"
  key_id = aws_kms_key.parameter_key.id
}

# Parameter policy (Advanced tier only)
resource "aws_ssm_parameter_with_policy" "db_password_with_ttl" {
  name   = "/prod/database/password"
  type   = "SecureString"
  value  = "mypassword123"
  tier   = "Advanced"
  
  policy = jsonencode({
    Type    = "Expiration"
    Version = "1.0"
    Rules = {
      ParameterMaxAge = {
        MaxAgeInDays = 30  # Parameter expires after 30 days
      }
    }
  })
}
```

---

### Retrieving Parameters in Java/Spring Boot

#### Method 1: Spring Cloud AWS

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-ssm</artifactId>
    <version>3.0.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  cloud:
    aws:
      ssm:
        enabled: true
```

```java
@Component
public class DatabaseConfig {
    
    @Value("${/prod/database/host}")
    private String dbHost;
    
    @Value("${/prod/database/port:3306}")  // With default
    private String dbPort;
    
    @Value("${/prod/database/password}")
    private String dbPassword;
    
    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://" + dbHost + ":" + dbPort + "/mydb");
        config.setUsername("admin");
        config.setPassword(dbPassword);
        return new HikariDataSource(config);
    }
}
```

#### Method 2: Using AWS SDK Directly

```java
import software.amazon.awssdk.services.ssm.SsmClient;
import software.amazon.awssdk.services.ssm.model.GetParameterRequest;
import software.amazon.awssdk.services.ssm.model.GetParameterResponse;

@Component
public class ParameterStore {
    
    private final SsmClient ssmClient;
    private final Map<String, String> cache = new ConcurrentHashMap<>();
    
    @Autowired
    public ParameterStore(SsmClient ssmClient) {
        this.ssmClient = ssmClient;
    }
    
    public String getParameter(String paramName) {
        // Check cache first
        if (cache.containsKey(paramName)) {
            return cache.get(paramName);
        }
        
        try {
            GetParameterRequest request = GetParameterRequest.builder()
                .name(paramName)
                .withDecryption(true)
                .build();
            
            GetParameterResponse response = ssmClient.getParameter(request);
            String value = response.parameter().value();
            
            // Cache for 5 minutes
            cache.put(paramName, value);
            
            return value;
        } catch (Exception e) {
            throw new RuntimeException("Failed to get parameter: " + paramName, e);
        }
    }
}
```

---

## **PART 3: Secrets Manager vs Parameter Store Comparison**

| Feature | Secrets Manager | Parameter Store |
|---------|-----------------|-----------------|
| **Automatic Rotation** | ✅ Yes | ❌ No |
| **Cost** | $0.40/secret/month + API calls | Free (Standard) / $0.04/param (Advanced) |
| **Encryption** | ✅ Always (KMS) | ✅ SecureString only |
| **Cross-Account** | ✅ Yes (resource policy) | ❌ No |
| **Cross-Region Replication** | ✅ Yes | ❌ No |
| **Hierarchical Structure** | ❌ Flat | ✅ Yes (/prod/db/host) |
| **Max Secrets** | 500,000 | 1,000,000 (Advanced) |
| **Parameter Size** | - | 4KB (Standard) / 8KB (Advanced) |
| **Audit Trail** | ✅ CloudTrail | ✅ CloudTrail |
| **Best For** | Critical credentials, rotation | Config values, non-rotating secrets |

---

## **PART 4: Security Best Practices**

### 1. Use Least Privilege IAM

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:region:account:secret:prod/database/*"
      ]
    },
    {
      "Effect": "Deny",
      "Action": [
        "secretsmanager:DeleteSecret",
        "secretsmanager:PutSecretValue"
      ],
      "Resource": "*"
    }
  ]
}
```

### 2. Encrypt with Customer-Managed Keys

```bash
# Create customer-managed KMS key
aws kms create-key --description "Secrets encryption key"

# Use with Secrets Manager
aws secretsmanager create-secret \
  --name prod/password \
  --secret-string "value" \
  --kms-key-id arn:aws:kms:region:account:key/key-id
```

### 3. Enable CloudTrail Logging

```bash
# Automatically logs all Secrets Manager and Parameter Store access
aws cloudtrail start-logging --trail-name my-trail

# Query logs
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceType,AttributeValue=AWS::SecretsManager::Secret
```

### 4. Implement Secret Rotation

```bash
# Set rotation schedule
aws secretsmanager rotate-secret \
  --secret-id prod/database/password \
  --rotation-lambda-arn arn:aws:lambda:...:function:rotate \
  --rotation-rules AutomaticallyAfterDays=30
```

### 5. Use Caching in Application

```java
@Component
public class SecretCache {
    
    private final SecretsManagerClient secretsClient;
    private final Cache<String, String> cache;
    
    public SecretCache() {
        this.secretsClient = SecretsManagerClient.builder().build();
        this.cache = CacheBuilder.newBuilder()
            .expireAfterWrite(1, TimeUnit.HOURS)
            .build();
    }
    
    public String getSecret(String secretName) {
        try {
            // Return from cache if available
            String cached = cache.getIfPresent(secretName);
            if (cached != null) {
                return cached;
            }
            
            // Fetch from Secrets Manager if not cached
            GetSecretValueResponse response = secretsClient.getSecretValue(
                GetSecretValueRequest.builder()
                    .secretId(secretName)
                    .build()
            );
            
            String secret = response.secretString();
            cache.put(secretName, secret);
            return secret;
            
        } catch (Exception e) {
            throw new RuntimeException("Failed to get secret", e);
        }
    }
}
```

### 6. Never Commit Secrets

```bash
# .gitignore
.env
*.key
secrets.json
application-prod.yml

# Use git-secrets hook
brew install git-secrets
git secrets --install
git secrets --register-aws
```

### 7. Use VPC Endpoints

```hcl
# Terraform: Access Secrets Manager privately
resource "aws_vpc_endpoint" "secrets_manager" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  
  private_dns_enabled = true
  subnet_ids          = [aws_subnet.private.id]
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
}
```

### 8. Enable MFA Delete (For Parameter Store)

```bash
# Requires MFA to delete sensitive parameters
aws ssm delete-parameter \
  --name "/prod/database/password" \
  --region us-east-1 \
  --profile mfa-required
```

---

## **PART 5: Common Implementation Patterns**

### Pattern 1: Database Credentials

```java
@Configuration
public class DatabaseConfiguration {
    
    @Bean
    public DataSource dataSource(
            @Value("${database.secret-name}") String secretName,
            SecretsManagerClient secretsClient) {
        
        // Get credentials
        GetSecretValueRequest request = GetSecretValueRequest.builder()
            .secretId(secretName)
            .build();
        
        GetSecretValueResponse response = secretsClient.getSecretValue(request);
        DatabaseCredentials creds = parseCredentials(response.secretString());
        
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://" + creds.getHost() + "/" + creds.getDatabase());
        config.setUsername(creds.getUsername());
        config.setPassword(creds.getPassword());
        config.setMaximumPoolSize(20);
        
        return new HikariDataSource(config);
    }
}
```

### Pattern 2: API Key Management

```java
@Component
public class ExternalAPIClient {
    
    @Value("${api.secret-name}")
    private String apiSecretName;
    
    private final SecretsManagerClient secretsClient;
    private String cachedApiKey;
    private long cachedTime;
    
    // Refresh cache every 1 hour
    private static final long CACHE_DURATION = 3600000;
    
    public String callExternalAPI(String endpoint) {
        String apiKey = getApiKey();
        
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(endpoint))
            .header("Authorization", "Bearer " + apiKey)
            .build();
        
        return client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
            .thenApply(HttpResponse::body)
            .join();
    }
    
    private synchronized String getApiKey() {
        // Check cache
        if (cachedApiKey != null && (System.currentTimeMillis() - cachedTime) < CACHE_DURATION) {
            return cachedApiKey;
        }
        
        // Fetch fresh key
        GetSecretValueResponse response = secretsClient.getSecretValue(
            GetSecretValueRequest.builder()
                .secretId(apiSecretName)
                .build()
        );
        
        cachedApiKey = response.secretString();
        cachedTime = System.currentTimeMillis();
        
        return cachedApiKey;
    }
}
```

### Pattern 3: Multi-Database Setup

```yaml
# application.yml
spring:
  cloud:
    aws:
      region: us-east-1

databases:
  primary:
    secret-name: prod/primary-db/password
    url: jdbc:mysql://primary-db.rds.amazonaws.com/mydb
    
  replica:
    secret-name: prod/replica-db/password
    url: jdbc:mysql://replica-db.rds.amazonaws.com/mydb
```

```java
@Configuration
public class MultiDatabaseConfig {
    
    @Bean("primaryDataSource")
    public DataSource primaryDataSource(
            @Value("${databases.primary.secret-name}") String secretName,
            SecretsManagerClient secretsClient) {
        return createDataSource(secretName, "${databases.primary.url}", secretsClient);
    }
    
    @Bean("replicaDataSource")
    public DataSource replicaDataSource(
            @Value("${databases.replica.secret-name}") String secretName,
            SecretsManagerClient secretsClient) {
        return createDataSource(secretName, "${databases.replica.url}", secretsClient);
    }
    
    private DataSource createDataSource(String secretName, String url, SecretsManagerClient client) {
        String password = client.getSecretValue(
            GetSecretValueRequest.builder()
                .secretId(secretName)
                .build()
        ).secretString();
        
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(url);
        config.setPassword(password);
        config.setMaximumPoolSize(10);
        
        return new HikariDataSource(config);
    }
}
```

---

## **PART 6: Troubleshooting**

### Issue: "AccessDeniedException"

```
Error: User is not authorized to perform: secretsmanager:GetSecretValue

Fix: Check IAM policy attached to the role
aws iam get-role-policy --role-name my-role --policy-name secrets-access

// Or check Secrets Manager resource policy
aws secretsmanager get-resource-policy --secret-id my-secret
```

### Issue: "ResourceNotFoundException"

```
Error: Secrets Manager can't find the specified secret

Fix:
1. Check secret name spelling
2. Verify secret exists in the region
aws secretsmanager list-secrets --region us-east-1

3. If using cross-account, check resource policy
aws secretsmanager get-resource-policy --secret-id arn:aws:secretsmanager:...
```

### Issue: "Timeout retrieving secret"

```
Solution 1: Use Lambda extension (built-in caching)
Solution 2: Implement application-level cache
Solution 3: Use VPC endpoint for faster access

// Verify endpoint connectivity
curl https://secretsmanager.us-east-1.amazonaws.com/
```

### Issue: "Rotation failed"

```
Check rotation Lambda logs:
aws logs tail /aws/lambda/SecretsManager-rotation --follow

Common causes:
1. Database connectivity from Lambda
2. Lambda doesn't have IAM permissions
3. Database user doesn't have ALTER USER permission
```

---

## **PART 7: Cost Optimization**

### Strategy 1: Use Free Standard Tier

```bash
# For non-critical configs, use Parameter Store Standard (free)
aws ssm put-parameter \
  --name "/dev/api/endpoint" \
  --value "https://api.dev.example.com" \
  --type "String"  # Free
```

### Strategy 2: Consolidate Secrets

```bash
# Instead of 50 individual secrets:
❌ prod/db/username → 1 secret
❌ prod/db/password → 1 secret
❌ prod/db/host → 1 secret
Total: $0.40 × 50 = $20/month

# Consolidate into single JSON secret:
✅ prod/database → 1 secret ($0.40/month)
  {
    "username": "...",
    "password": "...",
    "host": "..."
  }
```

### Strategy 3: Reduce API Calls

```java
// BAD: Multiple API calls per request
@GetMapping("/user/{id}")
public User getUser(@PathVariable Long id) {
    String dbPassword = secretsClient.getSecret("db-password"); // API call each time!
    String apiKey = secretsClient.getSecret("api-key");         // API call each time!
    return userRepository.findById(id);
}

// GOOD: Cache secrets
@Component
public class CachedSecretManager {
    private Map<String, CachedSecret> cache = new ConcurrentHashMap<>();
    
    public String getSecret(String name) {
        CachedSecret cached = cache.computeIfAbsent(name, k -> {
            String value = secretsClient.getSecret(k);
            return new CachedSecret(value, System.currentTimeMillis());
        });
        
        // Refresh every hour
        if (System.currentTimeMillis() - cached.timestamp > 3600000) {
            cache.remove(name);
            return getSecret(name);
        }
        
        return cached.value;
    }
}
```

### Estimated Monthly Costs

```
Scenario 1: 10 rotating database credentials
  Secrets Manager: 10 × $0.40 = $4
  API calls (100/day × 30): $0.05 × 3 = $0.15
  Rotation (monthly): ~$0.10
  Total: ~$5/month

Scenario 2: 100 application configs (mostly non-rotating)
  Parameter Store Standard Tier (free): $0
  Secrets Manager for 5 critical secrets: $2
  Total: ~$2/month

Scenario 3: Enterprise (1000 secrets, high rotation)
  Secrets Manager: 1000 × $0.40 = $400
  API calls (10,000/day × 30): $0.05 × 30 = $1.50
  Rotations (monthly for 500): ~$50
  Total: ~$450/month
```

---

## **PART 8: Security Checklist**

- [ ] Enable IAM permissions (least privilege)
- [ ] Use customer-managed KMS keys (not AWS managed)
- [ ] Enable CloudTrail logging
- [ ] Set up secret rotation (every 30-90 days)
- [ ] Use VPC endpoints for private access
- [ ] Implement application-level caching
- [ ] Never commit secrets to Git
- [ ] Use secrets in environment variables, not config files
- [ ] Audit who accessed secrets (CloudTrail)
- [ ] Monitor failed access attempts
- [ ] Enable MFA delete for critical parameters
- [ ] Use resource policies for cross-account access
- [ ] Test secret rotation in staging first
- [ ] Document secret naming convention
- [ ] Regular security reviews of secret access

---

## **SUMMARY: Which Service to Use?**

Use **Secrets Manager** for:
- ✅ Database passwords (need rotation)
- ✅ API keys / OAuth tokens
- ✅ Third-party integrations requiring rotation
- ✅ Compliance requirements (PCI, HIPAA)
- ✅ Cross-account/cross-region scenarios
- ✅ Enterprise environments with audit needs

Use **Parameter Store** for:
- ✅ Application configuration (non-sensitive)
- ✅ Feature flags
- ✅ Database connection strings (read-only)
- ✅ Service endpoints and IDs
- ✅ Cost-sensitive startups
- ✅ Simple hierarchical config storage

Use **Both** (Recommended for most organizations):
- Secrets Manager for credentials
- Parameter Store for config
- Cache both in application
- Monitor with CloudTrail
