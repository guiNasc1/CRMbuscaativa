# Guia 02 — Caixa de entrada do CRM (receber e responder pelo WhatsApp)

> **Objetivo:** o responsável manda mensagem no WhatsApp → ela aparece numa **caixa de entrada** no Angular → o funcionário responde pela tela → a resposta chega no celular do responsável.
> **Pré-requisito:** Guia 01 completo (ping funcionando, app da Meta criado, webhook verificado, `cloudflared` instalado).
> Cada etapa termina com um ✅ **checkpoint**; os commits ficam nos pontos marcados.

---

## 0. O caminho de uma mensagem

São **dois fluxos**, e só um deles começa no Angular:

```
RECEBER (a Meta chama a gente)
 WhatsApp do      Meta          POST /api/webhook/whatsapp        ┌───────────────┐   ┌─────────┐
 responsável ──► (nuvem) ──────────────────────────────────────►  │ WebhookCtrl   │ → │ banco   │
                                (assinado com o App Secret)       │ → Conversa-   │   │ mensagem│
                                                                  │   Service     │   └─────────┘
                                                                  └───────────────┘        ▲
RESPONDER (o funcionário chama a gente)                                                     │
 ┌────────┐  POST /api/conversas/{id}/mensagens  ┌──────────────┐   ┌──────────────┐        │
 │ Angular│ ───────────────────────────────────► │ ConversaCtrl │ → │ Conversa-    │ ───────┘
 │ (chat) │ ◄─── a cada 5s: GET (polling) ────── │              │   │ Service      │ ──► WhatsAppClient ──► Meta ──► celular
 └────────┘                                      └──────────────┘   └──────────────┘
```

| Peça nova | Para quê | No JSF seria… |
|---|---|---|
| **Webhook** | a Meta chama o seu sistema quando chega mensagem | um *servlet* chamado por sistema externo |
| **Assinatura HMAC** | provar que quem chamou o webhook foi a Meta | (não tinha) |
| **`RestClient`** | o seu sistema chama a API da Meta | `HttpURLConnection` / JAX-RS Client |
| **`@ConfigurationProperties`** | agrupar as configs `whatsapp.*` num `record` | `@ConfigProperty` do MicroProfile / `context-param` |
| **Polling** | a tela pergunta "tem novidade?" a cada 5s | `<p:poll interval="5">` |

**Sem entidade `Conversa`:** uma conversa é simplesmente **um responsável + as mensagens dele**. Se um dia precisar de "conversa finalizada" ou "atribuída a funcionário", a entidade nasce aí.

Pacotes no fim deste guia:

```
com.escola.buscaativacrm
├── controller/  PingController, ConversaController, WhatsAppWebhookController
├── dto/         ConversaResponse, MensagemResponse, EnviarMensagemRequest
├── exception/   NaoEncontradoException, RegraNegocioException, GlobalExceptionHandler
├── model/       Responsavel, Mensagem, Direcao, StatusMensagem
├── repository/  ResponsavelRepository, MensagemRepository
├── service/     ConversaService
└── whatsapp/    WhatsAppProperties, WhatsAppClient, WebhookPayload, AssinaturaMeta, WhatsAppWebhookService
```

> `whatsapp/` é o **único pacote que conhece o formato da Meta**. Se um dia trocar de provedor (ex.: Twilio), só ele muda — é a decisão de "isolar o WhatsApp" do projeto original.

---

# PARTE A — Backend

## A1. Enums

`model/Direcao.java`:

```java
package com.escola.buscaativacrm.model;

public enum Direcao {
    ENTRADA,   // responsável → escola
    SAIDA      // escola → responsável
}
```

`model/StatusMensagem.java`:

```java
package com.escola.buscaativacrm.model;

// A ORDEM IMPORTA: o status só avança (ver ConversaService.atualizarStatus)
public enum StatusMensagem {
    ENVIADA, ENTREGUE, LIDA, FALHOU
}
```

## A2. Entidades — primeiro relacionamento

`model/Responsavel.java`:

```java
package com.escola.buscaativacrm.model;

import java.time.Duration;
import java.time.Instant;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "responsavel")
@Getter
@Setter
@NoArgsConstructor
public class Responsavel {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String nome;

    @Column(nullable = false, unique = true, length = 20)
    private String telefone;            // formato do WhatsApp (wa_id): só dígitos com DDI, ex.: 5544999998888

    private Instant ultimaMensagemEm;   // ordena a caixa de entrada
    private Instant ultimaRecebidaEm;   // última mensagem DO responsável → abre a janela de 24h

    public boolean janelaAberta() {
        return ultimaRecebidaEm != null
                && ultimaRecebidaEm.isAfter(Instant.now().minus(Duration.ofHours(24)));
    }
}
```

`model/Mensagem.java`:

```java
package com.escola.buscaativacrm.model;

import java.time.Instant;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "mensagem")
@Getter
@Setter
@NoArgsConstructor
public class Mensagem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)   // várias mensagens → um responsável
    @JoinColumn(name = "responsavel_id")                   // coluna FK na tabela mensagem
    private Responsavel responsavel;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 10)
    private Direcao direcao;

    @Column(nullable = false, columnDefinition = "text")   // sem limite de tamanho
    private String texto;

    @Column(unique = true, length = 100)
    private String wamid;              // id da mensagem na Meta ("wamid.HBgM...")

    @Enumerated(EnumType.STRING)
    @Column(length = 10)
    private StatusMensagem status;     // só nas de SAIDA; null nas de ENTRADA

    @Column(nullable = false)
    private Instant dataHora;
}
```

Pontos para fixar:

| Escolha | Por quê |
|---|---|
| `Instant` (não `LocalDateTime`) | é um **instante absoluto em UTC**; o Postgres grava como `timestamp with time zone` e o Angular converte para o fuso do navegador |
| `FetchType.LAZY` | o padrão do `@ManyToOne` é `EAGER` (carrega o responsável junto **sempre**). LAZY só busca se você acessar — regra de ouro: **sempre LAZY** |
| **Sem** `@OneToMany List<Mensagem>` no `Responsavel` | relacionamento **unidirecional**: as mensagens vêm do repository, já ordenadas. Menos chance de `LazyInitializationException` e de carregar 10 mil mensagens sem querer |
| `wamid` `unique` | a Meta **reenvia** o mesmo evento se demorarmos a responder; o `wamid` repetido é como detectamos |

> ⚠️ **Pegadinha do 9º dígito (Brasil):** para muitos celulares, o WhatsApp manda o `wa_id` **sem o 9** (ex.: `554499998888` em vez de `5544999998888`). Como neste guia o responsável é criado **a partir da mensagem recebida**, guardamos exatamente o que a Meta mandou e tudo funciona. O problema aparece quando o admin cadastrar telefones à mão (Guia 03) — é um dos exercícios.

✅ **Checkpoint:** rode o backend. No log aparecem `create table responsavel` e `create table mensagem` (com `foreign key (responsavel_id)`). Confira no **Supabase → Table Editor**.

```powershell
git add backend
```

