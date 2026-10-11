# Login IM no Keycloak

O tema `im` é distribuído por `applications/auth/login-theme-configmap.yaml`
e montado como volume somente leitura em `/opt/keycloak/themes/im/login`.
Estende o tema nativo `keycloak` da versão 26.6.3, preservando validação,
recuperação de senha e MFA. Não requer outra imagem Docker.

## Publicação e ativação

1. Faça commit e push das alterações deste repositório.
2. Aguarde o Argo CD sincronizar a aplicação `auth` e o rollout do Keycloak.
3. No console do Keycloak, abra o realm utilizado pelo front e selecione
   **Clients → cliente do front → Settings → Login theme → im**. Salve.
   Para usar em todo o realm, selecione **Realm settings → Themes → Login theme → im**.
4. Em janela anônima, verifique login, senha incorreta, logout e os fluxos
   de recuperação/MFA habilitados.

O ConfigMap instala os arquivos, mas não altera a configuração persistida
no banco do Keycloak. A seleção do tema no cliente/realm é necessária.

## Atualizações

Edite os arquivos dentro de `data` no ConfigMap. O favicon IM está em
`binaryData.favicon.ico`, codificado em base64, e é montado em
`resources/img/favicon.ico`, substituindo o ícone padrão do Keycloak.
Atualize também a anotação
`checksum/login-theme` no template do deployment para disparar um rollout
e invalidar o cache do tema. O checksum é o SHA-256 do objeto
`{"data": ..., "binaryData": ...}` serializado como JSON pelo Python
(`json.dumps(payload, sort_keys=True)`). Inclua as duas seções ao calculá-lo.

Não é necessário gerar uma imagem Docker: commit e push neste repositório
disparam a sincronização automática configurada em `bootstrap/auth.yaml`.
Se a aba ainda mostrar o ícone antigo, confira em uma janela anônima.

A origem da logo e do tema está no repositório `finance-front`, em
`public/im-logo.svg` e `keycloak/themes/im/login`.

## Reversão

Remova a seleção `im` do cliente/realm antes de retirar o ConfigMap e o
volume. Isso restaura o tema herdado sem alterar usuários ou credenciais.

Referência: https://www.keycloak.org/ui-customization/themes
