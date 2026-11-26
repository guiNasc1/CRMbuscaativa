# Guia 01 — Setup do Busca Ativa CRM (do zero até o WhatsApp conversando com o sistema)

> **Objetivo:** terminar este guia com backend (Spring Boot) + frontend (Angular) rodando, banco no Supabase, código no GitHub e **uma mensagem de WhatsApp do responsável chegando no log do backend**.
> Cada etapa termina com um ✅ **checkpoint** e um **commit**.

---

## 0. O que muda em relação ao Busca Ativa original

No Busca Ativa o WhatsApp era **semi-automático** (`wa.me`): o sistema só abria o WhatsApp do funcionário. Ele **nunca via a resposta**.
Um **CRM** precisa de conversa **nos dois sentidos** dentro do sistema → usamos a **WhatsApp Business Cloud API** (oficial da Meta).

```
                         ENVIO                                         RECEBIMENTO
┌───────────┐ HTTP ┌──────────────┐ HTTPS ┌──────────────┐      ┌──────────────┐ webhook ┌──────────────┐
│ Angular   │ ───► │ Spring Boot  │ ────► │ Meta Graph   │ ───► │ WhatsApp do  │ ──────► │ Spring Boot  │
│ :4200     │      │ :8080        │       │ API (nuvem)  │      │ responsável  │  (POST) │ /api/webhook │
└───────────┘      └──────┬───────┘       └──────────────┘      └──────────────┘         └──────┬───────┘
                          │ JDBC                                                                │
                          ▼                                                                     ▼
                    ┌──────────┐                                                          grava a mensagem
                    │ Supabase │ ◄────────────────────────────────────────────────────────────────┘
                    └──────────┘
```

| Opção | Envia | Recebe resposta no sistema | Custo | Decisão |
|---|---|---|---|---|
| `wa.me` (link) | ✔ (manual) | ✘ | grátis | fallback |
| **Cloud API da Meta** | ✔ | ✔ (webhook) | grátis no teste; template pago por mensagem em produção | **usar** |
| Twilio | ✔ | ✔ | Meta + taxa Twilio | desnecessário |
| Libs não-oficiais | ✔ | ✔ | grátis | ✘ risco de banir o número |

**Duas regras da Meta que moldam o CRM:**
1. **Janela de 24h:** depois que o responsável manda mensagem, a escola pode responder com texto livre por 24h.
2. **Fora da janela**, a escola só pode **iniciar** conversa com um **template aprovado** (ex.: "Olá, {{1}} faltou hoje…"). É aí que existe cobrança por mensagem — confira a tabela de preços da Meta para o Brasil antes de ir para produção.

---

## 1. Instalar as ferramentas

### 1.1 O que você já tem (conferido na sua máquina ✔)

| Ferramenta | Versão | Para quê |
|---|---|---|
| JDK Zulu | 21.0.9 | compilar/rodar o Spring |
| Maven | 3.9.15 | opcional — o projeto traz o `mvnw` (Maven Wrapper) |
| Node / npm | 24.19 / 11.10 | rodar o Angular |
| Angular CLI | 22.2.2 | `ng new`, `ng serve`, `ng generate` |
| Git | 2.54 | versionamento |
| IntelliJ IDEA | 2025.3 | backend |
| VS Code | — | frontend |
| Docker | 29.9 | não precisa agora |

**Falta instalar:**

```powershell
winget install Cloudflare.cloudflared
```

```powershell
winget install GitHub.cli
```

- `cloudflared` → cria um **túnel HTTPS público** para o seu `localhost:8080` (a Meta precisa de uma URL pública para o webhook).
- `gh` → opcional, cria o repositório no GitHub pela linha de comando.

Feche e abra o terminal depois de instalar (para o PATH atualizar).

### 1.2 Em um PC novo (referência)

```powershell
winget install Azul.Zulu.21.JDK
```

```powershell
winget install OpenJS.NodeJS.LTS
```

