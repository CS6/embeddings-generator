# Debug Guide: 驗證 Embeddings Generator 執行結果

## 可能的問題原因

### 1. 資料庫 Schema 問題
程式碼使用 `schema: 'docs'`，如果您的 Supabase 資料庫沒有建立這個 schema，資料會寫入失敗。

### 2. 檔案未被發現
如果 `docs-root-path` 設定錯誤或沒有找到 `.md`/`.mdx` 檔案，`embeddingSources.length` 會是 0。

### 3. 所有檔案被跳過
如果檔案內容沒有變更（checksum 相同），且不是 refresh 模式，會跳過處理。

### 4. 資料被刪除
程式最後會刪除 `version` 不等於 `refreshVersion` 的資料，如果執行過程中出錯，可能導致資料被誤刪。

### 5. 錯誤被靜默處理
程式碼中的錯誤只會 console.error，不會導致 action 失敗（除非是最外層的錯誤）。

## 驗證步驟

### 步驟 1: 檢查 GitHub Actions Logs

查看以下關鍵日誌：

1. **發現的檔案數量**：
   ```
   Discovered X pages
   ```
   如果顯示 `Discovered 0 pages`，表示沒有找到檔案。

2. **處理的檔案**：
   ```
   [path/to/file.md] Adding X page sections (with embeddings)
   ```
   如果有看到這個訊息，表示正在處理。

3. **錯誤訊息**：
   ```
   Failed to generate embeddings for...
   Page '...' or one/multiple of its page sections failed to store properly
   ```

### 步驟 2: 檢查 Supabase 資料庫

#### 2.1 檢查 Schema 是否存在

```sql
-- 在 Supabase SQL Editor 執行
SELECT schema_name 
FROM information_schema.schemata 
WHERE schema_name = 'docs';
```

如果沒有結果，需要建立 schema：

```sql
CREATE SCHEMA IF NOT EXISTS docs;
```

#### 2.2 檢查資料表是否存在

```sql
-- 檢查 page 表
SELECT table_name, table_schema 
FROM information_schema.tables 
WHERE table_schema = 'docs' 
AND table_name IN ('page', 'page_section');
```

#### 2.3 檢查資料是否被寫入

```sql
-- 檢查 page 表（即使失敗也可能有記錄，但 checksum 為 null）
SELECT 
  id, 
  path, 
  checksum, 
  type, 
  source,
  version,
  last_refresh,
  created_at
FROM docs.page 
ORDER BY created_at DESC 
LIMIT 10;
```

```sql
-- 檢查 page_section 表
SELECT 
  ps.id,
  ps.page_id,
  ps.slug,
  ps.heading,
  LEFT(ps.content, 50) as content_preview,
  ps.token_count,
  CASE 
    WHEN ps.embedding IS NULL THEN 'NULL'
    ELSE 'HAS_EMBEDDING'
  END as embedding_status
FROM docs.page_section ps
ORDER BY ps.created_at DESC 
LIMIT 10;
```

#### 2.4 檢查是否有 checksum 為 null 的頁面（表示處理失敗）

```sql
-- 找出處理失敗的頁面
SELECT 
  id, 
  path, 
  checksum, 
  type,
  created_at
FROM docs.page 
WHERE checksum IS NULL
ORDER BY created_at DESC;
```

#### 2.5 檢查最後的 version（可能所有資料被刪除）

```sql
-- 檢查是否有任何資料
SELECT COUNT(*) as total_pages FROM docs.page;
SELECT COUNT(*) as total_sections FROM docs.page_section;
```

### 步驟 3: 檢查檔案路徑設定

確認 `docs-root-path` 設定正確：

```yaml
# .github/workflows/generate_embeddings.yml
- uses: supabase/embeddings-generator@v0.x.x
  with:
    docs-root-path: 'docs'  # 確認這個路徑存在且包含 .md/.mdx 檔案
```

