# Deploy da Calculadora RTC na Vercel (Container Runtime)

## Visão Geral

Este documento descreve a arquitetura e o processo para fazer deploy da **Calculadora de Tributos da Receita Federal** (projeto \calculadora-rtc-regime-regular\) na Vercel utilizando o **runtime de containers Docker**.

A aplicação é uma API REST Spring Boot 3.5.16 (Java 21) que utiliza **SQLite em modo somente leitura** como banco de dados embarcado. O banco é gerado em tempo de build via Flyway e incluído na imagem Docker final.

---

## Arquitetura de Deploy

\\\
┌─────────────────────────────────────────────────────────────────┐
│                        VERCEL CONTAINER RUNTIME                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Docker Container (eclipse-temurin:21-jre-alpine)         │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  Spring Boot App (porta \)                    │  │  │
│  │  │  - Context path: /api                               │  │  │
│  │  │  - Endpoints: /api/calculadora/*                    │  │  │
│  │  │  - SQLite DB: ./calculadora/db/calculadora-pro.db  │  │  │
│  │  │    (read-only, mode=ro)                             │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │  User: appuser (UID 1000, non-root)                      │  │
│  └───────────────────────────────────────────────────────────┘  │
│  - Porta: definida pela variável PORT (Vercel)                 │
│  - Filesystem: efêmero (read-only após deploy)                 │
│  - Memória: 1024 MB (configurável)                             │
│  - Timeout: 60s (configurável)                                 │
└─────────────────────────────────────────────────────────────────┘
\\\

### Características Principais

- **Multi-stage build**: Separação de build (JDK) e runtime (JRE) para imagem enxuta
- **SQLite read-only**: Banco gerado no build, aberto em \mode=ro\ no runtime
- **Non-root user**: Execução com usuário \ppuser\ (UID 1000)
- **Port binding**: Respeita variável \PORT\ injetada pela Vercel
- **Graceful shutdown**: Entrypoint em exec form para tratamento correto de SIGTERM
- **JVM otimizada**: \UseContainerSupport\, \MaxRAMPercentage=75\, entropy otimizada

---

## Pré-requisitos

- **Docker** instalado localmente (para build e teste)
- **Conta Vercel** com acesso ao projeto
- **Git** para versionamento
- **Java 21** e **Maven** (apenas para desenvolvimento local, não necessários no CI/CD da Vercel)

---

## Estrutura dos Arquivos Criados

| Arquivo | Descrição |
|---------|-----------|
| \Dockerfile.vercel\ | Multi-stage Dockerfile otimizado para Vercel |
| \.dockerignore\ | Exclui arquivos desnecessários do contexto de build |
| \ercel.json\ | Configuração do projeto na Vercel |
| \DEPLOY_VERCEL.md\ | Esta documentação |

---

## Como Construir a Imagem Localmente

\\\ash
# Na raiz do projeto
docker build -f Dockerfile.vercel -t calculadora-rtc:local .
\\\

### O que acontece no build:

1. **Stage \uilder\** (eclipse-temurin:21-jdk-alpine):
   - Baixa dependências Maven (cacheado)
   - Executa \lyway:migrate\ para gerar \calculadora-pro.db\ em \./calculadora/db/\
   - Compila e empacota o JAR (\pi-regime-geral.jar\)

2. **Stage \untime\** (eclipse-temurin:21-jre-alpine):
   - Copia apenas o JAR e o banco SQLite
   - Cria usuário não-root
   - Define entrypoint com suporte a \PORT\

---

## Como Executar o Container Localmente

\\\ash
# Porta 8080 (padrão local)
docker run --rm -p 8080:8080 -e PORT=8080 calculadora-rtc:local

# Porta customizada (simulando Vercel)
docker run --rm -p 3000:3000 -e PORT=3000 calculadora-rtc:local
\\\

A aplicação estará disponível em:
- **API Base**: \http://localhost:8080/api\ (ou porta configurada)
- **Swagger UI**: \http://localhost:8080/api/swagger-ui.html\
- **OpenAPI Docs**: \http://localhost:8080/api/api-docs\
- **Health Check**: \http://localhost:8080/api/actuator/health\
- **Metrics (Prometheus)**: \http://localhost:9101/metrics\ (porta separada, não exposta na Vercel)

---

## Como Testar os Endpoints

### 1. Health Check
\\\ash
curl http://localhost:8080/api/actuator/health
\\\

### 2. Cálculo de Tributos (Regime Geral)

Use um dos arquivos de exemplo em \src/test/resources/entradas/calculoscorretos/\:

\\\ash
# Exemplo simples
curl -X POST http://localhost:8080/api/calculadora/regime-geral \
  -H "Content-Type: application/json" \
  -d @src/test/resources/entradas/calculoscorretos/cenario_01_estado_input.json

# Exemplo com vários itens
curl -X POST http://localhost:8080/api/calculadora/regime-geral \
  -H "Content-Type: application/json" \
  -d @src/test/resources/entradas/calculoscorretos/varios_itens/Teste_Varios_Itens.json
\\\

### 3. Observabilidade (cálculo com erros por item)
\\\ash
curl -X POST http://localhost:8080/api/calculadora/observabilidade/regime-geral \
  -H "Content-Type: application/json" \
  -d @src/test/resources/entradas/calculoscorretos/observabilidade/entrada.json
\\\

### 4. Dados Abertos - Classificações Tributárias
\\\ash
curl http://localhost:8080/api/calculadora/dados-abertos/classificacoes-tributarias/cbs-ibs
\\\

### 5. Dados Abertos - NCM
\\\ash
curl "http://localhost:8080/api/calculadora/dados-abertos/ncm?codigo=200043&data=2026-01-01"
\\\

### 6. Base de Cálculo Unificada (NF-e)
\\\ash
curl -X POST http://localhost:8080/api/calculadora/base-calculo/base-calculo-unificada \
  -H "Content-Type: application/json" \
  -d @src/test/resources/entradas/basecalculo/entrada.json
\\\

### 7. Validação XML
\\\ash
curl -X POST "http://localhost:8080/api/calculadora/xml/validate?tipo=nfe&subtipo=nota" \
  -H "Content-Type: application/xml" \
  --data-binary @src/test/resources/xml/nfe/valido/nota/nfe_valido.xml
\\\

### 8. Versão Offline (apenas perfil offline)
\\\ash
curl http://localhost:8080/api/calculadora/versao/status
\\\

---

## Como Configurar o Projeto na Vercel

### 1. Via Dashboard (Interface Web)

1. Acesse [vercel.com/dashboard](https://vercel.com/dashboard)
2. Clique em **Add New...** → **Project**
3. Importe o repositório Git (GitHub, GitLab, Bitbucket)
4. Na configuração do projeto:
   - **Framework Preset**: \Other\ (ou deixe vazio)
   - **Build Command**: \docker build -f Dockerfile.vercel .\ (já no vercel.json)
   - **Output Directory**: deixe vazio
   - **Install Command**: deixe vazio
   - **Development Command**: deixe vazio
5. Clique em **Deploy**

### 2. Via Vercel CLI

\\\ash
# Instalar Vercel CLI
npm i -g vercel

# Login
vercel login

# Deploy (na raiz do projeto)
vercel --prod
\\\

### 3. Configurações Importantes no Painel da Vercel

Após o primeiro deploy, configure no painel da Vercel (**Settings** → **Functions**):

| Configuração | Valor Recomendado | Observação |
|--------------|-------------------|------------|
| **Memory** | 1024 MB | Mínimo 512 MB para Spring Boot + SQLite cache |
| **Max Duration** | 60 segundos | Cálculos complexos podem demorar |
| **Regions** | \gru1\ (São Paulo) | Latência mínima para usuários BR |
| **Concurrency** | 10-50 | Dependendo do plano |

---

## Variáveis de Ambiente Necessárias

Configure no painel da Vercel (**Settings** → **Environment Variables**):

| Variável | Obrigatória | Valor | Descrição |
|----------|-----------|-------|-----------|
| \SPRING_PROFILES_ACTIVE\ | Sim | \offline\ | Ativa perfil de produção com SQLite read-only |
| \VERSAO_OFFLINE_URL\ | Não | \https://piloto-cbs.tributos.gov.br/servico/calculadora-consumo/api/calculadora/dados-abertos/versao\ | URL para verificação de versão remota (feature opcional) |
| \APPLICATION_DOWNLOAD_PLATFORMS_*\ | Não | \{"windows":"https://...","linux":"https://...","mac":"https://..."}\ | URLs de download por plataforma (configuração complexa, usar JSON) |

### Nota sobre \APPLICATION_DOWNLOAD_PLATFORMS\

Se precisar configurar plataformas de download, use a sintaxe de variáveis aninhadas da Vercel ou configure via arquivo \pplication-offline.yml\ customizado no build.

---

## Como Configurar o Domínio

1. No painel da Vercel: **Settings** → **Domains**
2. Adicione seu domínio personalizado (ex: \pi.calculadora.gov.br\)
3. Configure os registros DNS conforme instruções da Vercel:
   - **A record**: \76.76.21.21\ (ou CNAME para \cname.vercel-dns.com\)
4. Aguarde propagação SSL (automática via Let's Encrypt)

---

## Como Configurar CORS

A aplicação já possui CORS habilitado com configuração padrão do Spring Security (\cors(withDefaults())\ em \WebSecurityConfig.java\).

### Para produção, recomenda-se restringir as origens:

**Opção 1: Via variável de ambiente (recomendado)**

Adicione ao \pplication-offline.yml\ ou via variáveis de ambiente na Vercel:

\\\yaml
spring:
  web:
    cors:
      allowed-origins: "https://seu-frontend.vercel.app,https://app.seudominio.com.br"
      allowed-methods: "GET,POST,OPTIONS"
      allowed-headers: "Content-Type,Authorization,X-Requested-With"
      allow-credentials: false
      max-age: 3600
\\\

**Opção 2: Via WebSecurityConfig (alteração de código)**

Modifique \WebSecurityConfig.java\ para usar \CorsConfigurationSource\ customizado.

### Princípios de Segurança Aplicados

- ✅ Origens configuráveis por variável de ambiente
- ✅ Sem wildcard (\*\) indiscriminado em produção
- ✅ Apenas métodos necessários (\GET\, \POST\, \OPTIONS\)
- ✅ Apenas headers necessários
- ✅ Credenciais desabilitadas (\llow-credentials: false\)
- ⚠️ CORS **não** é mecanismo de autenticação; use WAF/API Gateway para proteção real

---

## Como Visualizar os Logs

### No Painel da Vercel

1. Acesse o projeto → **Deployments**
2. Clique no deployment ativo
3. Aba **Functions** → clique na função container
4. Visualize **Logs** em tempo real ou histórico

### Via Vercel CLI

\\\ash
# Logs em tempo real
vercel logs --follow

# Logs de um deployment específico
vercel logs <deployment-url>
\\\

### Logs Importantes para Monitorar

- **Startup**: \Started CalculadoraTributoApplication in X seconds\
- **Database**: \Banco de dados em uso: [calculadora-pro.db]\
- **Porta**: \Tomcat started on port(s): \\
- **Erros de cálculo**: \ERROR\ em \r.gov.serpro.rtc\ com stack trace
- **Métricas**: \/actuator/health\ e \/metrics\ (Prometheus)

---

## Limitações Conhecidas

| Limitação | Impacto | Mitigação |
|-----------|---------|-----------|
| **Cold Start** | ~3-8s na primeira requisição após inatividade | Configure *Minimum Instances* > 0 no plano Pro/Enterprise; use warm-up cron |
| **Timeout 60s** | Cálculos muito complexos podem exceder | Otimize payloads; divida em lotes; aumente no painel (máx 300s no Pro) |
| **Memória 1GB** | Cache Caffeine + JVM + SQLite pode pressionar | Ajuste \cache.specs\ no \pplication-offline.yml\; monitore via \/metrics\ |
| **Filesystem Efêmero** | Escrita impossível após deploy | Banco já é read-only (\mode=ro\); logs apenas stdout/stderr |
| **Sem Persistência** | Não há estado entre invocações | Projetado para stateless; cache em memória (Caffeine) por instância |
| **Arquitetura x86_64** | Imagem buildada para AMD64 | Vercel roda em AMD64; compatível |
| **Porta Única HTTP** | Porta de management (9101) não acessível externamente | Métricas via \/actuator/prometheus\ no mesmo endpoint se necessário |
| **Tamanho da Imagem** | ~200-300MB comprimida | Multi-stage build já otimizado; dentro dos limites da Vercel (1GB) |
| **Concorrência** | 1 request por instância por vez (padrão) | Configure \server.tomcat.threads.max\ e concurrency na Vercel |

---

## Possíveis Custos e Limites Operacionais

### Vercel Hobby (Gratuito)
- **Bandwidth**: 100 GB/mês
- **Function Executions**: 100.000/mês
- **Function Duration**: 100.000 GB-hours/mês
- **Max Duration**: 10s (Serverless) / 60s (Container)
- **Memory**: 1024 MB
- **Regions**: 1 (padrão)
- **Concurrency**: 10

### Vercel Pro (/mês por membro)
- **Bandwidth**: 1 TB/mês (+.15/GB extra)
- **Function Executions**: Ilimitado
- **Max Duration**: 300s
- **Memory**: Até 3072 MB
- **Regions**: Múltiplas
- **Concurrency**: 1000
- **Minimum Instances**: Configurável (evita cold start)

### Estimativa para Calculadora RTC
- **Imagem**: ~250 MB
- **Cold start**: ~5s (JVM + Spring Boot + Hibernate init)
- **Request típico**: 100-500ms
- **Request complexo (vários itens)**: 1-5s
- **Memória em uso**: ~400-600 MB (heap + metaspace + cache + DB mmap)

**Recomendação**: Plano **Pro** para produção (mínimas instâncias, timeout maior, concorrência).

---

## Processo de Atualização para Novas Versões do Código Oficial

Quando houver nova versão do repositório oficial da Receita Federal:

1. **Atualize o repositório local**
   \\\ash
   git pull origin main
   # ou faça merge da tag/version desejada
   \\\

2. **Verifique mudanças no \pom.xml\**
   - Versão do Spring Boot
   - Versão do Java
   - Novas dependências

3. **Verifique mudanças no \pplication-offline.yml\**
   - Novas propriedades de cache
   - Mudanças na configuração do banco
   - Novos endpoints de actuator

4. **Verifique migrações Flyway** (\lyway/sql/criacao/\)
   - Novos scripts \Vxxxx__*.sql\
   - O build automático executará as migrações

5. **Execute testes localmente** (se tiver Java/Maven)
   \\\ash
   ./mvnw clean test -Dspring.profiles.active=testes
   \\\

6. **Build e teste da imagem Docker**
   \\\ash
   docker build -f Dockerfile.vercel -t calculadora-rtc:test .
   docker run --rm -p 8080:8080 -e PORT=8080 calculadora-rtc:test
   # Teste endpoints (ver seção "Como Testar os Endpoints")
   \\\

7. **Deploy na Vercel**
   \\\ash
   git add .
   git commit -m "chore: update to RTC version X.Y.Z"
   git push origin main
   # Vercel fará auto-deploy se configurado
   # Ou: vercel --prod
   \\\

8. **Valide em produção**
   - Health check: \https://seu-dominio.vercel.app/api/actuator/health\
   - Teste de cálculo real
   - Verifique logs de startup
   - Monitore métricas por 24h

---

## Checklist de Deploy (Definition of Done)

- [ ] Imagem Docker builda sem erros (\docker build -f Dockerfile.vercel .\)
- [ ] Container inicia localmente (\docker run -p 8080:8080 -e PORT=8080 <image>\)
- [ ] Health check responde 200 (\/api/actuator/health\)
- [ ] Endpoint de cálculo responde corretamente (teste com payload oficial)
- [ ] Swagger UI acessível (\/api/swagger-ui.html\)
- [ ] Banco SQLite incluído na imagem (verificar \docker run --rm <image> ls -la /app/calculadora/db/\)
- [ ] Variáveis de ambiente configuradas na Vercel
- [ ] CORS configurado para domínio do frontend
- [ ] Domínio personalizado configurado e SSL ativo
- [ ] Logs de startup limpos (sem erros WARN/ERROR críticos)
- [ ] Métricas Prometheus expostas (\/api/actuator/prometheus\ ou \/metrics\)
- [ ] Deploy em produção validado com requisições reais
- [ ] Rollback testado (redeploy versão anterior)

---

## Arquivos de Referência

- **Documentação Vercel Containers**: https://vercel.com/docs/functions/docker-containers
- **Guia Docker na Vercel**: https://vercel.com/kb/guide/docker
- **Changelog Vercel Docker**: https://vercel.com/changelog/bring-your-dockerfile-to-vercel-functions
- **Spring Boot em Containers**: https://spring.io/guides/gs/spring-boot-docker
- **SQLite JDBC**: https://github.com/xerial/sqlite-jdbc

---

## Suporte e Contato

Para dúvidas sobre a calculadora oficial: consulte o repositório original da Receita Federal.
Para dúvidas sobre deploy na Vercel: consulte a documentação oficial da Vercel ou abra issue neste repositório.