```powershell
winget install Git.Git
```

```powershell
winget install Microsoft.VisualStudioCode
```

```powershell
npm install -g @angular/cli
```

IntelliJ: baixar em jetbrains.com/idea. Maven não precisa (o `mvnw` baixa sozinho).

**Problemas comuns no Windows:**

| Sintoma | Solução |
|---|---|
| `ng : o arquivo ng.ps1 não pode ser carregado` | `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` |
| `mvnw` diz que não acha o Java | conferir `java -version` e `$env:JAVA_HOME` (deve apontar para a pasta do JDK, sem `\bin`) |

### 1.3 Extensões das IDEs

- **VS Code:** *Angular Language Service*.
- **IntelliJ:** Lombok já vem embutido. Se aparecer `getX()` em vermelho: *Settings → Build → Compiler → Annotation Processors → Enable annotation processing*.

✅ **Checkpoint:** `cloudflared --version` e `gh --version` respondem.

---

## 2. Git + GitHub

### 2.1 Configurar o Git (só na primeira vez no PC)

```powershell
git config --global user.name "Seu Nome"
```

```powershell
git config --global user.email "seu-email-do-github"
```

### 2.2 Criar o repositório local

Na pasta `buscaativaCRM/`:

```powershell
git init -b main
```

Crie `buscaativaCRM/.gitignore`:

```gitignore
# ---------- Backend ----------
backend/target/
# credenciais (banco + token do WhatsApp) — NUNCA versionar
backend/src/main/resources/application-dev.properties

# ---------- Frontend ----------
frontend/node_modules/
frontend/dist/
frontend/.angular/

# ---------- IDEs / SO ----------
.idea/
*.iml
.vscode/
Thumbs.db
.DS_Store
```

Crie `buscaativaCRM/README.md`:

```markdown
# Busca Ativa CRM
Contato direto funcionário ↔ responsável pelo WhatsApp. Backend Spring Boot (`backend/`) + frontend Angular (`frontend/`).
```

```powershell
git add .
```

```powershell
git commit -m "chore: estrutura inicial do monorepo"
```

### 2.3 Subir no GitHub

**Opção A — pelo `gh` (mais rápido):**

```powershell
gh auth login
```

(escolha GitHub.com → HTTPS → login pelo navegador)

```powershell
gh repo create busca-ativa-crm --private --source . --push
```

**Opção B — pelo site:** github.com → *New repository* → nome `busca-ativa-crm`, **Private**, **sem** README/.gitignore → depois:

```powershell
git remote add origin https://github.com/SEU_USUARIO/busca-ativa-crm.git
```

```powershell
git push -u origin main
```

✅ **Checkpoint:** o repositório privado aparece no GitHub com o README.

---

## 3. Banco no Supabase

### 3.1 Criar o projeto

1. supabase.com → **Sign in with GitHub**.
2. Crie uma **organization** (plano **Free**).
3. **New project**:
   - Name: `busca-ativa-crm`
   - Database password: clique **Generate** e **guarde num gerenciador de senhas** (só aparece agora).
   - Region: **South America (São Paulo)**.
4. Aguarde ~2 min até o projeto ficar verde.

### 3.2 Pegar os dados de conexão

Botão **Connect** (topo do painel) → aba de connection string → tipo **JDBC** → método **Session pooler**.

| Campo | Formato |
|---|---|
| Host | `aws-X-sa-east-1.pooler.supabase.com` |
| Porta | `5432` |
| Usuário | `postgres.<id-do-projeto>` |
| Banco | `postgres` |

> **Por que Session pooler?** A conexão "Direct" do Supabase é só IPv6, e muita rede doméstica no Brasil é só IPv4. O pooler funciona nas duas.

> ⚠️ **Plano Free pausa o projeto após ~7 dias sem uso.** Se o backend não conectar depois de um tempo parado, entre no painel e clique **Restore**.

Guarde esses valores — vão no `application-dev.properties` (passo 4.3).