```powershell
git commit -m "feat(backend): entidades Responsavel e Mensagem"
```

## A3. Repositories

`repository/ResponsavelRepository.java`:

```java
package com.escola.buscaativacrm.repository;

import java.util.List;
import java.util.Optional;

import org.springframework.data.jpa.repository.JpaRepository;

import com.escola.buscaativacrm.model.Responsavel;

public interface ResponsavelRepository extends JpaRepository<Responsavel, Long> {

    Optional<Responsavel> findByTelefone(String telefone);

    List<Responsavel> findByUltimaMensagemEmNotNullOrderByUltimaMensagemEmDesc();
}
```

`repository/MensagemRepository.java`:

```java
package com.escola.buscaativacrm.repository;

import java.util.List;
import java.util.Optional;

import org.springframework.data.jpa.repository.JpaRepository;

import com.escola.buscaativacrm.model.Mensagem;

public interface MensagemRepository extends JpaRepository<Mensagem, Long> {

    boolean existsByWamid(String wamid);

    Optional<Mensagem> findByWamid(String wamid);

    List<Mensagem> findByResponsavelIdOrderByDataHoraAsc(Long responsavelId);
}
```

| Método | JPQL gerado (aproximado) |
|---|---|
| `findByUltimaMensagemEmNotNullOrderByUltimaMensagemEmDesc()` | `... where r.ultimaMensagemEm is not null order by r.ultimaMensagemEm desc` |
| `existsByWamid(w)` | `select count(m) > 0 ... where m.wamid = :w` |
| `findByResponsavelIdOrderByDataHoraAsc(id)` | `... where m.responsavel.id = :id order by m.dataHora asc` ← **navega** o relacionamento |

## A4. DTOs

`dto/ConversaResponse.java`:

```java
package com.escola.buscaativacrm.dto;

import java.time.Instant;

import com.escola.buscaativacrm.model.Responsavel;

public record ConversaResponse(
        Long responsavelId,
        String nome,
        String telefone,
        Instant ultimaMensagemEm,
        boolean janelaAberta) {

    public static ConversaResponse from(Responsavel r) {
        return new ConversaResponse(r.getId(), r.getNome(), r.getTelefone(),
                r.getUltimaMensagemEm(), r.janelaAberta());
    }
}
```

`dto/MensagemResponse.java`:

```java
package com.escola.buscaativacrm.dto;

import java.time.Instant;

import com.escola.buscaativacrm.model.Direcao;
import com.escola.buscaativacrm.model.Mensagem;
import com.escola.buscaativacrm.model.StatusMensagem;

public record MensagemResponse(
        Long id,
        Direcao direcao,
        String texto,
        StatusMensagem status,
        Instant dataHora) {

    public static MensagemResponse from(Mensagem m) {
        return new MensagemResponse(m.getId(), m.getDirecao(), m.getTexto(), m.getStatus(), m.getDataHora());
    }
}
```

`dto/EnviarMensagemRequest.java`:

```java
package com.escola.buscaativacrm.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record EnviarMensagemRequest(
        @NotBlank(message = "Digite a mensagem")
        @Size(max = 4096, message = "Máximo de 4096 caracteres")   // limite do WhatsApp
        String texto) {
}
```

> No JSON, `Instant` sai como texto ISO-8601: `"2026-11-25T13:45:00Z"` (o `Z` = UTC).

## A5. Exceções e tratamento global de erros

`exception/NaoEncontradoException.java`:

```java
package com.escola.buscaativacrm.exception;

public class NaoEncontradoException extends RuntimeException {
    public NaoEncontradoException(String mensagem) {
        super(mensagem);
    }
}
```

`exception/RegraNegocioException.java`:

```java
package com.escola.buscaativacrm.exception;

public class RegraNegocioException extends RuntimeException {
    public RegraNegocioException(String mensagem) {
        super(mensagem);
    }
}
```

`exception/GlobalExceptionHandler.java`:

```java
package com.escola.buscaativacrm.exception;

import java.util.LinkedHashMap;
import java.util.Map;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.client.RestClientResponseException;

import lombok.extern.slf4j.Slf4j;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NaoEncontradoException.class)
    public ProblemDetail naoEncontrado(NaoEncontradoException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());        // 404
    }

    @ExceptionHandler(RegraNegocioException.class)
    public ProblemDetail regraNegocio(RegraNegocioException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());         // 409
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail validacao(MethodArgumentNotValidException ex) {
        ProblemDetail problema = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Dados inválidos"); // 400
        Map<String, String> campos = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
                .forEach(erro -> campos.put(erro.getField(), erro.getDefaultMessage()));
        problema.setProperty("campos", campos);
        return problema;
    }

    @ExceptionHandler(RestClientResponseException.class)        // a API da Meta respondeu 4xx/5xx
    public ProblemDetail whatsApp(RestClientResponseException ex) {
        // o corpo do erro da Meta explica o motivo (token vencido, número fora da lista, janela fechada...)
        log.warn("Erro da API do WhatsApp: {} {}", ex.getStatusCode(), ex.getResponseBodyAsString());
        return ProblemDetail.forStatusAndDetail(HttpStatus.BAD_GATEWAY,
                "O WhatsApp recusou a mensagem. Veja o log do backend.");                      // 502
    }
}
```

> **502 Bad Gateway** = "eu estou bem, mas o serviço que eu chamei falhou". É o status certo quando o erro é da Meta, não do nosso sistema nem do usuário.

## A6. Configuração do WhatsApp num `record`

No Guia 01 usamos `@Value` para uma propriedade. Agora são cinco — vale agrupar.

**1)** `application.properties` (vai para o Git) — acrescente:

```properties
# versão da Graph API: copie a que aparece no exemplo da tela "API Setup" da Meta (ex.: v23.0)
whatsapp.api-version=vXX.0
```

**2)** `application-dev.properties` (fora do Git) — acrescente:

```properties
# Meta → seu app → App settings → Basic → App secret → Show
whatsapp.app-secret=
```

**3)** `whatsapp/WhatsAppProperties.java`:

```java
package com.escola.buscaativacrm.whatsapp;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("whatsapp")   // liga todas as chaves "whatsapp.*" aos campos do record
public record WhatsAppProperties(
        String token,
        String phoneNumberId,          // ← whatsapp.phone-number-id (o Spring converte kebab-case → camelCase)
        String verifyToken,
        String appSecret,
        String apiVersion) {
}
```

**4)** Registre na classe principal `BuscaativacrmApplication.java`:

```java
import org.springframework.boot.context.properties.ConfigurationPropertiesScan;

@SpringBootApplication
@ConfigurationPropertiesScan          // procura records/classes com @ConfigurationProperties
public class BuscaativacrmApplication {
    // main() igual
}
```

A partir daqui, qualquer bean pode pedir `WhatsAppProperties` no construtor.

> ⚠️ O **App Secret** é tão sensível quanto a senha do banco: quem tem ele consegue forjar mensagens "da Meta". Só no `application-dev.properties`.

## A7. `WhatsAppClient` — enviar mensagem

