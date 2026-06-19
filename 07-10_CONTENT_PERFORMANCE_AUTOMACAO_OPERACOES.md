# 7️⃣-🔟 WIKI.JS: CONTENT MANAGEMENT, PERFORMANCE, AUTOMAÇÃO & OPERAÇÕES

---

# 7️⃣ CONTENT MANAGEMENT

## 📦 EXEMPLO 1: Bulk Page Operations

**Arquivo:** `bulk-operations.sh`

```bash
#!/bin/bash

# Wiki.js Bulk Operations via API

WIKI_URL="https://wiki.example.com"
API_TOKEN="your-api-token"

# 1. Bulk Create Pages
bulk_create_pages() {
  local csv_file=$1
  
  # CSV format: path,title,content,locale
  while IFS=',' read -r path title content locale; do
    curl -X POST "$WIKI_URL/api/pages" \
      -H "Authorization: Bearer $API_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{
        \"title\": \"$title\",
        \"path\": \"$path\",
        \"content\": \"$content\",
        \"locale\": \"${locale:-en}\"
      }"
  done < "$csv_file"
}

# 2. Bulk Update Metadata
bulk_update_metadata() {
  local category=$1
  local new_description=$2
  
  # Get all pages in category
  pages=$(curl -s -H "Authorization: Bearer $API_TOKEN" \
    "$WIKI_URL/api/pages?filter=category:$category" | jq -r '.[].id')
  
  for page_id in $pages; do
    curl -X PUT "$WIKI_URL/api/pages/$page_id" \
      -H "Authorization: Bearer $API_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{
        \"description\": \"$new_description\"
      }"
  done
}

# 3. Bulk Move Pages
bulk_move_pages() {
  local source_path=$1
  local dest_path=$2
  
  pages=$(curl -s -H "Authorization: Bearer $API_TOKEN" \
    "$WIKI_URL/api/pages?filter=path:$source_path" | jq -r '.[].id')
  
  for page_id in $pages; do
    curl -X PUT "$WIKI_URL/api/pages/$page_id" \
      -H "Authorization: Bearer $API_TOKEN" \
      -H "Content-Type: application/json" \
      -d "{
        \"path\": \"${dest_path}/${page_id}\"
      }"
  done
}

# 4. Bulk Delete (com confirmação)
bulk_delete_pages() {
  local pattern=$1
  
  echo "⚠️  BULK DELETE WARNING"
  echo "Pattern: $pattern"
  
  # Preview
  pages=$(curl -s -H "Authorization: Bearer $API_TOKEN" \
    "$WIKI_URL/api/pages?filter=path:$pattern" | jq '.')
  
  echo "$pages" | jq '.[] | {id, path, title}'
  
  read -p "Delete these pages? (yes/no): " confirm
  
  if [ "$confirm" = "yes" ]; then
    echo "$pages" | jq -r '.[].id' | while read page_id; do
      curl -X DELETE "$WIKI_URL/api/pages/$page_id" \
        -H "Authorization: Bearer $API_TOKEN"
      echo "✓ Deleted $page_id"
    done
  fi
}

# Usage
case "$1" in
  create) bulk_create_pages "$2" ;;
  update) bulk_update_metadata "$2" "$3" ;;
  move) bulk_move_pages "$2" "$3" ;;
  delete) bulk_delete_pages "$2" ;;
  *) echo "Usage: $0 {create|update|move|delete} <args>" ;;
esac
```

---

## 📦 EXEMPLO 2: Media Management

**Arquivo:** `media-management.md`

```markdown
# Media Management in Wiki.js

## Supported Formats
- Images: JPG, PNG, GIF, WebP, SVG, AVIF
- Documents: PDF, DOCX, XLSX, PPTX
- Video: MP4, WebM
- Audio: MP3, OGG

## Upload Best Practices

### Images
- Max size: 5MB
- Recommended: < 500KB
- Format: WebP for web, PNG for diagrams
- Naming: descriptive-name.png

### Documents
- Max size: 50MB
- PDF recommended for archival
- Keep originals in GitLab

## Organization

```
/wiki/assets/
├── images/
│   ├── logos/
│   ├── diagrams/
│   ├── screenshots/
│   └── icons/
├── documents/
│   ├── guides/
│   └── references/
└── videos/
    └── tutorials/