✅ **Checkpoint:** você tem host, usuário e senha anotados.

---

## 4. Backend — Spring Boot

### 4.1 Gerar o projeto

Na raiz `buscaativaCRM/`:

```powershell
curl.exe -G https://start.spring.io/starter.zip -d type=maven-project -d language=java -d javaVersion=21 -d groupId=com.escola -d artifactId=buscaativacrm -d name=buscaativacrm -d packageName=com.escola.buscaativacrm -d packaging=jar -d dependencies=web,data-jpa,postgresql,validation,devtools,lombok -d baseDir=backend -o backend.zip
```

```powershell
Expand-Archive backend.zip -DestinationPath . -Force; Remove-Item backend.zip
```

| Dependência | Para quê | Equivalente no JSF |
|---|---|---|
| Spring Web | API REST + Tomcat embutido (e o webhook) | Tomcat + Servlet + JAX-RS |
| Spring Data JPA | Hibernate + repositórios prontos | Hibernate + seu `AbstractFacade` |
| PostgreSQL Driver | driver JDBC | o mesmo |
| Validation | `@NotBlank`, `@Size`… | o mesmo Bean Validation |
| DevTools | reinicia ao salvar | — |
| Lombok | getters/setters/construtores na compilação | o mesmo |

> Para chamar a API da Meta não precisa de dependência extra: o Spring já traz o `RestClient`.

Estrutura gerada:

```
backend/
├── mvnw, mvnw.cmd       ← Maven Wrapper (mesma versão do Maven em qualquer PC)
├── pom.xml
└── src/main/
    ├── java/com/escola/buscaativacrm/BuscaativacrmApplication.java
    └── resources/application.properties
```

> ⚠️ Toda classe tem que ficar dentro de `com.escola.buscaativacrm` (ou subpacotes) — é dali que o `@SpringBootApplication` faz o *component scan* (como o `beans.xml` do CDI).

### 4.2 `application.properties` (vai para o Git — sem segredo)

```properties
spring.application.name=buscaativacrm

# carrega também o application-dev.properties
spring.profiles.active=dev

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false
```

### 4.3 `application-dev.properties` (NÃO vai para o Git)

Crie `backend/src/main/resources/application-dev.properties`:

```properties
# ---- Supabase (passo 3.2) ----
spring.datasource.url=jdbc:postgresql://SEU_HOST:5432/postgres?sslmode=require
spring.datasource.username=postgres.SEU_ID
spring.datasource.password=SUA_SENHA

# ---- WhatsApp Cloud API (preenchido no passo 7) ----
whatsapp.token=
whatsapp.phone-number-id=
# senha que VOCÊ inventa; a Meta repete ela ao validar o webhook
whatsapp.verify-token=troque-por-uma-frase-qualquer
```

> Rode `git status` agora: `application-dev.properties` **não pode** aparecer. Se aparecer, o `.gitignore` está errado — corrija antes de qualquer commit.

### 4.4 Endpoint de teste

`backend/src/main/java/com/escola/buscaativacrm/controller/PingController.java`:

```java
package com.escola.buscaativacrm.controller;

import java.util.Map;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@RestController              // responde HTTP; o retorno vira JSON
@RequestMapping("/api")
public class PingController {

    @GetMapping("/ping")     // GET /api/ping
    public Map<String, String> ping() {
        log.info("Ping recebido");
        return Map.of("status", "ok");
    }
}
```

### 4.5 Rodar

```powershell
cd backend
```

```powershell
.\mvnw spring-boot:run
```

(ou no IntelliJ: *Open* → pasta `backend/` → rodar `BuscaativacrmApplication`)

No log, procure:
- `HikariPool-1 - Start completed` → **conectou no Supabase**;
- `Tomcat started on port 8080`.

Abra http://localhost:8080/api/ping → `{"status":"ok"}`.

✅ **Checkpoint:** JSON no navegador + `Ping recebido` no console.