`whatsapp/WhatsAppClient.java`:

```java
package com.escola.buscaativacrm.whatsapp;

import java.util.List;
import java.util.Map;

import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

@Component
public class WhatsAppClient {

    private final RestClient http;

    // sem @RequiredArgsConstructor: o campo não é o que vem injetado, é construído a partir dele
    public WhatsAppClient(WhatsAppProperties props) {
        this.http = RestClient.builder()
                .baseUrl("https://graph.facebook.com/" + props.apiVersion() + "/" + props.phoneNumberId())
                .defaultHeader(HttpHeaders.AUTHORIZATION, "Bearer " + props.token())
                .build();
    }

    /** Envia texto livre (só funciona com a janela de 24h aberta). Devolve o wamid. */
    public String enviarTexto(String telefone, String texto) {
        Map<String, Object> corpo = Map.of(
                "messaging_product", "whatsapp",
                "to", telefone,
                "type", "text",
                "text", Map.of("body", texto));

        RespostaEnvio resposta = http.post()
                .uri("/messages")
                .contentType(MediaType.APPLICATION_JSON)
                .body(corpo)                       // Map → JSON (Jackson)
                .retrieve()                        // 4xx/5xx → lança RestClientResponseException (tratada no A5)
                .body(RespostaEnvio.class);        // JSON → record

        return resposta.messages().getFirst().id();
    }

    // só o que usamos da resposta: {"messages":[{"id":"wamid...."}]}
    public record RespostaEnvio(List<Id> messages) {
        public record Id(String id) {}
    }
}
```

Leitura do `RestClient` (API fluente, lê-se de cima para baixo): **verbo → URL → corpo → executa → converte a resposta**. É o mesmo `Invoke-RestMethod` do Guia 01, só que em Java.

## A8. `ConversaService` — as regras

`service/ConversaService.java`:

```java
package com.escola.buscaativacrm.service;

import java.time.Instant;
import java.util.List;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import com.escola.buscaativacrm.dto.ConversaResponse;
import com.escola.buscaativacrm.dto.MensagemResponse;
import com.escola.buscaativacrm.exception.NaoEncontradoException;
import com.escola.buscaativacrm.exception.RegraNegocioException;
import com.escola.buscaativacrm.model.Direcao;
import com.escola.buscaativacrm.model.Mensagem;
import com.escola.buscaativacrm.model.Responsavel;
import com.escola.buscaativacrm.model.StatusMensagem;
import com.escola.buscaativacrm.repository.MensagemRepository;
import com.escola.buscaativacrm.repository.ResponsavelRepository;
import com.escola.buscaativacrm.whatsapp.WhatsAppClient;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ConversaService {

    private final ResponsavelRepository responsaveis;
    private final MensagemRepository mensagens;
    private final WhatsAppClient whatsApp;

    public List<ConversaResponse> listar() {
        return responsaveis.findByUltimaMensagemEmNotNullOrderByUltimaMensagemEmDesc()
                .stream().map(ConversaResponse::from).toList();
    }

    public List<MensagemResponse> mensagens(Long responsavelId) {
        buscarResponsavel(responsavelId);                                  // 404 se não existir
        return mensagens.findByResponsavelIdOrderByDataHoraAsc(responsavelId)
                .stream().map(MensagemResponse::from).toList();
    }

    // shortcut: a chamada HTTP à Meta acontece dentro da transação (segura uma conexão do banco por ~1s);
    // ok para 4–8 funcionários, rever se o volume crescer
    @Transactional
    public MensagemResponse enviar(Long responsavelId, String texto) {
        Responsavel responsavel = buscarResponsavel(responsavelId);
        if (!responsavel.janelaAberta()) {
            throw new RegraNegocioException(
                    "Janela de 24h fechada: o responsável precisa mandar uma mensagem antes (ou use um template).");
        }
        String wamid = whatsApp.enviarTexto(responsavel.getTelefone(), texto);

        Instant agora = Instant.now();
        Mensagem mensagem = novaMensagem(responsavel, Direcao.SAIDA, texto, wamid, agora);
        mensagem.setStatus(StatusMensagem.ENVIADA);
        responsavel.setUltimaMensagemEm(agora);                           // dirty checking: sem save()
        return MensagemResponse.from(mensagens.save(mensagem));
    }

    @Transactional
    public void registrarRecebida(String telefone, String nomePerfil, String wamid, String texto, Instant dataHora) {
        if (mensagens.existsByWamid(wamid)) {
            return;                       // evento repetido: a Meta reenvia se não respondermos 200 a tempo
        }
        Responsavel responsavel = responsaveis.findByTelefone(telefone)
                .orElseGet(() -> novoResponsavel(telefone, nomePerfil));

        mensagens.save(novaMensagem(responsavel, Direcao.ENTRADA, texto, wamid, dataHora));
        responsavel.setUltimaMensagemEm(dataHora);
        responsavel.setUltimaRecebidaEm(dataHora);                       // abre (renova) a janela de 24h
        log.info("Mensagem recebida de {}", telefone);
    }

    @Transactional
    public void atualizarStatus(String wamid, StatusMensagem novo) {
        mensagens.findByWamid(wamid).ifPresent(m -> {
            // a Meta pode entregar "delivered" DEPOIS de "read": o status só avança
            if (m.getStatus() == null || novo.ordinal() > m.getStatus().ordinal()) {
                m.setStatus(novo);
            }
        });
    }

    private Mensagem novaMensagem(Responsavel responsavel, Direcao direcao, String texto, String wamid, Instant dataHora) {
        Mensagem m = new Mensagem();
        m.setResponsavel(responsavel);
        m.setDirecao(direcao);
        m.setTexto(texto);
        m.setWamid(wamid);
        m.setDataHora(dataHora);
        return m;
    }

    private Responsavel novoResponsavel(String telefone, String nomePerfil) {
        Responsavel r = new Responsavel();
        r.setTelefone(telefone);
        r.setNome(nomePerfil != null ? nomePerfil : telefone);   // nome do perfil do WhatsApp; o admin corrige depois
        return responsaveis.save(r);
    }

    private Responsavel buscarResponsavel(Long id) {
        return responsaveis.findById(id)
                .orElseThrow(() -> new NaoEncontradoException("Responsável " + id + " não encontrado"));
    }
}
```

Regras que ficaram **no service** (e não no controller nem no front):

| Regra | Onde |
|---|---|
| Só responde texto livre com janela de 24h aberta | `enviar()` → 409 |
| Mensagem repetida da Meta é ignorada | `registrarRecebida()` → `existsByWamid` |
| Número desconhecido vira responsável novo | `registrarRecebida()` → `orElseGet` |
| Status nunca "volta" | `atualizarStatus()` → `ordinal()` |

## A9. Webhook — receber com segurança

### A9.1 O JSON que a Meta manda