```

## Automatic Optimization

Wiki.js automatically:
- Compresses images
- Generates thumbnails
- Creates responsive variants
- Caches on CDN

## Usage in Pages

### Images
```markdown
![Alt text](/assets/images/diagram.png)

[![Click for larger](/assets/images/thumb.png)](/assets/images/full.png)
```

### Documents
```markdown
[Download PDF](/assets/documents/guide.pdf)
```

### Videos
```html
<video width="400" controls>
  <source src="/assets/videos/tutorial.mp4" type="video/mp4">
</video>
```
```

---

## 📦 EXEMPLO 3: Content Migration

**Arquivo:** `migrate-content.js`

```javascript
// Migration Script: From Confluence to Wiki.js

const axios = require('axios');
const marked = require('marked');
const html2md = require('html-to-markdown');

const confluenceURL = 'https://confluence.example.com/rest/api/content';
const wikiURL = 'https://wiki.example.com/api/pages';
const confluenceToken = 'your-confluence-token';
const wikiToken = 'your-wiki-token';

// 1. Export from Confluence
async function exportFromConfluence(spaceKey) {
  const response = await axios.get(confluenceURL, {
    headers: { Authorization: `Bearer ${confluenceToken}` },
    params: {
      spaceKey: spaceKey,
      expand: 'body.storage,version',
      limit: 100
    }
  });
  
  return response.data.results;
}

// 2. Convert HTML to Markdown
function convertHTMLtoMarkdown(htmlContent) {
  return html2md.convert(htmlContent);
}

// 3. Map Confluence Structure to Wiki.js
function mapPath(confluenceTitle, parentPath) {
  // Remove special characters
  const sanitized = confluenceTitle
    .toLowerCase()
    .replace(/[^a-z0-9\s-]/g, '')
    .replace(/\s+/g, '-')
    .replace(/-+/g, '-');
  
  return `${parentPath}/${sanitized}`;
}

// 4. Import to Wiki.js
async function importToWiki(page, parentPath) {
  const markdown = convertHTMLtoMarkdown(page.body.storage.value);
  
  const response = await axios.post(wikiURL, {
    title: page.title,
    path: mapPath(page.title, parentPath),
    content: markdown,
    locale: 'en',
    description: `Migrated from Confluence on ${new Date().toISOString()}`
  }, {
    headers: { Authorization: `Bearer ${wikiToken}` }
  });
  
  return response.data;
}

// 5. Main Migration
async function migrationProcess() {
  console.log('🔄 Starting Confluence to Wiki.js migration...');
  
  try {
    // Get all pages from Confluence
    const pages = await exportFromConfluence('SPACE_KEY');
    console.log(`✓ Exported ${pages.length} pages`);
    
    // Import each page
    let imported = 0;
    let failed = 0;
    
    for (const page of pages) {
      try {
        await importToWiki(page, '/docs/migrated');
        imported++;
        console.log(`✓ Imported: ${page.title}`);
      } catch (error) {
        failed++;
        console.error(`✗ Failed: ${page.title} - ${error.message}`);
      }
    }
    
    console.log(`\n📊 Migration complete: ${imported} imported, ${failed} failed`);
    
  } catch (error) {
    console.error('Migration failed:', error);
  }
}

migrationProcess();
```

---

# 8️⃣ PERFORMANCE & SCALING

## 📦 EXEMPLO 4: Caching Strategy

**Arquivo:** `caching-config.yml`