```powershell
git add backend
```

```powershell
git commit -m "feat(backend): projeto Spring Boot com /api/ping"
```

---

## 5. Frontend — Angular

### 5.1 Gerar o projeto

Na raiz `buscaativaCRM/`:

```powershell
ng new buscaativacrm-web --directory frontend --skip-git --style=css --ssr=false
```

- `--skip-git` → **importante**: o repositório é a raiz, não crie outro dentro de `frontend/`.
- Perguntas extras (ferramentas de IA, zoneless etc.): aceite o padrão.

### 5.2 Proxy (evita CORS em desenvolvimento)

`frontend/proxy.conf.json`:

```json
{
  "/api": {
    "target": "http://localhost:8080",
    "secure": false
  }
}
```

No `frontend/angular.json`, dentro de `"serve"`:

```json
"serve": {
  "builder": "...(deixe o que já está)...",
  "options": {
    "proxyConfig": "proxy.conf.json"
  }
}
```

### 5.3 HttpClient + chamada ao backend

`frontend/src/app/app.config.ts` — adicione aos `providers`:

```ts
import { provideHttpClient } from '@angular/common/http';
// ...
providers: [
  // ...os que já existem...
  provideHttpClient()
]
```

`frontend/src/app/app.ts`:

```ts
import { Component, inject, signal } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet],
  templateUrl: './app.html',
  styleUrl: './app.css'
})
export class App {
  private http = inject(HttpClient);        // ≈ @Inject do CDI
  status = signal('carregando...');         // valor reativo: mudou, a tela atualiza

  constructor() {
    this.http.get<{ status: string }>('/api/ping')
      .subscribe(r => this.status.set(r.status));
  }
}
```

`frontend/src/app/app.html` (substitua tudo):

```html
<h1>Busca Ativa CRM</h1>
<p>Backend respondeu: <strong>{{ status() }}</strong></p>
<router-outlet />
```

### 5.4 Rodar

Com o backend rodando, em **outro terminal**:

```powershell
cd frontend
```

```powershell
ng serve
```

Abra http://localhost:4200 → **"Backend respondeu: ok"**.

✅ **Checkpoint:** a frase aparece.

```powershell
git add .
```

```powershell
git commit -m "feat(frontend): Angular chamando /api/ping via proxy"
```

```powershell
git push
```

---

## 6. Compilar para produção (para saber como é)

| | Comando (na pasta do projeto) | Resultado |
|---|---|---|
| Backend | `.\mvnw clean package` | `backend/target/buscaativacrm-0.0.1-SNAPSHOT.jar` |
| Rodar o JAR | `java -jar target\buscaativacrm-0.0.1-SNAPSHOT.jar` | o mesmo backend, sem IDE nem Maven |
| Frontend | `ng build` | `frontend/dist/buscaativacrm-web/browser/` (HTML/JS/CSS estáticos) |

> O `package` roda os testes, e o teste padrão (`contextLoads`) sobe a aplicação inteira — precisa do banco acessível. Para pular: `.\mvnw clean package -DskipTests`.

---

## 7. WhatsApp Cloud API (o coração do CRM)

### 7.1 Criar o app na Meta

1. developers.facebook.com → login com sua conta do Facebook → **My Apps → Create App**.
2. Caso de uso: **Connect with customers through WhatsApp** (ou tipo **Business**, dependendo da tela).
3. Vincule/crie um **Business portfolio** (pode ser "Colégio XYZ — teste").
4. No app criado → **WhatsApp → API Setup**. Ali estão:

| Campo da tela | Vai para |
|---|---|
| **Temporary access token** (vale 24h) | `whatsapp.token` |
| **Phone number ID** (do número de teste) | `whatsapp.phone-number-id` |
| WhatsApp Business Account ID | anote; usado para templates |

5. Em **To**, clique **Manage phone number list** e adicione **o seu celular** (recebe um código pelo WhatsApp). O número de teste só envia para até 5 números cadastrados assim.

