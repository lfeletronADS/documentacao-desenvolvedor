# 🔐 Auditoria Completa: Segurança no Git & GitHub

**Status:** 🚨 CRÍTICO - Análise de exposição de credenciais  
**Data:** 20/08/2026  
**Objetivo:** Proteger repositórios GitHub do vazamento de senhas/tokens

---

## ⚠️ O Problema

Seus projetos podem estar expostos a:

```
🚨 CREDENCIAIS em histórico de commits
   ├─ Senhas de database
   ├─ API keys
   ├─ Tokens JWT/OAuth
   ├─ URLs de conexão (user:password)
   └─ Chaves SSH privadas

🚨 ARQUIVOS SENSÍVEIS
   ├─ .env com credenciais
   ├─ config.json com secrets
   ├─ Arquivos de configuração
   └─ Certificados privados

🚨 HISTÓRICO NÃO LIMPO
   ├─ Commits com dados sensíveis
   ├─ Branches antigos com vulns
   ├─ Tags que exposição segredos
   └─ Pull requests históricos
```

---

## 📋 Plano de Ação (4 Fases)

```
FASE 1: DESCOBERTA
└─ Auditar seus repositórios GitHub

FASE 2: ANÁLISE PROVA DE CONCEITO
└─ Explorar .git do lab (demonstração)

FASE 3: LIMPEZA
└─ Remover credenciais do histórico

FASE 4: PROTEÇÃO
└─ Implementar .gitignore + resetar secrets
```

---

## ✅ FASE 1: DESCOBERTA

### Passo 1: Listar seus repositórios

```bash
# Ver repositórios locais
find ~/ -name ".git" -type d 2>/dev/null | head -20

# Ou, no GitHub via CLI:
gh repo list --limit 100
```

**Você tem acesso ao seu GitHub para clonar/auditar?**

---

## 🔍 FASE 2: PROVA DE CONCEITO - Explorar .git do Lab

Vamos demonstrar como extrair dados do `.git` exposto:

### Passo 1: Extrair arquivo de configuração

```bash
# Obter a URL do repositório (já temos)
curl -s http://192.168.0.102:8083/.git/config
```

**Output esperado:** `git@github.com:lfeletronADS/imob.git`

### Passo 2: Verificar histórico de commits

```bash
# Técnica: Enumerar arquivo HEAD
curl -s http://192.168.0.102:8083/.git/HEAD

# Técnica: Listar branches
curl -s http://192.168.0.102:8083/.git/packed-refs

# Técnica: Acessar logs
curl -s http://192.168.0.102:8083/.git/logs/HEAD
```

### Passo 3: Extrair Commits (Buscar credenciais)

```bash
# Se conseguirmos acesso ao objeto git
# Cada commit pode conter:
# - Author name + email
# - Commit message (pode ter credenciais!)
# - Diffs (pode expor senhas em código)
# - Arquivo .env com secrets
```

### Passo 4: Automatizar Extração

```bash
#!/bin/bash
# git-secrets-finder.sh

TARGET_URL="http://192.168.0.102:8083"

echo "=== SCANNING FOR EXPOSED SECRETS ==="

# Tentar acessar arquivos sensíveis comuns
SENSITIVE_FILES=(
  ".env"
  "config.json"
  "secrets.json"
  ".env.example"
  "package.json"
  ".git/config"
  ".git/HEAD"
  ".gitignore"
)

for file in "${SENSITIVE_FILES[@]}"; do
  echo -n "Checking $file: "
  curl -s -o /dev/null -w "%{http_code}" "$TARGET_URL/$file"
  echo ""
done

# Procurar por credenciais em commits
echo ""
echo "=== POTENTIAL SECRETS IN COMMITS ==="
curl -s "$TARGET_URL/.git/logs/HEAD" | \
  grep -i "password\|secret\|token\|api_key\|credential" || \
  echo "Nenhum encontrado (ou não acessível)"
```

---