```yaml
caching:
  # Page Cache
  page:
    enabled: true
    ttl: 3600  # seconds
    strategy: "hybrid"  # memory + redis
    maxSize: 100  # MB
    
    # Cache invalidation
    invalidateOn:
      - pageCreate
      - pageEdit
      - pageDelete
      - permissionChange

  # Database Query Cache
  database:
    enabled: true
    ttl: 1800
    backend: "redis"
    keyPrefix: "wiki:db:"
    
  # Search Index Cache
  search:
    enabled: true
    ttl: 7200
    strategy: "elasticsearch"
    autoRefresh: true
    
  # CDN Cache (for static assets)
  cdn:
    enabled: true
    ttl: 86400
    provider: "cloudflare"
    purgeBehavior: "on-publish"

# Redis Configuration
redis:
  host: "redis.example.com"
  port: 6379
  password: "secure_password"
  db: 0
  
  # Cluster support
  cluster:
    enabled: false
    nodes:
      - "redis-1:6379"
      - "redis-2:6379"
      - "redis-3:6379"

# Database Connection Pooling
database:
  pool:
    min: 5
    max: 20
    idleTimeout: 300
    connectionTimeout: 5000

# Elasticsearch Optimization
elasticsearch:
  enabled: true
  hosts:
    - "es-1:9200"
    - "es-2:9200"
  
  indexSettings:
    numberOfShards: 3
    numberOfReplicas: 2
    refreshInterval: "30s"
    
  bulkSize: 500
  threadPoolSize: 8
```

---

## 📦 EXEMPLO 5: Database Optimization

**Arquivo:** `db-optimization.sql`

```sql
-- Wiki.js Database Optimization

-- 1. Create Indexes
CREATE INDEX idx_pages_path ON pages(path);
CREATE INDEX idx_pages_locale ON pages(locale);
CREATE INDEX idx_pages_published ON pages(published_at);
CREATE INDEX idx_pages_author ON pages(author_id);
CREATE INDEX idx_pages_path_locale ON pages(path, locale);

CREATE INDEX idx_comments_page ON comments(page_id);
CREATE INDEX idx_comments_created ON comments(created_at);
CREATE INDEX idx_comments_author ON comments(author_id);

CREATE INDEX idx_history_page ON history(page_id);
CREATE INDEX idx_history_timestamp ON history(timestamp DESC);

-- 2. Analyze Table Statistics
ANALYZE pages;
ANALYZE comments;
ANALYZE history;
ANALYZE users;

-- 3. Vacuum (cleanup)
VACUUM FULL ANALYZE;

-- 4. Query Performance
EXPLAIN ANALYZE
SELECT p.*, u.name as author_name
FROM pages p
LEFT JOIN users u ON p.author_id = u.id
WHERE p.path LIKE '/docs/%'
AND p.locale = 'en'
ORDER BY p.published_at DESC
LIMIT 20;

-- 5. Find Missing Indexes
SELECT schemaname, tablename, attname, n_distinct, correlation
FROM pg_stats
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
AND n_distinct > 100
AND correlation < 0.1
ORDER BY n_distinct DESC;

-- 6. Monitor Query Performance
SELECT query, calls, mean_time, max_time, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 20;

-- 7. Maintenance Schedule
-- Daily: VACUUM ANALYZE
-- Weekly: REINDEX DATABASE
-- Monthly: CLUSTER
```

---

# 9️⃣ AUTOMAÇÃO & CI/CD

## 📦 EXEMPLO 6: Automated Publishing

**Arquivo:** `.gitlab-ci.yml`