Preencha `whatsapp.token` e `whatsapp.phone-number-id` no `application-dev.properties`.

### 7.2 Teste de envio (sem código)

Na própria tela **API Setup**, clique **Send message** → chega o template `hello_world` no seu celular.

Ou pelo PowerShell (troque os valores; use a versão `vXX.0` que aparece no exemplo da própria tela):

```powershell
$token = "SEU_TOKEN"; $phoneId = "SEU_PHONE_NUMBER_ID"
$body = @{ messaging_product = "whatsapp"; to = "5544999998888"; type = "template"; template = @{ name = "hello_world"; language = @{ code = "en_US" } } } | ConvertTo-Json -Depth 5
Invoke-RestMethod -Method Post -Uri "https://graph.facebook.com/vXX.0/$phoneId/messages" -Headers @{ Authorization = "Bearer $token" } -ContentType "application/json" -Body $body
```

✅ **Checkpoint:** o "Hello World" chegou no seu WhatsApp.

### 7.3 Webhook: receber as respostas no Spring

A Meta faz **duas** coisas no seu endpoint:

| Método | Quando | O que o Spring devolve |
|---|---|---|
| `GET` | uma vez, ao salvar a URL no painel | o `hub.challenge`, se o `hub.verify_token` bater |
| `POST` | a cada evento (mensagem recebida, entregue, lida…) | `200 OK` rápido (senão a Meta reenvia) |

`backend/src/main/java/com/escola/buscaativacrm/controller/WhatsAppWebhookController.java`:

```java
package com.escola.buscaativacrm.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@RestController
@RequestMapping("/api/webhook/whatsapp")
public class WhatsAppWebhookController {

    @Value("${whatsapp.verify-token}")       // lê do application-dev.properties
    private String verifyToken;

    // 1) Verificação: a Meta chama UMA vez quando você salva o webhook no painel
    @GetMapping
    public ResponseEntity<String> verificar(@RequestParam("hub.mode") String mode,
                                            @RequestParam("hub.verify_token") String token,
                                            @RequestParam("hub.challenge") String challenge) {
        if ("subscribe".equals(mode) && verifyToken.equals(token)) {
            return ResponseEntity.ok(challenge);
        }
        return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
    }

    // 2) Eventos: por enquanto só loga o JSON cru para você ver o formato
    @PostMapping
    public ResponseEntity<Void> receber(@RequestBody String payload) {
        log.info("Webhook WhatsApp: {}", payload);
        return ResponseEntity.ok().build();
    }
}
```

> Paralelo JSF: é um *servlet* que um sistema externo chama — não tem tela, não tem usuário logado.

> ⚠️ **Antes de gravar qualquer coisa no banco a partir deste `POST`**, valide o header `X-Hub-Signature-256` (HMAC-SHA256 do corpo com o **App Secret** do app). Sem isso, qualquer um que descubra a URL injeta mensagens falsas no CRM. Fica para o Guia 02.

### 7.4 Expor o localhost para a Meta (túnel)

Com o backend rodando, em outro terminal:

```powershell
cloudflared tunnel --url http://localhost:8080
```

Ele imprime algo como `https://palavras-aleatorias.trycloudflare.com`. Teste no navegador: `https://…trycloudflare.com/api/ping` → `{"status":"ok"}`.

> A URL **muda toda vez** que você reinicia o `cloudflared` → precisa atualizar no painel da Meta. Se isso incomodar: o **ngrok** (conta grátis) dá um domínio fixo.

### 7.5 Registrar o webhook na Meta

App → **WhatsApp → Configuration → Webhook → Edit**:

| Campo | Valor |
|---|---|
| Callback URL | `https://…trycloudflare.com/api/webhook/whatsapp` |
| Verify token | o mesmo texto de `whatsapp.verify-token` |

**Verify and save** → depois, em **Webhook fields**, assine (**Subscribe**) o campo **`messages`**.

