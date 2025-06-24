# Connecting Supabase PostgreSQL to n8n on a Self-Hosted Instance

This guide explains how to connect a Supabase PostgreSQL database to a self-hosted n8n instance by configuring firewall rules and using the correct connection string.

## Step 1: Configure Firewall Rules in Your VPC Network

To allow your n8n instance to communicate with Supabase, add a firewall rule in your VPC network settings (e.g., Google Cloud, AWS, or other cloud provider).

1. Go to your **VPC Network** settings.
![image](https://github.com/user-attachments/assets/84a9dfc3-3c7a-4e27-a96b-31b267eb4f0d)
![image](https://github.com/user-attachments/assets/601b7454-b291-43d1-8a8a-d5e8a7a845e6)

3. Add a new firewall rule with the following configuration:
   - **Name**: `allow-postgres-outbound`
   - **Direction**: Outbound
   - **Destination Filter**: IPv4 ranges
   - **Destination IPv4 Ranges**: `0.0.0.0/0` (allows all destinations; restrict to Supabase IPs for better security)
   - **Source Filter**: 
     - Select `IPv4 ranges` and specify the n8n VM's IP range, or
     - Select `None` if the connection can originate from any VM
   - **Protocols and Ports**:
     - Protocol: `TCP`
     - Port: `5432` (default PostgreSQL port)
![image](https://github.com/user-attachments/assets/7b1539e1-5697-4659-8f9d-63c65b914b4b)

4. Save the firewall rule.

## Step 2: Get the Supabase Connection String (Supabase Cloud Account)

1. Log in to your **Supabase Dashboard**.
2. Navigate to your project and click the **Connect** button in the header.
![image](https://github.com/user-attachments/assets/41da7827-bc21-4787-88d8-31749debff5d)

3. Copy the PostgreSQL connection string, which looks like:
   ```
   postgresql://postgres.uniqueaddress:[YOUR-PASSWORD]@aws-0-eu-central-1.pooler.supabase.com:6543/postgres
   ```
You can use either the Transaction Pooler or the Session Pooler but I would recommend the Transaction Pooler
   - Replace `[YOUR-PASSWORD]` with the actual password for your Supabase account (omit the square brackets) and leave the rest unchanged
   - Note: The port may vary (e.g., `6543` or `5432`). Use the port provided in the connection string.

## Step 3: Configure n8n to Connect to Supabase

1. In your self-hosted n8n instance, create a new workflow or edit an existing one.
2. Add a **PostgreSQL node** to your workflow.
3. Configure the node with the following details:
   - **Host**: `aws-0-eu-central-1.pooler.supabase.com` (from the connection string)
   - **Port**: `6543` (or the port specified in the connection string)
   - **Database**: `postgres`
   - **User**: `postgres.fkelse...` (from the connection string - your unique identifier)
   - **Password**: Your Supabase account password
   - **SSL**: Enable if required (Supabase typically requires SSL)

4. Test the connection in n8n to ensure it works.

---

## Connecting n8n to Self-Hosted Supabase via API

If you're running both n8n and Supabase on your own infrastructure (self-hosted), you can use the n8n Supabase node to connect via the Supabase API instead of direct PostgreSQL access. However, this requires additional configuration due to authentication header conflicts.

### Prerequisites for Self-Hosted Setup

1. **Required Firewall Ports**: Ensure these ports are open in your cloud provider's firewall rules:
   - `80` (HTTP)
   - `443` (HTTPS) 
   - `8000` (Supabase Kong API Gateway)
   - `3000` (Supabase Studio - optional)
   - `5432` (PostgreSQL - optional for direct DB access)

### Step 1: Configure Supabase Environment Variables

Edit your Supabase `.env` file to use your external domain:

```bash
cd ~/supabase/docker
nano .env
```

Ensure these variables are set correctly:
```env
API_EXTERNAL_URL=https://your-subdomain.your-domain.com
SUPABASE_PUBLIC_URL=https://your-subdomain.your-domain.com
```

**Important**: Do NOT use `localhost:8000` for `SUPABASE_PUBLIC_URL` as this will prevent external access.

### Step 2: Fix Kong Authorization Header Conflict

n8n's Supabase node sends both `apikey` and `Authorization` headers simultaneously. Kong/Supabase prioritizes the `Authorization` header and ignores the `apikey` header, causing authentication failures.

**Solution**: Configure Kong to remove the `Authorization` header before processing the request.

1. **Edit Kong configuration**:
```bash
nano ~/supabase/docker/volumes/api/kong.yml
```

2. **Find the `rest-v1` service** (around line 93) and add the `request-transformer` plugin:

```yaml
  ## Secure REST routes
  - name: rest-v1
    _comment: 'PostgREST: /rest/v1/* -> http://rest:3000/*'
    url: http://rest:3000/
    routes:
      - name: rest-v1-all
        strip_path: true
        paths:
          - /rest/v1/
    plugins:
      - name: cors
      - name: key-auth
        config:
          hide_credentials: true
      - name: request-transformer    # ADD THIS PLUGIN
        config:
          remove:
            headers:
              - Authorization        # REMOVE Authorization HEADER
      - name: acl
        config:
          hide_groups_header: true
          allow:
            - admin
            - anon
```

3. **Ensure the `request-transformer` plugin is enabled** in `docker-compose.yml`:
```yaml
KONG_PLUGINS: request-transformer,cors,key-auth,acl,basic-auth
```

4. **Restart Kong**:
```bash
sudo docker-compose restart kong
```

### Step 3: Configure n8n Supabase Credentials

1. **Get your Service Role Key**:
   
   **Method 1 - From .env file**:
   ```bash
   cd ~/supabase/docker
   cat .env | grep SERVICE_ROLE_KEY
   ```
   
   **Method 2 - From Supabase Studio**:
   - Go to `https://your-subdomain.your-domain.com`
   - Navigate to **API Docs** → **Authentication** 
   - Copy the **SERVICE KEY** from the Service Keys section
![Screenshot 2025-06-24 112631](https://github.com/user-attachments/assets/799dc438-a0f8-42b5-84cc-24155c52d3fe)

2. In n8n, create new Supabase credentials with:
   - **Host**: `your-subdomain.your-domain.com` (without https:// prefix or /rest/v1/ suffix)
   - **Service Role Secret**: The SERVICE_ROLE_KEY you copied above

3. Test the connection - it should now show "Connection tested successfully".

### Why This Configuration is Necessary

- **n8n Behavior**: The n8n Supabase node automatically sends both authentication headers
- **Kong Priority**: Kong processes the `Authorization` header first and ignores the `apikey`
- **Header Removal**: The `request-transformer` plugin removes the problematic `Authorization` header
- **Result**: Kong only sees the `apikey` header and authentication succeeds

### API Keys and Security

Your self-hosted Supabase instance provides two types of API keys:

- **Client API Key (anon)**: For frontend applications with limited permissions controlled by Row Level Security (RLS)
- **Service Role Key**: For backend services like n8n with full administrative access

Both keys are private to your installation and not publicly accessible.

### Row Level Security (RLS)

By default, Supabase tables created through the Table Editor have RLS enabled. This means:
- The `anon` key can only access data according to your RLS policies
- The `service_role` key bypasses RLS and has full access (used by n8n)

To create policies for controlled access with the `anon` key, use the SQL Editor in Supabase Studio.

---

## Setting Up Vector Database for AI/LLM Applications

For AI chatbots and LLM applications requiring semantic search capabilities, you can set up a vector database using pgvector extension.

### Step 1: Enable pgvector Extension

In your Supabase SQL Editor, run:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### Step 2: Create Vector Table and Search Function



```sql
-- Create knowledge base table with vector embeddings
-- IMPORTANT: Check your embedding model documentation for correct dimensions:
-- - OpenAI text-embedding-ada-002: 1536 dimensions
-- - OpenAI text-embedding-3-small: 1536 dimensions  
-- - OpenAI text-embedding-3-large: 3072 dimensions
-- - Google text-embedding-004: 768 dimensions
-- - Cohere embed-english-v3.0: 1024 dimensions
-- - Anthropic/Voyage AI: varies (check docs)

CREATE TABLE stardawnpublicdata ( -- change to whatever you like
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(768)
);

CREATE FUNCTION query_vectors (
  query_embedding vector(768),
  filter jsonb DEFAULT '{}',
  match_count int DEFAULT null
) RETURNS TABLE (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    stardawnpublicdata.id,        -- MUST match your table name
    stardawnpublicdata.content,   -- MUST match your table name  
    stardawnpublicdata.metadata,  -- MUST match your table name
    1 - (stardawnpublicdata.embedding <=> query_embedding) as similarity
  FROM stardawnpublicdata         -- MUST match your table name
  WHERE stardawnpublicdata.metadata @> filter
  ORDER BY stardawnpublicdata.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

### Why SELECT with Table Names?

**The Problem**: PostgreSQL doesn't know which table to query from if you just write `SELECT id, content` without specifying the table name in the SELECT clause.

**The Solution**: Always prefix column names with the table name: `stardawnpublicdata.id`

---

## ⚠️ Common Error: Supabase Template Copy-Paste Mistake

If you copy the template from [Supabase docs](https://supabase.com/docs/guides/ai/langchain?database-method=sql):

```sql
-- Original template uses "documents" table
BEGIN
  RETURN QUERY
  SELECT 
    id, content, metadata,
    1 - (documents.embedding <=> query_embedding) as similarity
  FROM documents
  WHERE metadata @> filter
  ORDER BY documents.embedding <=> query_embedding
  LIMIT match_count;
END;
```

**But change your table name to something else** (e.g., `stardawnpublicdata`), you get this error:
```
PGRST202 Could not find the function public.match_documents
```

**Why?** The SELECT statement still references `documents.embedding` but your table is named `stardawnpublicdata`.

**Fix:** Update ALL table references in the SELECT statement:
```sql
SELECT 
  stardawnpublicdata.id, 
  stardawnpublicdata.content, 
  stardawnpublicdata.metadata,
  1 - (stardawnpublicdata.embedding <=> query_embedding) as similarity
FROM stardawnpublicdata  -- Your actual table name
```

**Rule**: When changing table names from templates, update EVERY reference in the function body.

### Step 3: Configure n8n Vector Store

1. **Add Supabase Vector Store node** to your n8n workflow
2. **Configure the node**:
   - **Credential**: Select your Supabase credentials
   - **Operation Mode**: "Retrieve Documents (As Tool for AI Agent)"
   - **Table Name**: `stardawnpublicdata`
   - **Function**: Select `query_vectors` from dropdown
   - **Limit**: Set desired number of results (e.g., 4)
   - **Include Metadata**: Enable if needed

### Step 4: In Use





### Multiple Vector Databases

To support multiple use cases, create additional tables and functions:

```sql
-- Customer support knowledge base
CREATE TABLE customer_support_vectors (
  id bigserial primary key,
  content text,
  metadata jsonb,
  embedding vector(768)
);

CREATE FUNCTION search_support_vectors (
  query_embedding vector(768),
  filter jsonb DEFAULT '{}',
  match_count int DEFAULT null
) RETURNS TABLE (
  id bigint,
  content text,
  metadata jsonb,
  similarity float
) ...
```

Then configure separate Supabase Vector Store nodes for each use case.

## Security Tips (Optional)

- **Restrict Firewall Rules**: Instead of allowing `0.0.0.0/0`, use Supabase's specific IP ranges for better security. Check Supabase's documentation for their IP addresses.
- **Use Environment Variables**: Store your Supabase password in n8n's environment variables instead of hardcoding it.
- **Enable SSL**: Ensure SSL is enabled in the n8n PostgreSQL node for secure communication.
- **Close Direct Database Access**: Remove port 5432 from firewall rules if you only need API access
- **Configure RLS Policies**: Set up Row Level Security policies to control data access with the anon key

## Troubleshooting

- **Connection Timeout**: Verify the firewall rule allows outbound TCP traffic on port `5432` (or the port in your connection string).
- **Invalid Credentials**: Double-check the username, password, and database name in the connection string.
- **SSL Issues**: If SSL is required but not enabled, you may see connection errors. Enable SSL in the n8n node settings.
- **"Unauthorized" with Supabase Node**: Ensure the Kong `request-transformer` plugin is properly configured to remove Authorization headers.
- **"No API key found"**: This is normal when testing API endpoints directly - the Supabase node will automatically include the required headers.
- **Vector Function Not Found**: Ensure the function name in n8n matches exactly the SQL function name (case-sensitive).
- **Embedding Dimension Mismatch**: Verify your vector dimensions match your embedding model (768 for Google embeddings-004).

## 🔌 Connection Methods Summary

Your self-hosted Supabase supports multiple integration approaches:

### Method 1: Supabase API (Recommended)
- **Requirements**: Host (sb.example.com) + SERVICE_ROLE_KEY
- **Security**: Very secure - keys are private, not publicly accessible
- **Access**: Only you (keys generated from your JWT_SECRET)
- **Features**: Full Supabase functionality, Row Level Security, real-time subscriptions, vector search

### Method 2: Direct PostgreSQL
- **Requirements**: Host + Port 5432 + postgres user + POSTGRES_PASSWORD
- **Security**: Less secure - bypasses all Supabase security features
- **Access**: Anyone with IP + port + credentials
- **Features**: Basic CRUD operations only

### Method 3: Vector Database Integration
- **Requirements**: Supabase API + pgvector extension + custom functions
- **Security**: Same as Supabase API method
- **Access**: Controlled via RLS and API keys
- **Features**: Semantic search, AI chatbots, LLM context retrieval

**Recommendation**: Use Supabase API method with vector database for AI applications and close port 5432 in firewall for maximum security.