Exemplo (resumido) de uma mensagem recebida — é o que apareceu no seu log no Guia 01:

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "...",
    "changes": [{
      "field": "messages",
      "value": {
        "metadata": { "phone_number_id": "..." },
        "contacts": [{ "profile": { "name": "Maria" }, "wa_id": "554499998888" }],
        "messages": [{
          "from": "554499998888",
          "id": "wamid.HBgM...",
          "timestamp": "1795655524",
          "type": "text",
          "text": { "body": "Meu filho estava doente" }
        }]
      }
    }]
  }]
}
```

Quando é **atualização de status** de uma mensagem que **nós** enviamos, no lugar de `messages` vem `"statuses": [{ "id": "wamid...", "status": "delivered" }]`.

### A9.2 Espelho em `record`s

`whatsapp/WebhookPayload.java`:

```java
package com.escola.buscaativacrm.whatsapp;

import java.util.List;

// Espelho PARCIAL do JSON da Meta. Campos que não estão aqui são ignorados
// (o Spring Boot configura o Jackson para não falhar com campo desconhecido).
public record WebhookPayload(List<Entry> entry) {

    public WebhookPayload {                       // compact constructor: roda antes de atribuir os campos
        entry = entry == null ? List.of() : entry;
    }

    public record Entry(List<Change> changes) {
        public Entry {
            changes = changes == null ? List.of() : changes;
        }
    }

    public record Change(String field, Value value) {}

    public record Value(List<Contact> contacts, List<Message> messages, List<Status> statuses) {
        public Value {                            // lista ausente no JSON vira lista vazia: nada de NullPointerException
            contacts = contacts == null ? List.of() : contacts;
            messages = messages == null ? List.of() : messages;
            statuses = statuses == null ? List.of() : statuses;
        }
    }

    public record Contact(Profile profile) {}

    public record Profile(String name) {}

    public record Message(String from, String id, String timestamp, String type, Text text) {}

    public record Text(String body) {}

    public record Status(String id, String status) {}
}
```

> **Compact constructor** é o construtor de `record` sem a lista de parâmetros: você só "ajusta" os valores, e o Java atribui aos campos no final.

### A9.3 Conferir a assinatura

A Meta calcula `HMAC-SHA256(corpo da requisição, App Secret)` e manda no header `X-Hub-Signature-256: sha256=<hex>`. Nós calculamos o mesmo e comparamos. Sem isso, **qualquer um que descubra a URL do webhook injeta mensagens falsas** no CRM.

`whatsapp/AssinaturaMeta.java`:

```java
package com.escola.buscaativacrm.whatsapp;

import java.nio.charset.StandardCharsets;
import java.security.GeneralSecurityException;
import java.security.MessageDigest;
import java.util.HexFormat;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;

/** Confere o header X-Hub-Signature-256: "sha256=" + HMAC-SHA256(corpo, App Secret) em hexadecimal. */
public final class AssinaturaMeta {

    private AssinaturaMeta() {}                    // classe utilitária: só métodos estáticos

    public static boolean valida(byte[] corpo, String header, String segredo) {
        if (header == null || segredo == null || segredo.isBlank()) {
            return false;
        }
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(segredo.getBytes(StandardCharsets.UTF_8), "HmacSHA256"));
            String esperado = "sha256=" + HexFormat.of().formatHex(mac.doFinal(corpo));
            // comparação em tempo constante: o tempo de resposta não revela quantos caracteres bateram
            return MessageDigest.isEqual(
                    esperado.getBytes(StandardCharsets.UTF_8),
                    header.getBytes(StandardCharsets.UTF_8));
        } catch (GeneralSecurityException e) {
            throw new IllegalStateException("HmacSHA256 indisponível", e);
        }
    }
}
```

**Teste unitário** — primeiro teste do projeto. `backend/src/test/java/com/escola/buscaativacrm/whatsapp/AssinaturaMetaTest.java`:

```java
package com.escola.buscaativacrm.whatsapp;

import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.nio.charset.StandardCharsets;

import org.junit.jupiter.api.Test;

class AssinaturaMetaTest {

    // vetor de teste conhecido do HMAC-SHA256 (chave "key")
    private static final byte[] CORPO =
            "The quick brown fox jumps over the lazy dog".getBytes(StandardCharsets.UTF_8);
    private static final String HEADER =
            "sha256=f7bc83f430538424b13298e6aa6fb143ef4d59a14946175997479dbc2d1a3cd8";

    @Test
    void aceitaAssinaturaCorreta() {
        assertTrue(AssinaturaMeta.valida(CORPO, HEADER, "key"));
    }

    @Test
    void recusaSegredoErrado() {
        assertFalse(AssinaturaMeta.valida(CORPO, HEADER, "outro"));
    }

    @Test
    void recusaSemHeader() {
        assertFalse(AssinaturaMeta.valida(CORPO, null, "key"));
    }
}
```

Na pasta `backend/`:

```powershell
.\mvnw test -Dtest=AssinaturaMetaTest
```

✅ **Checkpoint:** `Tests run: 3, Failures: 0`. (Este teste não sobe o Spring nem o banco — é Java puro, roda em segundos.)

### A9.4 Traduzir o JSON em chamadas ao service

`whatsapp/WhatsAppWebhookService.java`:

```java
package com.escola.buscaativacrm.whatsapp;

import java.time.Instant;

import org.springframework.stereotype.Service;

import com.escola.buscaativacrm.model.StatusMensagem;
import com.escola.buscaativacrm.service.ConversaService;

import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class WhatsAppWebhookService {

    private final ConversaService conversaService;

    public void processar(WebhookPayload payload) {
        for (WebhookPayload.Entry entry : payload.entry()) {
            for (WebhookPayload.Change change : entry.changes()) {
                if ("messages".equals(change.field()) && change.value() != null) {
                    processar(change.value());
                }
            }
        }
    }

    private void processar(WebhookPayload.Value value) {
        String nome = nomeDoPerfil(value);

        for (WebhookPayload.Message msg : value.messages()) {
            String texto = "text".equals(msg.type()) && msg.text() != null
                    ? msg.text().body()
                    : "[mensagem do tipo " + msg.type() + " ainda não suportada]";
            Instant dataHora = Instant.ofEpochSecond(Long.parseLong(msg.timestamp()));  // segundos desde 1970
            conversaService.registrarRecebida(msg.from(), nome, msg.id(), texto, dataHora);
        }

        for (WebhookPayload.Status st : value.statuses()) {
            StatusMensagem status = converter(st.status());
            if (status != null) {
                conversaService.atualizarStatus(st.id(), status);
            }
        }
    }

    private static String nomeDoPerfil(WebhookPayload.Value value) {
        if (value.contacts().isEmpty() || value.contacts().getFirst().profile() == null) {
            return null;
        }
        return value.contacts().getFirst().profile().name();
    }

    private static StatusMensagem converter(String status) {
        return switch (status) {               // switch de Java 21: aceita "case null"
            case "sent" -> StatusMensagem.ENVIADA;
            case "delivered" -> StatusMensagem.ENTREGUE;
            case "read" -> StatusMensagem.LIDA;
            case "failed" -> StatusMensagem.FALHOU;
            case null, default -> null;        // status que não usamos
        };
    }
}
```

> Cada chamada a `conversaService.xxx()` vem de **outro bean**, então passa pelo proxy do Spring e ganha **sua própria transação** (`@Transactional`). Se uma mensagem falhar, as anteriores já estão gravadas.

### A9.5 Controller do webhook (substitui o do Guia 01)

`controller/WhatsAppWebhookController.java` — troque o arquivo inteiro:

```java
package com.escola.buscaativacrm.controller;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import com.escola.buscaativacrm.whatsapp.AssinaturaMeta;
import com.escola.buscaativacrm.whatsapp.WebhookPayload;
import com.escola.buscaativacrm.whatsapp.WhatsAppProperties;
import com.escola.buscaativacrm.whatsapp.WhatsAppWebhookService;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import tools.jackson.databind.json.JsonMapper;

