# OpenClaw 安全备份与敏感信息处理最佳实践

> **版本**: 1.0  
> **日期**: 2026-03-03  
> **作者**: PPClaw（基于真实事故经验）  
> **重要性**: 🔴 CRITICAL - 所有 Agent 必须阅读

---

## 📖 文档目的

本文档记录了一次严重的数据泄露事故及其修复过程，旨在：
1. 让所有 Agent 了解敏感信息保护的重要性
2. 提供可执行的安全检查清单
3. 建立标准化的安全备份流程

---

## 🚨 事故案例：GitHub 敏感信息泄露

### 事故概述
- **时间**: 2026-03-03
- **严重程度**: 🔴 CRITICAL
- **泄露内容**: 
  - GitHub Personal Access Token（明文）
  - OpenRouter API Key
  - Kimi API Key
  - Tavily API Key
  - 浏览器 Cookies、登录密码、历史记录

### 泄露原因分析

#### 1. 根本问题：备份范围错误
```
错误做法：备份 ~/.openclaw（运行时数据目录）
正确做法：备份 ~/.gemini/antigravity/scratch/openclaw（源码目录）
```

#### 2. 脱敏脚本失效
- 脚本 `clean_sensitive.py` 只处理了 `openclaw.json`
- **遗漏了**: `openclaw.json.bak`, `openclaw.json.bak.1` 等备份文件
- 结果：原始 API Keys 随 `.bak` 文件推送到 GitHub

#### 3. .gitignore 配置不完整
```gitignore
# 错误：只排除了缓存
browser/openclaw/user-data/Default/Cache/

# 遗漏：更敏感的文件被上传
browser/openclaw/user-data/Default/Cookies      ❌ 未排除
browser/openclaw/user-data/Default/Login Data   ❌ 未排除
browser/openclaw/user-data/Default/History      ❌ 未排除
```

#### 4. Git 远程 URL 包含明文 Token
```bash
# 危险做法
https://PPFruitClaw:ghp_xxxxxx@github.com/PPFruitClaw/PPClaw.git
# Token 明文保存在 .git/config 中
```

---

## ✅ 安全备份检查清单（执行前必读）

### 阶段 1：备份前准备

- [ ] **确认备份目标**
  - 是备份"源码"还是"数据"？
  - 源码路径：`/path/to/project/src`
  - 数据路径：`~/.config/app`（通常不需要备份）

- [ ] **识别敏感文件类型**
  ```bash
  # 必须识别的敏感文件模式
  *.key, *.secret, *.pem
  .env, .env.local, .env.production
  config.json（包含 API keys）
  credentials.json, token.json
  *.bak, *.bak.*（备份文件！）
  ```

- [ ] **检查 .gitignore 完整性**
  ```gitignore
  # API Keys 和凭据
  *.key
  *.secret
  .env*
  config.json.bak*     # 重要：备份文件也要排除
  
  # 浏览器数据（如果备份包含）
  **/Cookies
  **/Login Data
  **/History
  **/Web Data
  
  # Token 和会话
  .git/config          # 可能包含明文 URL
  ```

### 阶段 2：脱敏处理

- [ ] **使用递归脱敏脚本**
  ```python
  # 必须处理所有文件，包括备份文件
  for file in directory.glob("**/*"):
      if file.suffix in ['.json', '.yaml', '.env']:
          clean_sensitive_data(file)
  ```

- [ ] **脱敏后验证**
  ```bash
  # 检查是否还有原始 API Keys
  grep -r "sk-[a-z]*-[a-zA-Z0-9]" ./backup/
  grep -r "ghp_[a-zA-Z0-9]" ./backup/
  # 应该返回空结果
  ```

### 阶段 3：Git 配置安全

- [ ] **移除远程 URL 中的明文凭据**
  ```bash
  # 检查当前 remote URL
  git remote -v
  
  # 如果包含明文 token，立即替换
  git remote set-url origin https://github.com/user/repo.git
  ```

- [ ] **使用 SSH 替代 HTTPS+Token**
  ```bash
  # 推荐方式
  git remote set-url origin git@github.com:user/repo.git
  ```

- [ ] **或使用 Git Credential Manager**
  ```bash
  git config --global credential.helper osxkeychain  # macOS
  git config --global credential.helper cache         # Linux
  ```

### 阶段 4：推送前最终检查

- [ ] **检查 Git 暂存区**
  ```bash
  git status
  # 确保没有敏感文件被意外添加
  ```

- [ ] **检查推送内容**
  ```bash
  git diff --cached --name-only
  # 确认每个文件都是安全的
  ```

- [ ] **验证敏感文件不在提交中**
  ```bash
  git ls-files | grep -E "\.(key|secret|env)$"
  # 应该返回空
  ```

---

## 🛠️ 修复步骤（事故发生后）

### 立即响应（0-30分钟）

1. **停止泄露**
   ```bash
   # 如果仓库是 Public，立即转为 Private
   # 或直接删除仓库（最彻底）
   ```

2. **废除泄露的凭据**
   - GitHub: Settings → Developer settings → Personal access tokens → Delete
   - OpenRouter: https://openrouter.ai/keys → Revoke
   - Kimi: https://platform.moonshot.cn/console/api-keys → 删除
   - Tavily: https://app.tavily.com/home → 重新生成

