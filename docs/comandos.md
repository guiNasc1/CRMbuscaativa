# Cola de comandos — Busca Ativa CRM

> Terminal: **PowerShell**. Pastas: `backend/` (Spring Boot) e `frontend/` (Angular).
> Conferido com Angular CLI 22.2 (testes com **Vitest** por padrão, zoneless por padrão) e Spring Boot 4.1.

---

## 1. Conferir o ambiente

| Comando | Para quê |
|---|---|
| `java -version` | JDK 21 |
| `node -v` / `npm -v` | Node 24 / npm 11 |
| `ng version` | Angular CLI + versões do projeto (rodando dentro de `frontend/`) |
| `git --version` | Git |
| `npm install -g @angular/cli` | instalar/atualizar o Angular CLI global |
| `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` | corrige o erro "ng.ps1 não pode ser carregado" |

---

## 2. Angular (rodar dentro de `frontend/`)

### 2.1 Criar projeto

| Comando | Para quê |
|---|---|
| `ng new buscaativacrm-web --directory frontend --skip-git --style=css --ssr=false` | cria o projeto (rodar na **raiz**) |
| `npm install` | baixa as dependências do `package.json` → `node_modules/` (≈ `mvn dependency:resolve`) |
| `npm ci` | igual, mas exato pelo `package-lock.json` (use após clonar / em servidor) |

### 2.2 Rodar

| Comando | Para quê |
|---|---|
| `ng serve` | servidor de desenvolvimento em http://localhost:4200, recarrega ao salvar |
| `ng serve -o` | idem e já abre o navegador |
| `ng serve --port 4300` | outra porta |

> O `proxy.conf.json` só é lido na inicialização → mudou o proxy, reinicie o `ng serve` (Ctrl+C).

### 2.3 Gerar código (`ng generate` = `ng g`)

| Comando | Gera |
|---|---|
| `ng g component conversas/inbox` | `inbox.ts`, `.html`, `.css`, `.spec.ts` (classe `Inbox`) |
| `ng g component conversas/chat --skip-tests` | idem, sem o `.spec.ts` |
| `ng g service conversas/conversa --type=service` | `conversa.service.ts` (sem `--type`, o arquivo sai `conversa.ts`) |
| `ng g interface conversas/conversa --type=model` | `conversa.model.ts` |
| `ng g enum conversas/direcao` | `direcao.ts` com `enum` |
| `ng g guard auth/auth` | guarda de rota (login — etapa futura) |
| `ng g interceptor auth/auth` | interceptor HTTP (enviar token — etapa futura) |
| `ng g pipe shared/telefone` | pipe de formatação (ex.: `5544999998888` → `(44) 99999-8888`) |
| `ng g component x --dry-run` | **só mostra** o que seria criado, sem criar |

### 2.4 Instalar bibliotecas

| Comando | Para quê |
|---|---|
| `npm install <pacote>` | dependência normal (vai para `dependencies`) |
| `npm install -D <pacote>` | só de desenvolvimento/teste (`devDependencies`) |
| `ng add <pacote>` | instala **e configura** o projeto (só para libs que suportam, ex.: `@angular/material`) |
| `npm install primeng @primeuix/themes` | PrimeNG (o "PrimeFaces do Angular") |
| `npm uninstall <pacote>` | remove |
| `npm outdated` | lista o que tem versão nova |
| `ng update` | lista atualizações do Angular; `ng update @angular/core @angular/cli` aplica com migração automática |

### 2.5 Testar

| Comando | Para quê |
|---|---|
| `ng test` | roda os `*.spec.ts` com Vitest em modo **watch** (reexecuta ao salvar) |
| `ng test --watch=false` | roda uma vez e sai (antes de commit) |
| `ng test --include src/app/conversas` | só os testes de uma pasta |
| `ng test --coverage` | relatório de cobertura em `coverage/` (pede `npm install -D @vitest/coverage-v8` na 1ª vez) |
| `ng lint` | análise estática; na 1ª vez pergunta se instala o `angular-eslint` |
| `ng e2e` | testes ponta a ponta; na 1ª vez pede para escolher (Playwright/Cypress) |

### 2.6 Compilar

| Comando | Para quê |
|---|---|
| `ng build` | build de produção → `dist/buscaativacrm-web/browser/` |
| `ng build --configuration development` | build sem otimização (mais rápido, para depurar) |

---