@Slf4j
@RestController
@RequestMapping("/api/webhook/whatsapp")
@RequiredArgsConstructor
public class WhatsAppWebhookController {

    private final WhatsAppProperties props;
    private final WhatsAppWebhookService webhookService;
    private final JsonMapper jsonMapper;          // o conversor JSON que o Spring já configura (Jackson 3)

    // 1) Verificação: a Meta chama UMA vez quando você salva o webhook no painel
    @GetMapping
    public ResponseEntity<String> verificar(@RequestParam("hub.mode") String mode,
                                            @RequestParam("hub.verify_token") String token,
                                            @RequestParam("hub.challenge") String challenge) {
        if ("subscribe".equals(mode) && props.verifyToken().equals(token)) {
            return ResponseEntity.ok(challenge);
        }
        return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
    }

    // 2) Eventos: mensagem recebida, status de entrega...
    @PostMapping
    public ResponseEntity<Void> receber(
            @RequestHeader(value = "X-Hub-Signature-256", required = false) String assinatura,
            @RequestBody byte[] corpo) {

        if (!AssinaturaMeta.valida(corpo, assinatura, props.appSecret())) {
            log.warn("Webhook com assinatura inválida, ignorado");
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();          // 401
        }
        webhookService.processar(jsonMapper.readValue(corpo, WebhookPayload.class));
        return ResponseEntity.ok().build();
    }
}
```

Duas perguntas que você deve estar se fazendo:

| Pergunta | Resposta |
|---|---|
| Por que `byte[]` e não `@RequestBody WebhookPayload` direto? | A assinatura é calculada sobre os **bytes exatos** que a Meta mandou. Se o Spring convertesse antes, perderíamos esses bytes. Então: confere a assinatura → **depois** converte com o `JsonMapper`. |
| E se der erro no meio (banco fora do ar)? | O Spring responde **500**, a Meta **reenvia** mais tarde, e o `existsByWamid` impede duplicar o que já tinha sido gravado. |

> No Spring Boot 4 o Jackson é a **versão 3**: o pacote mudou de `com.fasterxml.jackson.databind` para **`tools.jackson.databind`**. Se o IntelliJ sugerir o import antigo, recuse.

## A10. `ConversaController` — a API que o Angular usa

`controller/ConversaController.java`:

```java
package com.escola.buscaativacrm.controller;

import java.util.List;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

import com.escola.buscaativacrm.dto.ConversaResponse;
import com.escola.buscaativacrm.dto.EnviarMensagemRequest;
import com.escola.buscaativacrm.dto.MensagemResponse;
import com.escola.buscaativacrm.service.ConversaService;

import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;

@RestController
@RequestMapping("/api/conversas")
@RequiredArgsConstructor
public class ConversaController {

    private final ConversaService service;

    @GetMapping                                            // GET /api/conversas
    public List<ConversaResponse> listar() {
        return service.listar();
    }

    @GetMapping("/{responsavelId}/mensagens")              // GET /api/conversas/1/mensagens
    public List<MensagemResponse> mensagens(@PathVariable Long responsavelId) {
        return service.mensagens(responsavelId);
    }

    @PostMapping("/{responsavelId}/mensagens")             // POST /api/conversas/1/mensagens
    @ResponseStatus(HttpStatus.CREATED)                    // 201
    public MensagemResponse enviar(@PathVariable Long responsavelId,
                                   @Valid @RequestBody EnviarMensagemRequest request) {
        return service.enviar(responsavelId, request.texto());
    }
}
```

> Diferente do CRUD de Turma, o `POST` aqui **não** devolve `Location`: não existe `GET` de uma mensagem isolada. O `201` + o corpo com a mensagem criada bastam.

## A11. Testar a API sem front

### A11.1 Simular a Meta (sem túnel, sem celular)

`backend/http/simular-webhook.ps1` — assina o corpo com o App Secret e chama o webhook local, como a Meta faria:

```powershell
# Simula a Meta chamando o webhook local, com assinatura válida.
# Uso (na pasta backend):
#   .\http\simular-webhook.ps1 -Segredo "SEU_APP_SECRET" -Texto "Oi, aqui e a mae do Joao"
#   .\http\simular-webhook.ps1 -Segredo "SEU_APP_SECRET" -Wamid "wamid.repetido"   (rode 2x: so grava 1)
param(
    [Parameter(Mandatory)] [string] $Segredo,
    [string] $Telefone = "5544999998888",
    [string] $Texto = "Mensagem de teste",
    [string] $Wamid = "wamid.teste.$([DateTimeOffset]::UtcNow.ToUnixTimeMilliseconds())",
    [string] $Url = "http://localhost:8080/api/webhook/whatsapp"
)

$agora = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()
$json = @{
    entry = @(@{ changes = @(@{ field = "messages"; value = @{
        contacts = @(@{ profile = @{ name = "Responsavel Teste" } })
        messages = @(@{ from = $Telefone; id = $Wamid; timestamp = "$agora"; type = "text"; text = @{ body = $Texto } })
    } }) })
} | ConvertTo-Json -Depth 10 -Compress

$bytes = [Text.Encoding]::UTF8.GetBytes($json)
$hmac = [Security.Cryptography.HMACSHA256]::new([Text.Encoding]::UTF8.GetBytes($Segredo))
$assinatura = "sha256=" + (-join ($hmac.ComputeHash($bytes) | ForEach-Object { $_.ToString("x2") }))

Invoke-RestMethod -Method Post -Uri $Url -Body $bytes -ContentType "application/json" -Headers @{ "X-Hub-Signature-256" = $assinatura }
"Enviado: $Texto ($Wamid)"
```

Com o backend rodando, na pasta `backend/`:

```powershell
.\http\simular-webhook.ps1 -Segredo "COLE_O_APP_SECRET" -Texto "Meu filho estava doente"
```

### A11.2 Requisições da API

`backend/http/conversas.http`:

```http
### Listar conversas (espera o "Responsavel Teste" com janelaAberta: true)
GET http://localhost:8080/api/conversas

### Mensagens de um responsável
GET http://localhost:8080/api/conversas/1/mensagens

### Responder (ENVIA DE VERDADE pelo WhatsApp)
POST http://localhost:8080/api/conversas/1/mensagens
Content-Type: application/json

{ "texto": "Obrigado pelo retorno! Melhoras para ele." }