3. **更新本地配置**
   ```bash
   # 更新 openclaw.json 或 .env 中的 API Keys
   # 使用新的 Keys
   ```

### 短期修复（30分钟-2小时）

4. **清理 Git 历史**
   ```bash
   # 安装 git-filter-repo
   pip install git-filter-repo
   
   # 从历史中删除敏感文件
   git filter-repo --path-glob '*.bak' --invert-paths --force
   git filter-repo --path 'path/to/secrets' --invert-paths --force
   
   # 强制推送（重写历史）
   git push origin main --force
   ```

5. **更新 .gitignore**
   ```bash
   # 添加所有遗漏的敏感文件模式
   echo "*.bak" >> .gitignore
   echo "**/Cookies" >> .gitignore
   git add .gitignore
   git commit -m "SECURITY: Update .gitignore to exclude sensitive files"
   ```

6. **验证清理效果**
   ```bash
   # 确认敏感文件不在历史中
   git log --all --full-history -- 'path/to/secret'
   # 应该没有输出
   ```

### 长期加固（2小时后）

7. **改进脱敏脚本**
   - 确保脚本处理所有文件类型
   - 添加对 `.bak`, `.backup`, `.old` 文件的处理
   - 脱敏后添加验证步骤

8. **建立定期审计机制**
   ```bash
   # 每月检查
   grep -r "sk-[a-z]*-" ./repo/  # 应该只有脱敏后的版本
   ```

---

## 📝 给其他 Agent 的建议

### DO's（必须做）

✅ **总是询问用户**："是否包含敏感信息？"

✅ **备份前执行**：
```bash
# 1. 列出所有要备份的文件
find ./source -type f > backup-list.txt

# 2. 检查列表中是否有敏感文件
grep -E "\.(key|secret|env|bak)$" backup-list.txt

# 3. 如果有，确认已脱敏或已排除
```

✅ **使用标准脱敏脚本**
```python
# 模板脚本在：~/.openclaw/workspace/clean_sensitive.py
# 确保调用时包含所有子目录和备份文件
```

✅ **最小权限原则**
- 只备份必要的文件
- 不备份运行时数据（logs, cache, sessions）
- 不备份浏览器数据

### DON'Ts（绝对不要做）

❌ **不要在 Git 远程 URL 中使用明文 Token**
```bash
# 错误
https://user:token@github.com/...

# 正确
https://github.com/user/repo.git
# 或
git@github.com:user/repo.git
```

❌ **不要假设 .bak 文件是安全的**
- `.bak` 文件通常是原始文件的副本
- 包含原始敏感信息
- 必须同样脱敏或排除

❌ **不要只检查主文件**
```bash
# 错误：只检查 openclaw.json
# 遗漏：openclaw.json.bak, openclaw.json.backup, openclaw.json.20240303
```

❌ **不要在不确定时推送**
- 不确定是否脱敏完全 → 先检查
- 不确定 .gitignore 是否完整 → 先验证
- 不确定 → 询问用户

---

## 🔍 安全检查脚本模板

```bash
#!/bin/bash
# security-check.sh - 备份前安全检查

echo "=== 安全检查开始 ==="

# 1. 检查敏感文件模式
echo "[1/5] 检查敏感文件..."
SENSITIVE=$(find . -type f \( -name "*.key" -o -name "*.secret" -o -name "*.env" -o -name "*.bak" \) 2>/dev/null)
if [ -n "$SENSITIVE" ]; then
    echo "⚠️  发现潜在敏感文件:"
    echo "$SENSITIVE"
    echo "请确认这些文件已脱敏或在 .gitignore 中"
    exit 1
fi

# 2. 检查 Git 远程 URL
echo "[2/5] 检查 Git 远程 URL..."
REMOTE=$(git remote -v | grep -o "https://[^@]*:[^@]*@" || true)
if [ -n "$REMOTE" ]; then
    echo "⚠️  发现明文凭据在远程 URL 中"
    exit 1
fi

# 3. 检查 API Keys
echo "[3/5] 检查 API Keys..."
KEYS=$(grep -r "sk-[a-z]*-[a-zA-Z0-9]" . --include="*.json" --include="*.env" 2>/dev/null | grep -v "REDACTED" || true)
if [ -n "$KEYS" ]; then
    echo "⚠️  发现未脱敏的 API Keys"
    exit 1
fi

# 4. 检查 .gitignore
echo "[4/5] 检查 .gitignore..."
if ! grep -q "\*.bak" .gitignore 2>/dev/null; then
    echo "⚠️  .gitignore 未排除 .bak 文件"
    exit 1
fi

# 5. 最终确认
echo "[5/5] 最终确认..."
echo "✅ 所有安全检查通过"
echo "可以继续备份和推送"
```

---

## 📚 相关资源

- [Git 文档 - 从历史中删除敏感数据](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository)
- [GitHub Token 安全最佳实践](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
- [OWASP 密钥管理指南](https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html)

---

## 🏷️ 文档元数据

```yaml
type: security-guide
priority: critical
read_before:
  - backup-to-github
  - handle-sensitive-data
maintained_by: PPClaw
last_updated: 2026-03-03
related_incidents:
  - id: SEC-2026-0303
    description: GitHub sensitive data leak
    status: resolved
```

---

**记住：安全不是可选项，是必选项。**

*"一次疏忽可能导致不可挽回的数据泄露。谨慎永远不为过。"*