## 3. Spring Boot (rodar dentro de `backend/`)

`mvnw` = Maven Wrapper (não depende do Maven instalado). No IntelliJ, a aba **Maven** tem os mesmos comandos.

| Comando | Para quê | Equivalente que você conhece |
|---|---|---|
| `.\mvnw spring-boot:run` | sobe a aplicação em http://localhost:8080 | deploy no Tomcat |
| `.\mvnw compile` | só compila | Build do NetBeans/Eclipse |
| `.\mvnw test` | roda todos os testes (JUnit) | — |
| `.\mvnw test -Dtest=AssinaturaMetaTest` | uma classe de teste | — |
| `.\mvnw test -Dtest=AssinaturaMetaTest#validaAssinaturaCorreta` | um método de teste | — |
| `.\mvnw clean package` | testa e gera `target\buscaativacrm-0.0.1-SNAPSHOT.jar` | gerar o WAR |
| `.\mvnw clean package -DskipTests` | gera o JAR sem rodar testes | — |
| `java -jar target\buscaativacrm-0.0.1-SNAPSHOT.jar` | roda o JAR gerado | — |
| `.\mvnw clean` | apaga `target/` (resolve "classe fantasma" depois de renomear) | Clean |
| `.\mvnw dependency:tree` | árvore de dependências (descobrir conflito de versão) | — |

**Adicionar biblioteca no backend:** não tem comando — edite o `pom.xml` (dentro de `<dependencies>`) e rode `.\mvnw compile` (ou clique em *Reload Maven* no IntelliJ). Exemplos que vamos usar:

| Lib | `artifactId` | Etapa |
|---|---|---|
| Spring Security | `spring-boot-starter-security` | login |
| Apache Commons CSV | `commons-csv` (group `org.apache.commons`) | importação |

---

## 4. Testar a API e o WhatsApp

| Comando | Para quê |
|---|---|
| `Invoke-RestMethod http://localhost:8080/api/ping` | GET rápido sem navegador |
| `Invoke-RestMethod -Method Post http://localhost:8080/api/... -ContentType "application/json" -Body '{"texto":"oi"}'` | POST com JSON |
| arquivo `backend/http/*.http` | requisições prontas pelo IntelliJ (botão ▶ ao lado de cada uma) |
| `cloudflared tunnel --url http://localhost:8080` | URL HTTPS pública para o webhook da Meta (muda a cada execução) |

---

## 5. Git

| Comando | Para quê |
|---|---|
| `git status` | o que mudou — **confira que `application-dev.properties` não aparece** |
| `git add .` | prepara tudo para commit |
| `git commit -m "feat: ..."` | grava (prefixos: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`) |
| `git push` | envia ao GitHub |
| `git pull` | traz do GitHub (outro PC) |
| `git log --oneline -10` | últimos commits |
| `git diff` | mudanças ainda não preparadas |
| `git restore <arquivo>` | descarta alterações de um arquivo (⚠️ perde o que não foi commitado) |
| `git switch -c feat/inbox` | cria e entra num branch |
| `git clone https://github.com/SEU_USUARIO/busca-ativa-crm.git` | baixar o projeto em outro PC |

**Depois de clonar em outro PC:**
1. recriar `backend/src/main/resources/application-dev.properties` à mão;
2. `cd frontend` → `npm ci`.

---

## 6. Socorro

| Problema | Comando |
|---|---|
| "Port 8080 was already in use" | `Get-NetTCPConnection -LocalPort 8080 \| Select-Object OwningProcess` → `Stop-Process -Id <número>` |
| "Port 4200 is already in use" | mesmo comando com `4200`, ou `ng serve --port 4300` |
| Angular com erro estranho depois de atualizar libs | `Remove-Item -Recurse -Force node_modules, .angular` → `npm install` |
| Backend com classe antiga/fantasma | `.\mvnw clean` e rodar de novo |
| Não conecta no Supabase depois de dias parado | painel do Supabase → **Restore** (plano Free pausa) |
| Envio do WhatsApp com `401` | token temporário venceu (24h) → gerar outro em *WhatsApp → API Setup* |

---

## 7. Rotina de teste antes de cada commit

```powershell
cd backend; .\mvnw test; cd ..
```

```powershell
cd frontend; ng test --watch=false; ng build; cd ..
```

```powershell
git status
```

Tudo verde e nenhum arquivo de credencial na lista → `git add .` → `git commit` → `git push`.
