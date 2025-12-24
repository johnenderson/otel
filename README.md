# OpenTelemetry com Spring Boot 4.0 Demo

Esta demonstração apresenta o novo `spring-boot-starter-opentelemetry` introduzido no Spring Boot 4.0, fornecendo uma
maneira simplificada de integrar observabilidade OpenTelemetry em suas aplicações Spring Boot.

## Novidades no Spring Boot 4.0

O Spring Boot 4.0 introduz um starter oficial de OpenTelemetry da equipe Spring. Diferente das abordagens anteriores que
exigiam múltiplas dependências e configuração complexa, este starter fornece:

> **Como isso foi possível?** A [modularização do Spring Boot](https://spring.io/blog/2025/10/28/modularizing-spring-boot)
> na versão 4.0 permitiu que a equipe criasse starters focados e opcionais como este. Para saber mais sobre a arquitetura
> modular do Spring Boot 4, confira os [exemplos de modularização](https://github.com/danvega/sb4/tree/master/features/modularization).

- **Dependência única**: Basta adicionar `spring-boot-starter-opentelemetry`
- **Exportação OTLP automática**: Métricas e traces são exportados via protocolo OTLP
- **Integração com Micrometer**: Usa a ponte de rastreamento do Micrometer para exportar traces no formato OTLP
- **Agnóstico de fornecedor**: Funciona com qualquer backend compatível com OpenTelemetry (Grafana, Jaeger, etc.)

### Por Que Esta Abordagem?

Existem três maneiras de usar OpenTelemetry com Spring Boot:

1. **OpenTelemetry Java Agent** - Sem mudanças no código, mas pode ter problemas de compatibilidade de versão
2. **Starter OpenTelemetry de Terceiros** - Do projeto OTel, mas puxa dependências alpha
3. **Spring Boot Starter (esta demo)** - Suporte oficial do Spring, estável, bem integrado

O insight chave é que **é o protocolo (OTLP) que importa**, não a biblioteca.
O Spring Boot usa Micrometer internamente, mas exporta tudo via OTLP para qualquer backend compatível.

### E o Spring Boot Actuator?

O Spring Boot Actuator é a abordagem tradicional do Spring para observabilidade e prontidão para produção. Veja como eles se comparam:

| Aspecto | Spring Boot Actuator | OpenTelemetry Starter |
|---------|---------------------|----------------------|
| **Protocolo** | Prometheus, OTLP, JMX, + muitos outros | OTLP (agnóstico de fornecedor) |
| **Rastreamento Distribuído** | Integrado via Micrometer Tracing (adicionar dependência de ponte) | Integrado, automático |
| **Dependência de Backend** | Agnóstico via Micrometer (suporta mais de 15 backends incluindo OTLP) | Funciona com qualquer backend OTLP |
| **Health Checks** | Integrado `/actuator/health` | Não incluído (requer Actuator) |
| **Prontidão para Produção** | Suite completa (info, env, beans, metrics, etc.) | Focado apenas em telemetria |
| **Complexidade de Setup** | Mais endpoints para configurar/proteger | Único endpoint OTLP |
| **Dependências (Spring Boot 4)** | `spring-boot-starter-actuator` + deps de ponte | `spring-boot-starter-opentelemetry` |

**Escolha Actuator quando:**
- Você precisa de health checks, probes de readiness/liveness para Kubernetes
- Você quer expor informações da aplicação, ambiente ou detalhes de beans
- Sua stack de monitoramento já é baseada em Prometheus com scraping

**Escolha OpenTelemetry Starter quando:**
- Você quer observabilidade agnóstica de fornecedor (fácil troca de backends)
- Rastreamento distribuído entre serviços é uma prioridade
- Você prefere telemetria baseada em push ao invés de scraping baseado em pull

**Nota:** Eles não são mutuamente exclusivos—muitas aplicações em produção usam ambos (Actuator para health/readiness, OTel para telemetria).

## Pré-requisitos

- Java 17+
- Maven
- Docker (para a stack Grafana LGTM)

A stack LGTM é a stack de observabilidade open-source da Grafana Labs. A sigla significa:

- Loki — para logs (sistema de agregação de logs)
- Grafana — para visualização e dashboards
- Tempo — para traces (backend de rastreamento distribuído)
- Mimir — para métricas (armazenamento de longo prazo para métricas Prometheus)

## Dependências

A dependência chave é o novo starter OpenTelemetry:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
</dependency>
```

Este starter inclui:
- API OpenTelemetry
- Ponte de rastreamento Micrometer para OpenTelemetry
- Exportadores OTLP para métricas e traces

## Configuração

```yaml
spring:
  application:
    name: ot

management:
  tracing:
    sampling:
      probability: 1.0  # 100% de amostragem para desenvolvimento
  otlp:
    metrics:
      export:
        url: http://localhost:4318/v1/metrics
  opentelemetry:
    tracing:
      export:
        otlp:
          endpoint: http://localhost:4318/v1/traces
    logging:
      export:
        otlp:
          endpoint: http://localhost:4318/v1/logs
```

### Notas de Configuração

- **sampling.probability**: Configure para `1.0` em desenvolvimento (todos os traces). Use valores menores em produção (padrão é `0.1`)
- **Porta 4318**: Endpoint OTLP HTTP (use 4317 para gRPC)
- O módulo `spring-boot-docker-compose` configura automaticamente esses endpoints ao usar Docker Compose

### Entendendo a Configuração de Exportação OTLP

**`management.otlp.metrics.export.url`** — Informa ao Spring Boot onde enviar **métricas** (contadores, gauges, histogramas como contagens de requisições, tempos de resposta, uso de memória). Os dados vão para um coletor compatível com OTLP.

**`management.opentelemetry.tracing.export.otlp.endpoint`** — Informa ao Spring Boot onde enviar **traces** (dados de tempo/fluxo mostrando como as requisições se movem pela sua aplicação, spans mostrando cada operação e duração).

**Por que duas configurações separadas?** A observabilidade do Spring Boot evoluiu ao longo do tempo:
- Métricas usam o exportador OTLP do Micrometer (daí `management.otlp.metrics`)
- Traces usam a ponte de rastreamento OpenTelemetry (daí `management.opentelemetry.tracing`)

Ambos enviam dados para o mesmo coletor (porta 4318), mas os caminhos de configuração diferem devido a como as bibliotecas são integradas.

## Executando a Demo

1. **Inicie a aplicação:**

```bash
./mvnw spring-boot:run
```

Isso inicia automaticamente o container Grafana LGTM via Docker Compose.

2. **Gere alguns traces:**

```bash
# Endpoint simples
curl http://localhost:8080/

# Saudação com variável de caminho
curl http://localhost:8080/greet/World

# Operação lenta (500ms)
curl http://localhost:8080/slow
```

3. **Visualize traces no Grafana:**

- Abra http://localhost:3000
- Vá para **Explore** (ícone de bússola)
- Selecione **Tempo** como fonte de dados
- Clique em **Search** e selecione o serviço "ot"
- Clique em um trace para ver os detalhes do span

4. **Visualize métricas:**

- No Grafana, vá para **Explore**
- Selecione **Prometheus** como fonte de dados
- Consulte métricas como `http_server_requests_seconds_count`

## Endpoints

| Endpoint | Descrição |
|----------|-------------|
| `GET /` | Resposta simples hello world |
| `GET /greet/{name}` | Retorna saudação com 50ms de trabalho simulado |
| `GET /slow` | Simula uma operação lenta de 500ms |

## O Que Você Obtém Automaticamente

Com `spring-boot-starter-opentelemetry`, você obtém instrumentação automática para:

- Requisições HTTP do servidor (todos os endpoints de controller)
- Requisições HTTP do cliente (RestTemplate, RestClient, WebClient)
- Chamadas de banco de dados JDBC
- IDs de trace/span nos logs

## Visualizando Logs com Contexto de Trace

Os logs da sua aplicação incluem automaticamente IDs de trace e span. Procure por entradas de log como:

```
2025-12-22T11:30:05.801-05:00  INFO 13165 --- [ot] [nio-8080-exec-2] [f64d5e13e35ac0429ad2974512a3c4a7-2d46fc21f9ddfb2d] dev.danvega.ot.HomeController            : Greeting user: World
```

O contexto de trace aparece como `[traceId-spanId]` (ex: `f64d5e13e35ac0429ad2974512a3c4a7-2d46fc21f9ddfb2d`).

### Entendendo IDs de Trace e Span

- **Trace ID** (`f64d5e13e35ac0429ad2974512a3c4a7`): Identifica todo o fluxo da requisição através de todos os serviços. Cada log e span de uma única requisição compartilha este ID.
- **Span ID** (`2d46fc21f9ddfb2d`): Identifica uma operação específica dentro do trace. Uma única requisição pode ter múltiplos spans (controller → banco de dados → API externa).

### Por Que Isso Importa

1. **Pular do trace para logs**: No Grafana, ao visualizar um trace no Tempo, você pode clicar em um span para ver apenas os logs emitidos durante aquela operação exata—sem mais buscar através de milhares de linhas de log.

2. **Pular dos logs para trace**: Se você encontrar um erro nos seus logs, copie o trace ID e busque por ele no Tempo para ver o fluxo completo da requisição e identificar onde falhou.

3. **Debugar operações específicas**: Se uma requisição chama múltiplos serviços ou realiza várias operações, o span ID diz exatamente qual parte da requisição gerou uma entrada de log específica.

### Exemplo: Requisição Multi-Span

```
Requisição: GET /greet/World
├── Span A (controller) ─── logs mostram o ID do span A
│   └── Span B (simulateWork) ─── logs mostram o ID do span B
```

Quando você vê um log com o ID do span B, você sabe que aconteceu durante `simulateWork()`, não no método do controller.

## Exportando Logs via OTLP

Esta demo está configurada para exportar logs para Grafana/Loki via OTLP. Isso permite visualizar, buscar e correlacionar logs com traces diretamente no Grafana.

### Importante: Requisito de Versão do Spring Boot

**Spring Boot 4.0.1 ou superior é necessário** para exportação de logs OTLP sem o módulo actuator.

No Spring Boot 4.0.0, o `OtlpLoggingAutoConfiguration` fazia parte do módulo actuator, exigindo que você adicionasse `spring-boot-starter-actuator` apenas para exportar logs. Isso foi corrigido na [issue #48488](https://github.com/spring-projects/spring-boot/issues/48488) e lançado no Spring Boot 4.0.1, que moveu a auto-configuração de exportação de logs para o módulo core OpenTelemetry.

### Como Funciona

O Spring Boot fornece auto-configuração para exportar logs no formato OTLP, mas não instala um appender no Logback por padrão. Para habilitar a exportação de logs, você precisa:

#### 1. Adicionar a Dependência do Appender Logback

```xml
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-logback-appender-1.0</artifactId>
    <version>2.21.0-alpha</version>
</dependency>
```

> **Nota:** O sufixo `-alpha` indica que isso ainda é marcado como instável pelo projeto OpenTelemetry. Atualmente não há versões estáveis (não-alpha) deste appender.

#### 2. Criar `logback-spring.xml`

Crie `src/main/resources/logback-spring.xml` para adicionar o appender OpenTelemetry:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/base.xml"/>

    <appender name="OTEL" class="io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender">
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="OTEL"/>
    </root>
</configuration>
```

Esta configuração:
- Importa a configuração padrão de logging do Spring Boot
- Adiciona um appender OpenTelemetry que envia logs para o SDK OpenTelemetry
- Anexa tanto o console quanto os appenders OTEL ao logger raiz

#### 3. Criar o Bean de Instalação do Appender

O appender OpenTelemetry precisa saber qual instância `OpenTelemetry` usar. Crie este componente:

```java
@Component
class InstallOpenTelemetryAppender implements InitializingBean {

    private final OpenTelemetry openTelemetry;

    InstallOpenTelemetryAppender(OpenTelemetry openTelemetry) {
        this.openTelemetry = openTelemetry;
    }

    @Override
    public void afterPropertiesSet() {
        OpenTelemetryAppender.install(this.openTelemetry);
    }
}
```

Este bean:
- Obtém a instância `OpenTelemetry` auto-configurada injetada pelo Spring Boot
- Instala-a no appender Logback após o contexto Spring estar pronto
- Habilita o appender a enviar logs através do SDK OpenTelemetry para o endpoint OTLP

#### 4. Auto-Configuração do Docker Compose

Ao usar `spring-boot-docker-compose` com a imagem `grafana/otel-lgtm`, o Spring Boot **configura automaticamente** os endpoints OTLP para métricas, traces e logs. Você não precisa especificar URLs de endpoint explícitas no `application.yaml`.

A auto-configuração detecta o container em execução e configura:
- Endpoint de métricas: `http://localhost:4318/v1/metrics`
- Endpoint de traces: `http://localhost:4318/v1/traces`
- Endpoint de logs: `http://localhost:4318/v1/logs`

### Visualizando Logs no Grafana

1. Abra http://localhost:3000
2. Vá para **Explore** (ícone de bússola)
3. Selecione **Loki** como fonte de dados
4. Consulte logs: `{service_name="ot"}`
5. Clique em uma entrada de log para ver seu contexto de trace, depois clique no trace ID para pular diretamente para o trace no Tempo

### Configuração Manual (Opcional)

Se você não estiver usando auto-configuração do Docker Compose, pode definir explicitamente o endpoint:

```yaml
management:
  opentelemetry:
    logging:
      export:
        otlp:
          endpoint: http://localhost:4318/v1/logs
```

Para mais detalhes, veja o [post do blog OpenTelemetry com Spring Boot](https://spring.io/blog/2025/11/18/opentelemetry-with-spring-boot#exporting-logs).

## Próximos Passos

Para estender esta demo, você poderia:

1. **Adicionar spans customizados** usando a anotação `@Observed`:

```java
@Observed(name = "minha-operacao")
public void meuMetodo() {
    // ...
}
```

2. **Adicionar trace ID às respostas** usando um filtro servlet:

```java
@Component
class TraceIdFilter extends OncePerRequestFilter {
    private final Tracer tracer;

    // ... injetar tracer e adicionar cabeçalho X-Trace-Id
}
```

3. **Chamar outros serviços** para ver rastreamento distribuído através de múltiplas aplicações

## Recursos

- [OpenTelemetry com Spring Boot - Spring Blog](https://spring.io/blog/2025/11/18/opentelemetry-with-spring-boot)
- [Documentação de Tracing do Spring Boot](https://docs.spring.io/spring-boot/reference/actuator/tracing.html)
- [Documentação OpenTelemetry](https://opentelemetry.io/docs/)
- [Micrometer Tracing](https://micrometer.io/docs/tracing)