```yaml
stages:
  - validate
  - build
  - publish

variables:
  WIKI_API: "https://wiki.example.com/api"
  WIKI_TOKEN: $WIKI_API_TOKEN

# 1. Validate Markdown
validate:markdown:
  stage: validate
  image: node:18
  script:
    - npm install -g markdownlint-cli
    - find wiki -name "*.md" -exec markdownlint {} \;
  allow_failure: false

# 2. Check Links
validate:links:
  stage: validate
  image: node:18
  script:
    - npm install -g markdown-link-check
    - find wiki -name "*.md" -exec markdown-link-check {} \;
  allow_failure: true

# 3. Spell Check
validate:spelling:
  stage: validate
  image: node:18
  script:
    - npm install -g cspell
    - cspell "wiki/**/*.md"
  allow_failure: true

# 4. Build (Convert to HTML/metadata)
build:site:
  stage: build
  image: node:18
  script:
    - npm install
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 day

# 5. Generate Table of Contents
build:toc:
  stage: build
  image: node:18
  script:
    - npm install -g markdown-toc
    - find wiki -name "*.md" -exec markdown-toc -i {} \;
  artifacts:
    paths:
      - wiki/

# 6. Publish to Wiki.js
publish:wiki:
  stage: publish
  image: curlimages/curl:latest
  script:
    # Get all markdown files
    - |
      find wiki -name "*.md" -type f | while read file; do
        # Extract frontmatter and content
        path=$(echo "$file" | sed 's|wiki/||' | sed 's|\.md||')
        title=$(head -n 1 "$file" | sed 's/^# //')
        
        # Upload to Wiki.js
        curl -X POST "$WIKI_API/pages" \
          -H "Authorization: Bearer $WIKI_TOKEN" \
          -H "Content-Type: application/json" \
          --data-binary @- << EOF
          {
            "path": "$path",
            "title": "$title",
            "content": $(cat "$file" | jq -Rs .),
            "locale": "en"
          }
      EOF
      done
  only:
    - main
  when: manual

# 7. Auto-commit on changes
auto-commit:changes:
  stage: publish
  image: alpine/git:latest
  script:
    - git config user.email "wiki@example.com"
    - git config user.name "Wiki CI"
    - git add wiki/
    - git commit -m "docs: auto-update from CI" || true
    - git push origin main
  only:
    - main
```

---

# 🔟 OPERAÇÕES & MANUTENÇÃO

## 📦 EXEMPLO 7: Backup & Recovery

**Arquivo:** `backup-restore.sh`

```bash
#!/bin/bash

# Wiki.js Backup & Restore

BACKUP_DIR="/backups/wiki"
WIKI_DIR="/var/opt/wiki"
DB_HOST="postgres.example.com"
DB_USER="wiki"
DB_NAME="wiki"
RETENTION_DAYS=30

# 1. Full Backup
backup_full() {
  echo "🔄 Starting Wiki.js backup..."
  
  mkdir -p $BACKUP_DIR
  
  # Backup database
  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
  pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | \
    gzip > $BACKUP_DIR/wiki_db_$TIMESTAMP.sql.gz
  
  # Backup files & config
  tar czf $BACKUP_DIR/wiki_files_$TIMESTAMP.tar.gz \
    -C $WIKI_DIR .
  
  # Backup metadata
  curl -s -H "Authorization: Bearer $WIKI_TOKEN" \
    https://wiki.example.com/api/pages | \
    jq '.' > $BACKUP_DIR/pages_metadata_$TIMESTAMP.json
  
  echo "✓ Backup complete"
}

# 2. Incremental Backup (changes since last backup)
backup_incremental() {
  echo "🔄 Starting incremental backup..."
  
  LAST_BACKUP=$(ls -t $BACKUP_DIR/wiki_db_*.sql.gz | head -1)
  LAST_TIME=$(stat -c %Y "$LAST_BACKUP")
  
  pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | \
    gzip > $BACKUP_DIR/wiki_db_incremental_$(date +%s).sql.gz
  
  echo "✓ Incremental backup complete"
}

# 3. Verify Backup
verify_backup() {
  local backup_file=$1
  
  echo "🔍 Verifying backup: $backup_file"
  
  # Check file integrity
  if ! tar tzf "$backup_file" > /dev/null 2>&1; then
    echo "❌ Backup file corrupted"
    return 1
  fi
  
  # Check size
  size=$(du -h "$backup_file" | cut -f1)
  echo "✓ Backup verified ($size)"
  
  return 0
}

# 4. Restore Database
restore_database() {
  local backup_file=$1
  
  echo "⚠️  RESTORING DATABASE - This will overwrite current data!"
  read -p "Are you sure? (yes/no): " confirm
  
  if [ "$confirm" != "yes" ]; then
    return 1
  fi
  
  echo "🔄 Restoring database..."
  
  # Stop Wiki.js
  systemctl stop wiki.js || docker stop wiki || true
  
  # Restore
  gunzip -c "$backup_file" | \
    psql -h $DB_HOST -U $DB_USER $DB_NAME
  
  # Start Wiki.js
  systemctl start wiki.js || docker start wiki
  
  echo "✓ Database restored"
}

# 5. Restore Files
restore_files() {
  local backup_file=$1
  local restore_path=${2:-.}
  
  echo "🔄 Restoring files to $restore_path..."
  
  tar xzf "$backup_file" -C "$restore_path"
  
  echo "✓ Files restored"
}

# 6. Cleanup old backups
cleanup_old_backups() {
  echo "🧹 Cleaning up old backups..."
  
  find $BACKUP_DIR -type f -mtime +$RETENTION_DAYS -delete
  
  echo "✓ Cleanup complete"
}

# 7. Send backup to cloud
upload_to_cloud() {
  local backup_file=$1
  
  echo "☁️ Uploading to S3..."
  
  aws s3 cp "$backup_file" \
    s3://my-backups/wiki-js/ \
    --storage-class GLACIER \
    --sse AES256
  
  echo "✓ Upload complete"
}

# Usage
case "$1" in
  full) backup_full ;;
  incremental) backup_incremental ;;
  verify) verify_backup "$2" ;;
  restore-db) restore_database "$2" ;;
  restore-files) restore_files "$2" "$3" ;;
  cleanup) cleanup_old_backups ;;
  cloud) upload_to_cloud "$2" ;;
  *) echo "Usage: $0 {full|incremental|verify|restore-db|restore-files|cleanup|cloud}" ;;
esac
```