### Texto vazio (espera 400)
POST http://localhost:8080/api/conversas/1/mensagens
Content-Type: application/json

{ "texto": "" }

### Responsável inexistente (espera 404)
GET http://localhost:8080/api/conversas/9999/mensagens

### Webhook sem assinatura (espera 401)
POST http://localhost:8080/api/webhook/whatsapp
Content-Type: application/json

{ "entry": [] }
```

### A11.3 Roteiro

- [ ] `simular-webhook.ps1` → console mostra `Mensagem recebida de 5544999998888`.
- [ ] `GET /api/conversas` → aparece o "Responsavel Teste" com `janelaAberta: true`.
- [ ] `GET .../1/mensagens` → a mensagem com `direcao: ENTRADA`.
- [ ] Rode o script **duas vezes** com `-Wamid "wamid.repetido"` → só **uma** mensagem nova.
- [ ] Webhook sem assinatura → **401**, e nada gravado.
- [ ] Texto vazio → **400** com `campos.texto`. Id inexistente → **404**.
- [ ] **Responder** a conversa simulada → **502**, e no log `Erro da API do WhatsApp: ...`. Esperado! O número é falso e, para a Meta, essa conversa nunca existiu (a janela de 24h dela não está aberta). Leia a mensagem de erro no log — é assim que você vai diagnosticar problemas reais.
- [ ] **Janela fechada:** no Supabase → *SQL Editor*:
  ```sql
  update responsavel set ultima_recebida_em = now() - interval '25 hours' where id = 1;
  ```
  → responder de novo → **409** com "Janela de 24h fechada…" (o nosso sistema barrou **antes** de chamar a Meta).

**Teste real (com o celular):**

1. `cloudflared tunnel --url http://localhost:8080` e atualize a Callback URL na Meta (Guia 01, 7.4–7.5).
2. Do seu celular, mande uma mensagem para o número de teste.
3. `GET /api/conversas` → aparece **você**, com o nome do seu perfil e o telefone no formato `wa_id`.
4. Responda pelo `.http` usando o **id do seu responsável** → a mensagem chega no seu celular.
5. Abra a mensagem no celular e consulte as mensagens de novo → o status vai de `ENVIADA` → `ENTREGUE` → `LIDA` (vieram pelo webhook, como `statuses`).

✅ **Checkpoint:** todos os itens acima, inclusive a mensagem chegando no celular.

```powershell
git add backend
```

```powershell
git commit -m "feat(backend): caixa de entrada do WhatsApp (webhook assinado + envio)"
```

> Rode `git status` antes: `application-dev.properties` (agora com o App Secret) **não** pode aparecer.

---

# PARTE B — Frontend

## B0. Mapa JSF → Angular para esta parte

| JSF | Angular |
|---|---|
| `<p:poll interval="5" listener="...">` | `timer(0, 5000).pipe(switchMap(() => service.listar()))` |
| `<ui:include>` + `<ui:param name="id">` | componente filho `<app-chat [responsavelId]="...">` + `input()` |
| propriedade calculada no bean (`getSelecionada()`) | `computed(() => ...)` |
| `@PostConstruct` que recarrega quando um parâmetro muda | `effect(() => ...)` |
| `styleClass="#{m.saida ? 'saida' : ''}"` | `[class.saida]="m.direcao === 'SAIDA'"` |
| `<f:convertDateTime pattern="dd/MM HH:mm">` | `{{ m.dataHora \| date: 'dd/MM HH:mm' }}` |

**Por que polling e não "tempo real"?** É o mais simples que funciona: uma requisição pequena a cada 5s, para 4–8 funcionários, não pesa nada. Tempo real de verdade (SSE/WebSocket) fica para quando fizer falta.

## B1. Modelo

`frontend/src/app/conversas/conversa.model.ts`:

```ts
export type Direcao = 'ENTRADA' | 'SAIDA';
export type StatusMensagem = 'ENVIADA' | 'ENTREGUE' | 'LIDA' | 'FALHOU';

export interface Conversa {            // = ConversaResponse
  responsavelId: number;
  nome: string;
  telefone: string;
  ultimaMensagemEm: string;            // ISO-8601: JSON não tem tipo "data"; o DatePipe formata
  janelaAberta: boolean;
}

export interface Mensagem {            // = MensagemResponse
  id: number;
  direcao: Direcao;
  texto: string;
  status: StatusMensagem | null;
  dataHora: string;
}
```

## B2. Service

`frontend/src/app/conversas/conversa.service.ts` (crie o arquivo à mão, para manter o sufixo `.service`):

```ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

import { Conversa, Mensagem } from './conversa.model';

@Injectable({ providedIn: 'root' })
export class ConversaService {

  private http = inject(HttpClient);
  private readonly url = '/api/conversas';

  listar(): Observable<Conversa[]> {
    return this.http.get<Conversa[]>(this.url);
  }

  mensagens(responsavelId: number): Observable<Mensagem[]> {
    return this.http.get<Mensagem[]>(`${this.url}/${responsavelId}/mensagens`);
  }

  enviar(responsavelId: number, texto: string): Observable<Mensagem> {
    return this.http.post<Mensagem>(`${this.url}/${responsavelId}/mensagens`, { texto });  // { texto } = { texto: texto }
  }
}
```

## B3. Gerar os componentes

Em `frontend/`:

```powershell
ng generate component conversas/inbox
```

```powershell
ng generate component conversas/chat
```

- `Inbox` = a **página**: lista de conversas à esquerda, conversa aberta à direita.
- `Chat` = componente **filho**: balões + campo de envio de **uma** conversa.

## B4. Componente `Chat` (filho)

`conversas/chat/chat.ts`:

```ts
import { Component, effect, inject, input, signal } from '@angular/core';
import { DatePipe } from '@angular/common';
import { NonNullableFormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { switchMap, timer } from 'rxjs';

import { Mensagem } from '../conversa.model';
import { ConversaService } from '../conversa.service';

@Component({
  selector: 'app-chat',
  imports: [DatePipe, ReactiveFormsModule],
  templateUrl: './chat.html',
  styleUrl: './chat.css',
})
export class Chat {

  private service = inject(ConversaService);

  // entradas vindas do pai: <app-chat [responsavelId]="..." [janelaAberta]="...">
  responsavelId = input.required<number>();
  janelaAberta = input(false);

  mensagens = signal<Mensagem[]>([]);
  enviando = signal(false);
  erro = signal<string | null>(null);

  form = inject(NonNullableFormBuilder).group({
    texto: ['', [Validators.required, Validators.maxLength(4096)]],
  });

  constructor() {
    // roda de novo sempre que responsavelId() mudar (outra conversa foi selecionada)
    effect(onCleanup => {
      const id = this.responsavelId();
      this.mensagens.set([]);
      this.erro.set(null);

      // shortcut: um erro encerra o polling; selecionar a conversa de novo reinicia
      const assinatura = timer(0, 5000)                          // agora e a cada 5s (≈ <p:poll>)
        .pipe(switchMap(() => this.service.mensagens(id)))
        .subscribe({
          next: lista => this.mensagens.set(lista),
          error: () => this.erro.set('Não foi possível carregar as mensagens.'),
        });

      onCleanup(() => assinatura.unsubscribe());               // trocou de conversa ou saiu da tela: para o polling antigo
    });
  }

  enviar() {
    if (this.form.invalid) return;
    this.enviando.set(true);
    this.erro.set(null);

    this.service.enviar(this.responsavelId(), this.form.getRawValue().texto).subscribe({
      next: nova => {
        this.mensagens.update(lista => [...lista, nova]);       // aparece na hora, sem esperar o próximo poll
        this.form.reset();
        this.enviando.set(false);
      },
      error: e => {
        this.erro.set(e.error?.detail ?? 'Erro ao enviar.');    // 409 da janela, 502 da Meta...
        this.enviando.set(false);
      },
    });
  }
}
```