## 🛡️ FASE 3: LIMPEZA - Remover Credenciais do Histórico

### Ferramenta 1: `git filter-branch` (Tradicional)

```bash
# ⚠️ CUIDADO: Reescreve histórico!

cd seu-repositorio

# 1. Remover arquivo sensível de TODO histórico
git filter-branch --tree-filter 'rm -f .env secrets.json config.json' HEAD

# 2. Força push (cuidado!)
git push origin --force --all
```

### Ferramenta 2: `git-filter-repo` (Recomendado)

```bash
# Instalar
pip3 install git-filter-repo

# Remover arquivo de TODO histórico
git filter-repo --path .env --invert-paths

# Push forçado
git push origin --force --all
```

### Ferramenta 3: `BFG Repo Cleaner` (Mais Fácil)

```bash
# Instalar BFG
brew install bfg  # ou apt-get install bfg-repo-cleaner

cd seu-repositorio

# Remover todos os .env de TODO histórico
bfg --delete-files .env

# Remover strings com "password=" de TODO código
bfg --replace-text passwords.txt

# Limpar
git reflog expire --expire=now --all && git gc --prune=now --aggressive

# Push forçado
git push origin --force --all
```

### Exemplo: `.passwords.txt` para BFG

```
# passwords.txt
password=.*
apikey=.*
token=.*
secret=.*
```

---

## 🔐 FASE 4: PROTEÇÃO Futura

### Passo 1: Criar `.gitignore` Correto

```gitignore
# .gitignore — Protege seus segredos

# Variáveis de ambiente
.env
.env.local
.env.*.local

# Configuração com secrets
config.json
secrets.json
config/production.js

# Chaves SSH/Certs
*.pem
*.key
*.crt
id_rsa
id_dsa
*.pub

# Credenciais
.ssh/
.aws/
.azure/
.gcloud/

# IDEs (podem ter configurações sensíveis)
.idea/
.vscode/
*.sublime-project

# Dependências (podem ter cached credentials)
node_modules/
vendor/
.bundle/

# Logs com possíveis dados sensíveis
*.log
logs/
npm-debug.log

# Backups/temporários
*.bak
*.swp
*.swo
*~
.DS_Store

# Arquivos gerados (podem conter dados)
dist/
build/
.next/

# Database files
*.db
*.sqlite
*.sqlite3
```

### Passo 2: Resetar Credenciais (CRÍTICO!)

Se você acha que exposição senhas:

```bash
# 1. GitHub: Resetar Personal Access Tokens
# Settings → Developer settings → Personal access tokens
# → Delete old tokens, create new ones

# 2. Banco de Dados: Resetar senhas
# ALTER USER 'user'@'localhost' IDENTIFIED BY 'new_password';

# 3. AWS/Google Cloud: Rotate access keys
# Security credentials → Create new key pair → Delete old

# 4. API Keys: Regenerate
# Go to each service and regenerate keys

# 5. Certificados: Se certificados privados foram expostos
# Revoke e reemitir
```

### Passo 3: Implementar Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

# Prevenir commit de credenciais

PATTERNS=(
  'password'
  'apikey\|api_key'
  'secret\|secrets'
  'token'
  'private.*key'
)

for pattern in "${PATTERNS[@]}"; do
  if git diff --cached | grep -i "$pattern"; then
    echo "❌ Tentativa de commit com credencial detectada!"
    echo "Padrão encontrado: $pattern"
    exit 1
  fi
done

exit 0
```

Instalação:
```bash
chmod +x .git/hooks/pre-commit
```

### Passo 4: Usar `git-secrets` Tool

```bash
# Instalar
brew install git-secrets  # ou apt-get install

# Configurar no repositório
cd seu-repositorio
git secrets --install

# Adicionar padrões personalizados
git secrets --add 'password\s*=\s*'
git secrets --add 'api[_-]?key'
git secrets --add 'secret'

# Escanear histórico
git secrets --scan -r .

