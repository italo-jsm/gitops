# Implantação do AI em ai.italomariano.dev.br

## Estado verificado

- O projeto [ai](../ai) usa Java 21 e Spring Boot, atende na porta 8080 e roda como UID 10001. O workflow de `main` roda `./gradlew clean build` e publica `ghcr.io/italo-jsm/ai:sha-<7 caracteres>` para amd64 e arm64.
- O GitOps usa k3s, Traefik e Argo CD. O [Ingress do Finance](applications/finance/ingress.yaml) usa o certificado `wildcard-italomariano-dev-br` no namespace `default`, gerenciado pelo cert-manager. A consulta ao cluster mostrou Finance, Keycloak e `nous-enqueuer` prontos em `default`.
- O AI usa PostgreSQL com Liquibase e pgvector. O padrão local aponta para o loopback e precisa ser substituído no cluster. Todos os perfis de chat usam Ollama para os embeddings `nomic-embed-text` de 768 dimensões.
- O build atual não inclui Actuator. O endpoint de chat identifica a conta pelo `username` enviado no corpo, e o cadastro aceita um ID de Telegram opcional. Liberar essas rotas na internet exige controle de identidade e de vínculo de conta.

## 1. Preparar a aplicação e a imagem

- Adicionar Actuator no projeto AI e habilitar as probes de saúde. Usar `/actuator/health/liveness` para startup e liveness e `/actuator/health/readiness` para readiness, com o indicador do banco explicitamente incluído. Validar migração e acesso ao modelo em testes de implantação. Manter o Actuator fora das rotas públicas.
- Rodar o build com Java 21 e selecionar uma imagem publicada pelo workflow, fixada pelo digest no Deployment. Validar o acesso com `ghcr-credentials` em `default` se o pacote for privado.

## 2. Preparar banco e inferência

- Usar o PostgreSQL 18 que hospeda o Finance em `192.168.1.191:5432`, com banco e usuário exclusivos `ai`. O servidor aceita conexões e o pacote `postgresql-18-pgvector` está instalado. Um pod temporário em `default` confirmou o login como `ai` no banco `ai` e a presença de `vector`, `hstore` e `uuid-ossp`. Testar Liquibase e a criação do esquema vetorial com o usuário da aplicação. Incluir o banco `ai` na rotina de backup do servidor e testar a restauração. Se houver dados locais para migrar, restaurar uma cópia e validar usuários, memórias e vetores antes do corte.
- Usar inicialmente o perfil `ollama` com uma instância em endereço privado alcançável pelos pods. Configurar `OLLAMA_BASE_URL` e `OLLAMA_CHAT_MODEL` explicitamente. Preparar o modelo de chat escolhido e `nomic-embed-text` antes do rollout, pois o pull automático está desativado. Validar geração com ferramentas, embeddings e latência de dentro do cluster. Se a capacidade local for insuficiente, considerar o perfil `groq` ou `openai-compatible`, mantendo o Ollama privado para embeddings.

## 3. Implantar internamente via Argo CD

- Criar `bootstrap/ai.yaml` e `applications/ai/` com Deployment, Service ClusterIP (porta 80 para 8080) e ConfigMap. Usar `default` para seguir o padrão atual e reaproveitar o wildcard. Começar com uma réplica, configurar requests e limits a partir de um teste de carga e usar as probes da etapa 1.
- Definir `SPRING_DATASOURCE_URL=jdbc:postgresql://192.168.1.191:5432/ai`, perfil e parâmetros do Ollama no ConfigMap. Injetar usuário/senha do banco e credenciais de integrações por referências a Secrets provisionados fora do Git. Configurar inicialmente `TELEGRAM_CONSUMER_ENABLED=false` e `FINANCE_OAUTH_ENABLED=false`.
- Usar a conexão ao banco já validada a partir de um pod em `default`. Com a aplicação em `Synced/Healthy`, usar port-forward para testar cadastro, chat, persistência de memória após reinício e recuperação semântica.

## 4. Publicar o domínio

- Confirmar DNS para o mesmo destino dos outros serviços e certificado wildcard com condição `Ready`. Criar Ingress Traefik para `ai.italomariano.dev.br` com o Secret `wildcard-italomariano-dev-br` e rotas explícitas.
- Para expor a API, implementar autenticação que derive a conta da identidade autenticada. Separar ou restringir cadastro e vínculo de Telegram, verificar a posse da conta vinculada e limitar a taxa de chamadas ao chat. Validar que requisições anônimas e tentativas de acessar outra conta sejam rejeitadas antes de liberar as rotas. Se a primeira entrega usar apenas Telegram e Finance, publicar somente as duas rotas OAuth quando a integração estiver pronta.
- Verificar DNS, cadeia TLS e resposta HTTPS a partir de fora da rede. Testar também que as rotas internas de saúde, banco e Ollama permanecem inacessíveis publicamente.

## 5. Ativar as integrações

- **Telegram.** Reusar o RabbitMQ do `nous-enqueuer`. A configuração local documentada aponta para `10.42.0.1:5672`, vhost `nous`. Confirmar conectividade do pod, fila `telegram_updates`, formato da mensagem e política de dead-letter. Configurar `RABBITMQ_HOST`, `RABBITMQ_PORT`, `RABBITMQ_VHOST`, `RABBITMQ_USER`, `RABBITMQ_PASSWORD` e o `TELEGRAM_BOT_TOKEN` do mesmo bot do webhook. Configurar SMTP e remetente para novos cadastros. Antes de habilitar o consumidor de produção, verificar mensagens pendentes e encerrar qualquer consumidor local da mesma fila. Testar cadastro ou vínculo e resposta a uma mensagem.
- **Finance.** Configurar um cliente confidencial `ai-finance` no realm `finance` do Keycloak com redirect exato `https://ai.italomariano.dev.br/login/oauth2/code/finance`. Publicar `/finance/oauth/start` e `/login/oauth2/code/finance`, configurar `FINANCE_OAUTH_REDIRECT_URI`, client secret e chave persistente de criptografia dos tokens, então habilitar `FINANCE_OAUTH_ENABLED`. Testar vínculo e consulta ao Finance com uma conta de teste.
- **Brave Search.** Se usada, provisionar `BRAVE_API_KEY` e testar a ferramenta com uma consulta.

## Aceite e rollback

- Aceite quando Argo CD estiver sincronizado e saudável, HTTPS responder nas rotas aprovadas, chat e memória passarem nos testes, e cada integração habilitada funcionar de ponta a ponta. Conferir logs, latência, reinícios e fila após a ativação.
- Para rollback, reverter o commit GitOps para a imagem e configuração anteriores. Em incidentes na primeira implantação, retirar as rotas ou desabilitar o consumidor pelo Git antes de alterar os recursos do banco. Confirmar compatibilidade do esquema antes de voltar a imagem. Se a migração exigir restauração, usar o backup validado e considerar as gravações posteriores ao backup. Preservar o banco externo e a chave de criptografia.

A presença da imagem no GHCR, o DNS, a validade do certificado em execução, o armazenamento e a conectividade dos pods com as demais dependências ainda precisam ser confirmados durante a execução deste plano.