Pontos de sintaxe:

| Código | Significado |
|---|---|
| `input.required<number>()` | propriedade que **o pai é obrigado** a passar. Lê-se como signal: `this.responsavelId()` |
| `input(false)` | entrada opcional com valor padrão |
| `effect(fn)` | roda `fn` agora e **de novo sempre que um signal lido dentro dela mudar** |
| `onCleanup(...)` | o que fazer antes da próxima execução (ou quando o componente morrer) — sem isso, cada troca de conversa deixaria um polling "fantasma" rodando |
| `timer(0, 5000)` | emite em 0ms, 5s, 10s… |
| `switchMap` | para cada "tique", faz a requisição; se a anterior ainda não voltou, **cancela** ela |
| `mensagens.update(lista => [...lista, nova])` | cria uma lista **nova** com o item no fim (signal precisa de referência nova para perceber a mudança) |

`conversas/chat/chat.html`:

```html
<div class="mensagens">
  @for (m of mensagens(); track m.id) {
    <div class="balao" [class.saida]="m.direcao === 'SAIDA'">
      <p>{{ m.texto }}</p>
      <small>
        {{ m.dataHora | date: 'dd/MM HH:mm' }}
        @if (m.status) { · {{ m.status }} }
      </small>
    </div>
  } @empty {
    <p class="vazio">Nenhuma mensagem ainda.</p>
  }
</div>

@if (erro()) {
  <p class="erro">{{ erro() }}</p>
}

@if (janelaAberta()) {
  <form class="envio" [formGroup]="form" (ngSubmit)="enviar()">
    <input formControlName="texto" placeholder="Digite a resposta..." aria-label="Mensagem" autocomplete="off">
    <button type="submit" [disabled]="form.invalid || enviando()">Enviar</button>
  </form>
} @else {
  <p class="aviso">Janela de 24h fechada: só é possível responder depois que o responsável mandar uma mensagem.</p>
}
```

`conversas/chat/chat.css`:

```css
:host { display: flex; flex-direction: column; height: 100%; }   /* :host = a própria tag <app-chat> */

.mensagens { flex: 1; overflow-y: auto; display: flex; flex-direction: column; gap: 8px; padding: 12px; background: #ece5dd; }
.balao { max-width: 70%; align-self: flex-start; background: #fff; padding: 6px 10px; border-radius: 8px; }
.balao.saida { align-self: flex-end; background: #d9fdd3; }
.balao p { margin: 0; white-space: pre-wrap; }
.balao small { color: #667; font-size: 11px; }

.envio { display: flex; gap: 8px; padding: 8px; }
.envio input { flex: 1; }
.aviso { margin: 0; padding: 8px 12px; background: #fff3cd; }
.vazio { color: #667; }
.erro { margin: 0; padding: 8px 12px; }
```

> O CSS de componente vale **só** para ele (o Angular "carimba" um atributo único nas tags). O `.erro` daqui não afeta o `.erro` de outra tela.

## B5. Componente `Inbox` (a página)

`conversas/inbox/inbox.ts`:

```ts
import { Component, computed, inject, signal } from '@angular/core';
import { DatePipe } from '@angular/common';
import { takeUntilDestroyed } from '@angular/core/rxjs-interop';
import { switchMap, timer } from 'rxjs';

import { Chat } from '../chat/chat';
import { Conversa } from '../conversa.model';
import { ConversaService } from '../conversa.service';

@Component({
  selector: 'app-inbox',
  imports: [Chat, DatePipe],                  // usar <app-chat> no template exige importar o componente
  templateUrl: './inbox.html',
  styleUrl: './inbox.css',
})
export class Inbox {

  private service = inject(ConversaService);

  conversas = signal<Conversa[]>([]);
  selecionadaId = signal<number | null>(null);
  erro = signal<string | null>(null);

  // recalculado sozinho quando conversas() ou selecionadaId() mudam
  selecionada = computed(() => this.conversas().find(c => c.responsavelId === this.selecionadaId()));

  constructor() {
    timer(0, 5000)
      .pipe(
        switchMap(() => this.service.listar()),
        takeUntilDestroyed(),                 // para o polling quando o componente sai da tela
      )
      .subscribe({
        next: lista => {
          this.conversas.set(lista);
          this.erro.set(null);
        },
        error: () => this.erro.set('Não foi possível carregar as conversas.'),
      });
  }
}
```

**Por que guardar só o `id` selecionado, e não o objeto?** A cada 5s a lista é **substituída** por objetos novos (com `janelaAberta` atualizada). Guardando o id, o `computed` sempre aponta para a versão mais nova. E o `Chat` recebe só o **número** — como ele não muda, o `effect` do filho não reinicia a cada 5s.

`conversas/inbox/inbox.html`:

```html
<div class="inbox">
  <aside class="lista">
    @if (erro()) {
      <p class="erro">{{ erro() }}</p>
    }
    @for (c of conversas(); track c.responsavelId) {
      <button type="button" class="item"
              [class.ativo]="c.responsavelId === selecionadaId()"
              (click)="selecionadaId.set(c.responsavelId)">
        <strong>{{ c.nome }}</strong>
        <small>{{ c.telefone }} · {{ c.ultimaMensagemEm | date: 'dd/MM HH:mm' }}</small>
      </button>
    } @empty {
      <p class="vazio">Nenhuma conversa ainda. Quando um responsável mandar mensagem, ela aparece aqui.</p>
    }
  </aside>

  <section class="painel">
    @if (selecionada(); as c) {
      <header class="painel-topo">
        <strong>{{ c.nome }}</strong> <small>{{ c.telefone }}</small>
      </header>
      <app-chat [responsavelId]="c.responsavelId" [janelaAberta]="c.janelaAberta" />
    } @else {
      <p class="vazio">Selecione uma conversa.</p>
    }
  </section>
</div>
```

- `@if (selecionada(); as c)` → se existir, dá o apelido `c` (evita repetir `selecionada()!.nome`).
- Cada item da lista é um `<button>`: dá para navegar com **Tab + Enter**, de graça.

`conversas/inbox/inbox.css`:

