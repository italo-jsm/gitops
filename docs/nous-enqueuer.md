# Publicação do nous-enqueuer

O Argo CD gerencia Deployment, Service ClusterIP e Ingress do nous-enqueuer no
namespace `default`. RabbitMQ continua fora do Kubernetes em um container
Docker isolado com volume persistente. O enqueuer conecta ao broker pelo endereço
privado informado em `nous-enqueuer-secrets`; o Traefik não acessa o broker.
O Ingress encaminha somente `nous.italomariano.dev.br/telegram/webhook` para o
Service, sem reescrever o path. Actuator e RabbitMQ não são expostos pelo Ingress.

## Pré-requisitos para sincronizar

- Substitua a referência marcada em `deployment.yaml` pela tag imutável SHA ou
  digest publicado pelo CI. Não sincronize a referência `SET-BEFORE-ROLLOUT`.
- Provisione `ghcr-credentials` e `nous-enqueuer-secrets` no namespace
  `default` por mecanismo externo ao Git. O Secret de runtime deve conter
  `BOT_ID`, `WEBHOOK_SECRET`, `ALLOWED_SENDER_IDS`, `RABBITMQ_HOST`,
  `RABBITMQ_PORT`, `RABBITMQ_VHOST`, `RABBITMQ_USERNAME` e `RABBITMQ_PASSWORD`.
  Configure `RABBITMQ_USERNAME` como `nous_enqueuer` para corresponder à conta
  provisionada pelo inicializador do broker.
- Configure `RABBITMQ_HOST` com o IP ou DNS privado estável do host Docker onde
  o broker isolado escuta em `5672`. Confirme conectividade a partir da rede de
  pods e da AI, e limite o firewall às origens privadas necessárias. Use TLS
  AMQP se essa rede não for confiável.
- Confirme que `wildcard-italomariano-dev-br` existe no namespace `default`, que
  o DNS aponta para a entrada pública do Traefik e que a porta 443 chega ao cluster.

## Validação e sincronização

Valide os YAMLs e referências de Service/porta/namespace. Com o contexto correto,
execute:

```sh
kubectl apply --dry-run=server -f applications/nous-enqueuer/
kubectl apply --dry-run=server -f bootstrap/nous-enqueuer.yaml
```

O arquivo `bootstrap/nous-enqueuer.yaml` cria uma aplicação Argo CD com sync
automático, prune e self-heal. O app-of-apps acompanha `bootstrap` em `main`,
portanto publicar esse commit em `main` ativa a sincronização. Após confirmar os
pré-requisitos, verifique `Synced` e `Healthy`, os dois pods Ready e o acesso ao
RabbitMQ pelo endpoint de readiness. Teste o webhook pelo DNS e TLS públicos.

## Rollback

Para remover a rota e o nous-enqueuer, reverta o commit que adicionou o Application ao
`bootstrap` ou remova esse Application pelo Argo CD. O prune automático remove
Ingress, Service, Deployment e ConfigMap. Isso não para nem altera o container
RabbitMQ externo, seu volume ou seus dados.