### 步驟 4: 檢查檔案過濾規則

程式碼會過濾：
- 非 `.md`/`.mdx` 檔案
- `pages/404.mdx` 檔案

確認您的檔案符合規則。

## 除錯 SQL 查詢腳本

將以下查詢保存到 Supabase SQL Editor 執行：

```sql
-- 完整的診斷查詢
DO $$
DECLARE
  schema_exists BOOLEAN;
  page_count INT;
  section_count INT;
  failed_pages INT;
BEGIN
  -- 檢查 schema
  SELECT EXISTS (
    SELECT 1 FROM information_schema.schemata 
    WHERE schema_name = 'docs'
  ) INTO schema_exists;
  
  RAISE NOTICE 'Schema "docs" exists: %', schema_exists;
  
  IF schema_exists THEN
    -- 檢查資料
    SELECT COUNT(*) INTO page_count FROM docs.page;
    SELECT COUNT(*) INTO section_count FROM docs.page_section;
    SELECT COUNT(*) INTO failed_pages FROM docs.page WHERE checksum IS NULL;
    
    RAISE NOTICE 'Total pages: %', page_count;
    RAISE NOTICE 'Total sections: %', section_count;
    RAISE NOTICE 'Failed pages (checksum IS NULL): %', failed_pages;
    
    -- 顯示最近的頁面
    RAISE NOTICE '--- Recent pages ---';
    FOR rec IN 
      SELECT path, checksum, version, last_refresh 
      FROM docs.page 
      ORDER BY last_refresh DESC NULLS LAST
      LIMIT 5
    LOOP
      RAISE NOTICE 'Path: %, Checksum: %, Version: %, Last Refresh: %', 
        rec.path, 
        COALESCE(LEFT(rec.checksum, 20), 'NULL'),
        rec.version,
        rec.last_refresh;
    END LOOP;
  ELSE
    RAISE WARNING 'Schema "docs" does not exist! Data cannot be stored.';
  END IF;
END $$;
```

## 常見解決方案

### 問題 1: Schema 不存在

**解決方案**：在 Supabase SQL Editor 執行：

```sql
CREATE SCHEMA IF NOT EXISTS docs;
```

然後確認您的資料表在 `docs` schema 下，或修改程式碼中的 schema 設定。

### 問題 2: 資料表不存在

參考 `headless-vector-search` 專案的 migration 檔案建立資料表。

### 問題 3: 檔案未被找到

確認：
- `docs-root-path` 設定正確
- GitHub Actions 中有使用 `actions/checkout@v3` 來 checkout 程式碼
- 檔案路徑相對於 repository root

### 問題 4: 權限問題

確認 `SUPABASE_SERVICE_ROLE_KEY` 是 Service Role Key（不是 anon key），並且有權限：
- 讀寫 `docs.page` 表
- 讀寫 `docs.page_section` 表

### 問題 5: OpenAI API 錯誤

檢查：
- `OPENAI_KEY` 是否正確
- OpenAI 帳號是否有足夠額度
- 模型名稱是否正確（`text-embedding-ada-002` 或 `text-embedding-3-large`）

## 強制重新產生（如果需要）

如果發現資料有問題，可以：

1. 手動刪除所有資料：
```sql
DELETE FROM docs.page_section;
DELETE FROM docs.page;
```

2. 在 workflow 中加入 `shouldRefresh` 參數（如果支援）或手動觸發 action

## 驗證成功標準

當一切正常時，您應該看到：

1. **GitHub Actions Log** 顯示：
   - `Discovered X pages` (X > 0)
   - 每個檔案顯示 `[path] Adding X page sections (with embeddings)`
   - `Embedding generation complete`

2. **資料庫中**：
   - `docs.page` 表有資料
   - `docs.page_section` 表有資料
   - 所有 page 的 `checksum` 不為 NULL
   - `page_section.embedding` 欄位有值（向量陣列）