```css
.inbox { display: grid; grid-template-columns: 280px 1fr; height: calc(100vh - 110px); background: #fff; border: 1px solid #e3e3e3; }

.lista { border-right: 1px solid #e3e3e3; overflow-y: auto; }
.item { display: flex; flex-direction: column; align-items: flex-start; gap: 2px; width: 100%; padding: 10px 12px;
        border: 0; border-bottom: 1px solid #eee; border-radius: 0; background: #fff; color: inherit; text-align: left; cursor: pointer; }
.item:hover { background: #f3f6f9; }
.item.ativo { background: #e8f0f8; }
.item small { color: #667; }

.painel { display: flex; flex-direction: column; min-height: 0; }
.painel-topo { padding: 10px 12px; border-bottom: 1px solid #e3e3e3; }
app-chat { flex: 1; min-height: 0; }

.vazio { padding: 12px; color: #667; }

@media (max-width: 700px) {
  .inbox { grid-template-columns: 1fr; height: auto; }
  .lista { max-height: 40vh; border-right: 0; border-bottom: 1px solid #e3e3e3; }
  app-chat { height: 60vh; }
}
```

## B6. Rotas

`frontend/src/app/app.routes.ts`:

```ts
import { Routes } from '@angular/router';

import { Inbox } from './conversas/inbox/inbox';

export const routes: Routes = [
  { path: '', redirectTo: 'conversas', pathMatch: 'full' },
  { path: 'conversas', component: Inbox },
];
```

## B7. Layout (componente raiz)

O teste do `/api/ping` do Guia 01 sai daqui; o `App` vira só a moldura.

`frontend/src/app/app.ts`:

```ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  imports: [RouterOutlet, RouterLink, RouterLinkActive],
  templateUrl: './app.html',
  styleUrl: './app.css',
})
export class App {}
```

`frontend/src/app/app.html`:

```html
<header class="topo">
  <strong>Busca Ativa CRM</strong>
  <nav>
    <a routerLink="/conversas" routerLinkActive="ativo">Conversas</a>
  </nav>
</header>

<main class="conteudo">
  <router-outlet />
</main>
```

## B8. CSS global

`frontend/src/styles.css`:

```css
* { box-sizing: border-box; }
body { margin: 0; font-family: system-ui, sans-serif; background: #f5f6f8; color: #222; }

.topo { display: flex; gap: 24px; align-items: center; padding: 12px 24px; background: #1f4e79; color: #fff; }
.topo a { color: #fff; text-decoration: none; opacity: .8; }
.topo a.ativo { opacity: 1; font-weight: 600; }

.conteudo { max-width: 1100px; margin: 16px auto; padding: 0 16px; }

input { padding: 8px; border: 1px solid #ccc; border-radius: 4px; font: inherit; }
button { padding: 8px 14px; border: 0; border-radius: 4px; background: #1f4e79; color: #fff; cursor: pointer; font: inherit; }
button:disabled { opacity: .6; cursor: default; }

.erro { color: #b00020; }
```

## B9. Testar tudo

1. Backend rodando. 2. `ng serve` no `frontend/`. 3. Abra http://localhost:4200 → redireciona para `/conversas`.

Roteiro:

- [ ] Com o banco vazio (ou antes de simular): "Nenhuma conversa ainda…".
- [ ] Rode o `simular-webhook.ps1` com **outro telefone** (`-Telefone 5541988887777`) → em até 5s a conversa aparece na lista **sem recarregar a página**.
- [ ] Clique na conversa → balões cinza/brancos à esquerda (ENTRADA).
- [ ] Simule outra mensagem do mesmo telefone → aparece no chat aberto em até 5s, e a conversa sobe para o topo da lista.
- [ ] Campo vazio → botão **Enviar** desabilitado.
- [ ] Responder a conversa **simulada** → aparece em vermelho "O WhatsApp recusou a mensagem…" (502, esperado — veja A11.3).
- [ ] Rode o `update ... interval '25 hours'` do A11.3 nessa conversa → em até 5s o campo some e aparece o aviso amarelo da janela fechada.
- [ ] **Real:** túnel ligado, mande mensagem do seu celular → sua conversa aparece → responda pela tela → chega no celular → balão verde à direita, status `ENVIADA` → `ENTREGUE` → `LIDA`.
- [ ] Pare o backend → "Não foi possível carregar as conversas."
- [ ] F12 → aba *Network*: uma requisição `conversas` a cada 5s (e uma `mensagens` com chat aberto). Troque de conversa e confira que o polling da anterior **parou**.

✅ **Checkpoint:** todos os itens acima.

```powershell
git add .
```

```powershell
git commit -m "feat(frontend): caixa de entrada e chat com polling"
```

```powershell
git push
```

---

## Resumo

| Backend | Frontend |
|---|---|
| `@ManyToOne(fetch = LAZY)` unidirecional + `Instant` | `interface Conversa/Mensagem` = contrato com o JSON |
| Query derivada navegando relacionamento (`findByResponsavelId...`) | `ConversaService` com `HttpClient` |
| `@ConfigurationProperties` em `record` | Componente filho com `input()` / `input.required()` |
| `RestClient` chamando a API da Meta | Polling com `timer` + `switchMap` (+ `takeUntilDestroyed`) |
| Webhook: `byte[]` → assinatura HMAC → `JsonMapper` | `effect` + `onCleanup` reagindo à troca de conversa |
| Idempotência pelo `wamid`; status que só avança | `computed` para a conversa selecionada |
| `ProblemDetail` 404/409/400/**502** | Erro do back na tela via `e.error?.detail` |
| Primeiro teste JUnit (`AssinaturaMetaTest`) | |

## Exercícios para fixar (faça sozinho antes do Guia 03)

1. **Contador de não lidas:** campo `int naoLidas` em `Responsavel` (soma 1 a cada mensagem recebida; zera num novo `POST /api/conversas/{id}/lidas` chamado quando o funcionário abre a conversa). Mostre um "badge" com o número na lista.
2. **Prévia da última mensagem:** campo `ultimoTexto` em `Responsavel` (atualizado ao receber e ao enviar), exibido embaixo do nome na lista, cortado em 40 caracteres.
3. **Telefone BR normalizado:** um método `Telefones.variantes("5544999998888")` que devolve as duas formas (com e sem o 9 depois do DDD) e uma consulta `findByTelefoneIn(List<String>)` no repository. Use em `registrarRecebida` para não criar responsável duplicado quando o admin tiver cadastrado o número "com 9" e o WhatsApp mandar "sem 9". Escreva um teste JUnit para `variantes`.

## Próximo: Guia 03 — Alunos, Turmas e template de falta

- `Turma` e `Aluno` (`@ManyToOne` Aluno → Turma e Aluno → Responsavel); cadastro de responsável **pelo admin** (com o exercício 3 resolvido).
- **Iniciar** conversa fora da janela de 24h: criar e aprovar na Meta um template *Utility* (ex.: "Olá, {{1}}. {{2}} não compareceu hoje ao colégio…") e `WhatsAppClient.enviarTemplate(...)`.
- Depois: login com Spring Security + BCrypt (cada funcionário vê só as conversas das suas turmas).