### 7.6 Teste ponta a ponta

1. Do seu celular, **responda** a mensagem "Hello World" que recebeu do número de teste (ex.: "Meu filho estava doente").
2. Olhe o console do backend: aparece `Webhook WhatsApp: {"object":"whatsapp_business_account","entry":[…"text":{"body":"Meu filho estava doente"}…]}`.

✅ **Checkpoint:** a resposta do "responsável" apareceu no log do Spring. **O CRM está ligado ao WhatsApp.**

```powershell
git add .
```

```powershell
git commit -m "feat(backend): webhook do WhatsApp Cloud API (verificação + log)"
```

```powershell
git push
```

**Se não funcionar:**

| Sintoma | Causa provável |
|---|---|
| "The callback URL or verify token couldn't be validated" | backend ou túnel parados; verify token diferente; URL sem `/api/webhook/whatsapp` |
| Envio retorna `401` / "Session has expired" | token temporário venceu (24h) → gerar outro na tela API Setup |
| Envio retorna erro "recipient not in allowed list" | seu número não foi adicionado em **To → Manage phone number list** |
| Mensagem respondida mas nada no log | campo `messages` não está com **Subscribe**; use o botão **Test** ao lado dele para disparar um evento de exemplo |

---

## 8. Rotina do dia a dia

| Terminal | Pasta | Comando | Precisa sempre? |
|---|---|---|---|
| 1 | `backend/` | `.\mvnw spring-boot:run` (ou IntelliJ) | sim |
| 2 | `frontend/` | `ng serve` | sim |
| 3 | qualquer | `cloudflared tunnel --url http://localhost:8080` | só ao testar recebimento de WhatsApp (e atualizar a URL na Meta) |

Em outro PC: clone o repo, **recrie o `application-dev.properties`** à mão, e rode `npm install` dentro de `frontend/`.

---

## 9. Antes de produção (anotar, não fazer agora)

- **Token permanente:** Business Settings → *System users* → gerar token com `whatsapp_business_messaging` e `whatsapp_business_management` (o temporário vence em 24h).
- **Número real da escola:** adicionar e verificar em *WhatsApp → API Setup → Add phone number*. Um número registrado na API em geral sai do app WhatsApp comum — verifique a opção de **coexistência** com o WhatsApp Business App antes de migrar o número que a secretaria já usa.
- **Templates** para iniciar conversa (categoria *Utility*, ex.: aviso de falta) precisam ser aprovados pela Meta.
- **Verificação do negócio** (Business verification) na Meta para sair dos limites de teste.
- Banco de produção separado, `ddl-auto=validate`, HTTPS real (o webhook exige).

---

## 10. Resumo

| Etapa | Resultado |
|---|---|
| 1 | Ferramentas instaladas (+ `cloudflared`, `gh`) |
| 2 | Repositório privado no GitHub, credenciais fora do Git |
| 3 | Postgres no Supabase (São Paulo, Session pooler) |
| 4 | Spring Boot conectado ao banco, `GET /api/ping` |
| 5 | Angular chamando o Spring via proxy |
| 6 | Sabe gerar o JAR e o build do Angular |
| 7 | Envia template e **recebe a resposta do responsável no backend** |

## Próximo: Guia 02 — Modelo do CRM

```
Responsavel (telefone 55DDDNUMERO, nome) ──< Aluno >── Turma
Responsavel ──< Conversa ──< Mensagem (direcao: ENVIADA/RECEBIDA, texto, wamid, status, dataHora)
```

1. Validar a assinatura `X-Hub-Signature-256` e transformar o JSON do webhook em `Mensagem` no banco.
2. `WhatsAppService.enviarTexto(telefone, texto)` com `RestClient`.
3. Tela Angular de **caixa de entrada**: lista de conversas + chat (envio e resposta).
4. Depois: login (Spring Security/BCrypt), atualização em tempo real (SSE), templates de falta.