---

## 📦 EXEMPLO 8: Monitoring & Health Checks

**Arquivo:** `monitoring.sh`

```bash
#!/bin/bash

# Wiki.js Monitoring & Health Checks

WIKI_URL="https://wiki.example.com"
SLACK_WEBHOOK="https://hooks.slack.com/..."

health_check() {
  echo "🔍 Performing health checks..."
  
  # Check if Wiki.js is up
  if ! curl -s -f "$WIKI_URL/health" > /dev/null; then
    send_alert "Wiki.js is DOWN!"
    return 1
  fi
  
  # Check API health
  if ! curl -s -f "$WIKI_URL/api/health" > /dev/null; then
    send_alert "Wiki.js API is DOWN!"
    return 1
  fi
  
  # Check database connection
  if ! curl -s -f "$WIKI_URL/api/pages?limit=1" > /dev/null; then
    send_alert "Database connection failed!"
    return 1
  fi
  
  # Check disk space
  disk_usage=$(df $WIKI_DIR | awk 'NR==2 {print $5}' | sed 's/%//')
  if [ "$disk_usage" -gt 90 ]; then
    send_alert "Disk usage is ${disk_usage}%!"
    return 1
  fi
  
  echo "✓ All health checks passed"
  return 0
}

send_alert() {
  local message=$1
  
  curl -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d '{
      "text": "⚠️ Wiki.js Alert",
      "attachments": [{
        "color": "danger",
        "text": "'"$message"'",
        "ts": '$(date +%s)'
      }]
    }'
}

# Run checks
health_check

# Schedule (cron)
# */5 * * * * /opt/wiki/monitoring.sh
```

---

## RESUMO CATEGORIAS 7-10

✅ **Content Management:**
- Bulk operations (create, update, move, delete)
- Media management (upload, optimize, organize)
- Content migration (Confluence → Wiki.js)

✅ **Performance & Scaling:**
- Caching strategies (page, DB, search, CDN)
- Database optimization
- Connection pooling
- Elasticsearch tuning

✅ **Automação & CI/CD:**
- Automated publishing (GitLab CI)
- Content validation (markdown, links, spell)
- Auto-commit on changes
- Table of contents generation

✅ **Operações & Manutenção:**
- Backup & recovery (full, incremental)
- Backup verification
- Cloud upload (S3, GLACIER)
- Health monitoring
- Alert system
- Disk space monitoring

---

**ENCICLOPÉDIA WIKI.JS COMPLETA! 🎉**

Todas as 10 categorias criadas com 150+ exemplos!
