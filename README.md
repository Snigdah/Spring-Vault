# Spring Boot + HashiCorp Vault Integration for Dynamic Database Credentials

This project demonstrates how to configure **Spring Boot** with **HashiCorp Vault** to dynamically manage **PostgreSQL database credentials** using the **AppRole authentication method**.

---

## 📥 Prerequisites

- Download and install **Vault CLI**: [Vault Download](https://developer.hashicorp.com/vault/downloads)
- PostgreSQL Database running locally on `localhost:5432` (database name: `audit`)

---

## ⚙️ Architecture

```plaintext
+----------------------+          +-----------------------------+
| Spring Boot App      | <------> | Vault (AppRole Authentication)|
+----------------------+          +-----------------------------+
          |                                   |
          | Requests Dynamic DB Credentials   |
          |                                   |
          v                                   v
+---------------------+              +--------------------------+
| Dynamic DB Username | <----------  | PostgreSQL Database      |
| and Password (1h)   |              +--------------------------+
+---------------------+
```

- Vault dynamically creates database users valid for 1 hour.
- Spring Boot uses AppRole to securely retrieve these credentials.

---

## 🚀 Vault Setup Commands (Step by Step)

Follow these commands exactly to set up Vault for dynamic credential management.

### 1. Start Vault Server in Dev Mode

```bash
vault server -dev
```

Starts Vault in development mode.

### 2. Set Vault Environment Variables

```bash
set VAULT_ADDR=http://127.0.0.1:8200
set VAULT_TOKEN=<your-vault-token>  # Example: hvs.svwAf89OoanVW0qg8pMzaOzS
```

Configure Vault CLI to connect to the Vault server.

### 3. Enable AppRole Authentication

```bash
vault auth enable approle
```

Enables AppRole authentication in Vault.

### 4. Enable Database Secrets Engine

```bash
vault secrets enable database
```

Enables Vault's database secrets backend.

### 5. Configure PostgreSQL Connection in Vault

```bash
vault write database/config/postgresql ^
    plugin_name=postgresql-database-plugin ^
    allowed_roles="audit-role" ^
    connection_url="postgresql://{{username}}:{{password}}@localhost:5432/audit" ^
    username="postgres" ^
    password="12345"
```

Configures the PostgreSQL database connection for Vault.

### 6. Create Vault Database Role for Dynamic Credentials

 Grants connection + schema creation, but does not allow table access
```bash
vault write database/roles/audit-role ^
    db_name=postgresql ^
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT ALL PRIVILEGES ON DATABASE audit TO \"{{name}}\";" ^
    default_ttl="1h" ^
    max_ttl="24h"
```
Grants connection + table access
```bash
vault write database/roles/audit-role ^
    db_name=postgresql ^
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT CONNECT ON DATABASE development TO \"{{name}}\"; GRANT USAGE ON SCHEMA public TO \"{{name}}\"; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\"; ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO \"{{name}}\";" ^
    default_ttl="1h" ^
    max_ttl="24h"
```

Grants connection + table access +  revoked the lease.
```bash
vault write database/roles/audit-role ^
db_name=postgresql ^
creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT CONNECT ON DATABASE development TO \"{{name}}\"; GRANT USAGE ON SCHEMA public TO \"{{name}}\"; GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\"; ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO \"{{name}}\";" ^
revocation_statements="REASSIGN OWNED BY \"{{name}}\" TO postgres; DROP OWNED BY \"{{name}}\";" ^
default_ttl="1h" ^
max_ttl="24h"
```

Creates a Vault role to dynamically generate database users with 1-hour validity.

### 7. Create AppRole for the Spring Boot Application

```bash
vault write auth/approle/role/audit-app ^
    token_policies="audit-policy" ^
    token_ttl="1h" ^
    token_max_ttl="24h" ^
    bind_secret_id=true ^
    secret_id_num_uses=10 ^
    secret_id_ttl="10m"
```

Creates an AppRole for Spring Boot to authenticate and fetch credentials.

### 8. Verify Available Vault Policies

```bash
vault policy list
```

Lists all Vault policies.

### 9. Retrieve Role ID for AppRole

```bash
vault read auth/approle/role/audit-app/role-id
```

Retrieves the AppRole's Role ID (needed in Spring Boot config).

### 10. Generate Secret ID for AppRole

```bash
vault write -f auth/approle/role/audit-app/secret-id
```

Generates the Secret ID for AppRole authentication (needed in Spring Boot config).

### 11. Create Vault Policy File

Create a file named `audit-policy.hcl`:

```hcl
path "database/creds/audit-role" {
  capabilities = ["read"]
}

path "auth/token/lookup-self" {
  capabilities = ["read"]
}
```

Vault policy to allow reading dynamic credentials and self token inspection.

### 12. Upload Vault Policy

```bash
vault policy write audit-policy audit-policy.hcl
```

Uploads the Vault policy to the Vault server.

---

## ⚙️ Spring Boot Configuration

### bootstrap.properties

```properties
# Vault Configuration
spring.cloud.vault.enabled=true
spring.cloud.vault.scheme=http
spring.cloud.vault.host=127.0.0.1
spring.cloud.vault.port=8200

# AppRole Authentication Configuration
spring.cloud.vault.authentication=approle
spring.cloud.vault.app-role.role-id=<paste-your-role-id-here>
spring.cloud.vault.app-role.secret-id=<paste-your-secret-id-here>

# Database Secrets Engine Configuration
spring.cloud.vault.database.enabled=true
spring.cloud.vault.database.backend=database
spring.cloud.vault.database.role=audit-role
```

Configure Spring Boot to use Vault with AppRole authentication.

---

## 🔄 Dynamic Credential Rotation

- Vault automatically rotates database users every **1 hour** (configured with `default_ttl`).
- Spring Cloud Vault automatically refreshes the credentials before expiration.

---

## ✅ Summary

- ✅ Vault Server Running
- ✅ Database Secrets Engine Enabled
- ✅ PostgreSQL Connection Configured
- ✅ Vault Role and Policy Created
- ✅ AppRole Configured
- ✅ Spring Boot Connected to Vault

---

## 🔗 Useful Resources

- [Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [Spring Cloud Vault Documentation](https://docs.spring.io/spring-cloud-vault/docs/current/reference/html/)

---

## 📄 License

This project is open source and available under the MIT License.

---

## 🛠️ Troubleshooting

### Common Issues

1. **Vault Server Not Starting**
   - Ensure Vault CLI is properly installed
   - Check if port 8200 is available

2. **Database Connection Errors**
   - Verify PostgreSQL is running on `localhost:5432`
   - Check database credentials in Vault configuration

3. **AppRole Authentication Failures**
   - Ensure Role ID and Secret ID are correctly copied
   - Verify the audit-policy is properly uploaded

### Verification Commands

```bash
# Test Vault connectivity
vault status

# Test database credential generation
vault read database/creds/audit-role

# Test AppRole authentication
vault write auth/approle/login role_id=<your-role-id> secret_id=<your-secret-id>
```