# Hook automático para cada commit
git secrets --install -f
```

---

## 📊 Checklist: Segurança no Git

### Por Repositório

- [ ] `.gitignore` configurado corretamente
- [ ] Sem `.env` commitado
- [ ] Sem `config.json` com secrets
- [ ] Sem chaves SSH/privadas
- [ ] Sem banco de dados exposto
- [ ] Sem URLs com `user:password`
- [ ] Histórico limpo de credenciais

### No GitHub

- [ ] Personal Access Tokens rotacionados
- [ ] Webhooks seguros (use HTTPS)
- [ ] Deploy keys configuradas (não credenciais)
- [ ] Branch protection ativada
- [ ] Secrets (GitHub Actions) usadas corretamente
- [ ] Verificação de commit signatures

### Ferramentas Implementadas

- [ ] `.gitignore` robusto
- [ ] Git hooks (pre-commit)
- [ ] `git-secrets` instalado
- [ ] Scanning automático ativo

---

## 🚨 SEU CASO: GitHub Pessoal

**URL:** https://github.com/repos (não é URL exata, mas vamos escanear)

### Ação Imediata (TODO):

```bash
# 1. Listar repositórios
gh repo list --limit 100

# 2. Para cada repo, fazer auditoria
for repo in $(gh repo list --json nameWithOwner -q '.[].nameWithOwner'); do
  echo "=== Auditando $repo ==="
  
  # Clonar
  git clone https://github.com/$repo /tmp/audit-$repo
  
  # Escanear por secrets
  git secrets --scan -r /tmp/audit-$repo
  
  # Limpar
  rm -rf /tmp/audit-$repo
done
```

### Se Encontrar Credenciais:

1. **IMEDIATO:** Resetar aquela credencial (senha, token, chave)
2. **HOJE:** Limpar histórico com `git filter-repo` ou `BFG`
3. **HOJE:** Force push
4. **AMANHÃ:** Verificar acesso não autorizado

---

## 🎓 Lições Aprendidas

### NUNCA comitar:
```
❌ .env, config.json, secrets.json
❌ Chaves SSH privadas (id_rsa, pem, key)
❌ Tokens / API keys / JWT
❌ Senhas de database em código
❌ URLs com credenciais (http://user:pass@host)
❌ Certificados privados
❌ Arquivos de configuração com secrets
```

### SEMPRE usar:
```
✅ .gitignore robusto
✅ Environment variables (.env via .gitignore)
✅ GitHub Secrets (para CI/CD)
✅ Git hooks (pre-commit scanning)
✅ git-secrets (double-check)
✅ Code review antes de push
✅ Branch protection + required reviews
```

### ESTRUTURA RECOMENDADA:

```
seu-projeto/
├── .env.example          ← Template SEM valores
├── .env                  ← ← ← NÃO COMMITAR! (gitignore)
├── .gitignore            ← Protege seus segredos
├── .git/hooks/pre-commit ← Previne commits errados
├── config/
│   ├── config.example.json
│   └── config.json       ← ← ← NÃO COMMITAR!
└── src/
    └── app.js
```

---

## 📞 Próximos Passos

### Você quer fazer:

1. **Explorar a vulnerabilidade .git do lab** (prova de conceito)
2. **Auditar seus repositórios GitHub** (listar e verificar)
3. **Limpar histórico de um repositório** (remover credenciais)
4. **Implementar proteção** (.gitignore + git-secrets)
5. **Resetar credenciais** (mudar senhas/tokens)

**Qual é a prioridade?**

---

## ⚠️ Aviso Legal

- Auditoria de seus próprios repos: ✅ Legal
- Auditoria de repos de terceiros sem permissão: ❌ Ilegal
- Usar credenciais encontradas: ❌ Crime

Este guia é para **proteger SEUS sistemas**, não para invadir outros.

---

*Documento de segurança crítico*  
*Última atualização: Agosto/2026*
