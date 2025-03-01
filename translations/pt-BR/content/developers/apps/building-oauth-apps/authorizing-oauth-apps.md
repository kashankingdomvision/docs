---
title: Autorizar aplicativos OAuth
intro: '{% data reusables.shortdesc.authorizing_oauth_apps %}'
redirect_from:
  - /apps/building-integrations/setting-up-and-registering-oauth-apps/about-authorization-options-for-oauth-apps
  - /apps/building-integrations/setting-up-and-registering-oauth-apps/directing-users-to-review-their-access
  - /apps/building-integrations/setting-up-and-registering-oauth-apps/creating-multiple-tokens-for-oauth-apps
  - /v3/oauth
  - /apps/building-oauth-apps/authorization-options-for-oauth-apps
  - /apps/building-oauth-apps/authorizing-oauth-apps
  - /developers/apps/authorizing-oauth-apps
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
  ghec: '*'
topics:
  - OAuth Apps
---

A implementação OAuth de {% data variables.product.product_name %} é compatível com o [ tipo de código de autorização padrão](https://tools.ietf.org/html/rfc6749#section-4.1) e com o OAuth 2.0 [Concessão de Autorização do Dispositivo](https://tools.ietf.org/html/rfc8628) para aplicativos que não têm acesso a um navegador web.

Se você desejar ignorar a autorização do seu aplicativo da forma-padrão, como no teste do seu aplicativo, você poderá usar o fluxo do aplicativo [que não é web](#non-web-application-flow).

Para autorizar o seu aplicativo OAuth, considere qual fluxo de autorização melhor se adequa ao seu aplicativo.

- [Fluxo de aplicativos web](#web-application-flow): Usado para autorizar usuários para aplicativos OAuth padrão executados no navegador. (The [implicit grant type](https://tools.ietf.org/html/rfc6749#section-4.2) is not supported.)
- [device flow](#device-flow):  Used for headless apps, such as CLI tools.

## Fluxo do aplicativo web

{% note %}

**Observação:** Se você está criando um aplicativo GitHub, você ainda pode usar o fluxo do aplicativo web OAuth, mas a configuração tem algumas diferenças importantes. Consulte "[Identificando e autorizando usuários para aplicativos GitHub](/apps/building-github-apps/identifying-and-authorizing-users-for-github-apps/)" para obter mais informações.

{% endnote %}

O fluxo do aplicativo web para autorizar os usuários para seu aplicativo é:

1. Os usuários são redirecionados para solicitar sua identidade do GitHub
2. Os usuários são redirecionados de volta para o seu site pelo GitHub
3. Seu aplicativo acessa a API com o token de acesso do usuário

### 1. Solicitar identidade do GitHub de um usuário

    GET {% data variables.product.oauth_host_code %}/login/oauth/authorize

Quando seu aplicativo GitHub especifica um parâmetro do `login`, ele solicita aos usuários com uma conta específica que podem usar para iniciar sessão e autorizar seu aplicativo.

#### Parâmetros

| Nome           | Tipo     | Descrição                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| -------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `client_id`    | `string` | **Obrigatório**. O ID do cliente que você recebeu do GitHub quando você {% ifversion fpt or ghec %}[fez o cadastro](https://github.com/settings/applications/new){% else %}registrados{% endif %}.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `redirect_uri` | `string` | A URL no seu aplicativo para o qual os usuários serão enviados após a autorização. Veja os detalhes abaixo sobre [redirecionamento das urls](#redirect-urls).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| `login`        | `string` | Sugere uma conta específica para iniciar a sessão e autorizar o aplicativo.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `escopo`       | `string` | Uma lista de [escopos](/apps/building-oauth-apps/understanding-scopes-for-oauth-apps/) delimitada por espaço. Caso não seja fornecido, o `escopo`-padrão será uma lista vazia para usuários que não autorizaram nenhum escopo para o aplicativo. Para usuários que têm escopos autorizados para o aplicativo, a página de autorização OAuth com a lista de escopos não será exibida para o usuário. Em vez disso, esta etapa do fluxo será concluída automaticamente com o conjunto de escopos que o usuário autorizou para o aplicativo. Por exemplo, se um usuário já executou o fluxo web duas vezes e autorizou um token com escopo do `usuário` e outro token com o escopo do `repositório`, um terceiro fluxo web que não fornece um escopo `` receberá um token com os escopos do `usuário` e do `repositório`. |
| `estado`       | `string` | {% data reusables.apps.state_description %}
| `allow_signup` | `string` | Independentemente de os usuários serem autenticados, eles receberão uma opção para inscrever-se no GitHub durante o fluxo do OAuth. O padrão é `verdadeiro`. Use `falso` quando uma política proibir inscrições.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

### 2. Os usuários são redirecionados de volta para o seu site pelo GitHub

Se o usuário aceitar a sua solicitação, o {% data variables.product.product_name %} redireciona de volta para seu site com `código` temporário em um parâmetro de código, bem como o estado que você forneceu na etapa anterior em um `parâmetro de estado`. O código temporário irá expirar após 10 minutos. Se os estados não corresponderem, significa que uma terceira criou a solicitação, e você deverá abortar o processo.

Troque este `código` por um token de acesso:

    POST {% data variables.product.oauth_host_code %}/login/oauth/access_token

#### Parâmetros

| Nome            | Tipo     | Descrição                                                                                                                                                    |
| --------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `client_id`     | `string` | **Obrigatório.** O ID do cliente que você recebeu do {% data variables.product.product_name %} para o seu {% data variables.product.prodname_oauth_app %}. |
| `client_secret` | `string` | **Obrigatório.** O segredo do cliente que recebeu do {% data variables.product.product_name %} para o seu {% data variables.product.prodname_oauth_app %}. |
| `código`        | `string` | **Obrigatório.** O código que você recebeu como resposta ao Passo 1.                                                                                         |
| `redirect_uri`  | `string` | A URL do seu aplicativo para onde os usuários são enviados após a autorização.                                                                               |

#### Resposta

Por padrão, a resposta assume o seguinte formato:

```
access_token={% ifversion fpt or ghes > 3.1 or ghae or ghec %}gho_16C7e42F292c6912E7710c838347Ae178B4a{% else %}e72e16c7e42f292c6912e7710c838347ae178b4a{% endif %}&scope=repo%2Cgist&token_type=bearer
```

{% data reusables.apps.oauth-auth-vary-response %}

```json
Accept: application/json
{
  "access_token":"{% ifversion fpt or ghes > 3.1 or ghae or ghec %}gho_16C7e42F292c6912E7710c838347Ae178B4a{% else %}e72e16c7e42f292c6912e7710c838347ae178b4a{% endif %}",
  "scope":"repo,gist",
  "token_type":"bearer"
}
```

```xml
Accept: application/xml
<OAuth>
  <token_type>bearer</token_type>
  <scope>repo,gist</scope>
  <access_token>{% ifversion fpt or ghes > 3.1 or ghae or ghec %}gho_16C7e42F292c6912E7710c838347Ae178B4a{% else %}e72e16c7e42f292c6912e7710c838347ae178b4a{% endif %}</access_token>
</OAuth>
```

### 3. Use o token de acesso para acessar a API

O token de acesso permite que você faça solicitações para a API em nome de um usuário.

    Autorização: token OUTH-TOKEN
    GET {% data variables.product.api_url_code %}/user

Por exemplo, no cURL você pode definir o cabeçalho de autorização da seguinte forma:

```shell
curl -H "Authorization: token OAUTH-TOKEN" {% data variables.product.api_url_pre %}/user
```

## Fluxo de dispositivo

{% note %}

**Nota:** O fluxo do dispositivo está na versão beta pública e sujeito a alterações.

{% endnote %}

O fluxo de dispositivos permite que você autorize usuários para um aplicativo sem cabeçalho, como uma ferramenta de CLI ou um gerenciador de credenciais do Git.

### Visão geral do fluxo do dispositivo

1. O seu aplicativo solicita o dispositivo e o código de verificação do usuário e obtém a URL de autorização em que o usuário digitará o código de verificação do usuário.
2. O aplicativo solicita que o usuário insira um código de verificação em {% data variables.product.device_authorization_url %}.
3.  O aplicativo pesquisa status de autenticação do usuário. Uma vez que o usuário tenha autorizado o dispositivo, o aplicativo poderá fazer chamadas de API com um novo token de acesso.

### Passo 1: O aplicativo solicita o dispositivo e os códigos de verificação de usuário do GitHub

    POST {% data variables.product.oauth_host_code %}/login/device/code

Your app must request a user verification code and verification URL that the app will use to prompt the user to authenticate in the next step. This request also returns a device verification code that the app must use to receive an access token and check the status of user authentication.

#### Parâmetros de entrada

| Nome        | Tipo     | Descrição                                                                                                             |
| ----------- | -------- | --------------------------------------------------------------------------------------------------------------------- |
| `client_id` | `string` | **Obrigatório.** O ID do cliente que você recebeu do {% data variables.product.product_name %} para o seu aplicativo. |
| `escopo`    | `string` | O escopo ao qual o seu aplicativo está solicitando acesso.                                                            |

#### Resposta

Por padrão, a resposta assume o seguinte formato:

```
device_code=3584d83530557fdd1f46af8289938c8ef79f9dc5&expires_in=900&interval=5&user_code=WDJB-MJHT&verification_uri=https%3A%2F%{% data variables.product.product_url %}%2Flogin%2Fdevice
```

{% data reusables.apps.oauth-auth-vary-response %}

```json
Accept: application/json
{
  "device_code": "3584d83530557fdd1f46af8289938c8ef79f9dc5",
  "user_code": "WDJB-MJHT",
  "verification_uri": "{% data variables.product.oauth_host_code %}/login/device",
  "expires_in": 900,
  "interval": 5
}
```

```xml
Accept: application/xml
<OAuth>
  <device_code>3584d83530557fdd1f46af8289938c8ef79f9dc5</device_code>
  <user_code>WDJB-MJHT</user_code>
  <verification_uri>{% data variables.product.oauth_host_code %}/login/device</verification_uri>
  <expires_in>900</expires_in>
  <interval>5</interval>
</OAuth>
```

#### Parâmetros de resposta

| Nome               | Tipo      | Descrição                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ------------------ | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `device_code`      | `string`  | O código de verificação do dispositivo tem 40 caracteres e é usado para verificar o dispositivo.                                                                                                                                                                                                                                                                                                                                                                               |
| `user_code`        | `string`  | O código de verificação do usuário é exibido no dispositivo para que o usuário possa inserir o código no navegador. Este código tem 8 caracteres com um hífen no meio.                                                                                                                                                                                                                                                                                                         |
| `verification_uri` | `string`  | A URL de verificação em que os usuários devem digitar o código do usuário ``: {% data variables.product.device_authorization_url %}.                                                                                                                                                                                                                                                                                                                                         |
| `expires_in`       | `inteiro` | O número de segundos antes dos códigos `device_code` e `user_code` expirarem. O padrão é 900 segundos ou 15 minutos.                                                                                                                                                                                                                                                                                                                                                           |
| `interval`         | `inteiro` | O número mínimo de segundos que decorridos antes de você poder fazer uma nova solicitação de token de acesso (`POST {% data variables.product.oauth_host_code %}/login/oauth/access_token`) para concluir a autorização do dispositivo. Por exemplo, se o intervalo for 5, você não poderá fazer uma nova solicitação a partir de 5 segundos. Se você fizer mais de uma solicitação em 5 segundos, você atingirá o limite de taxa e receberá uma mensagem de erro `slow_down`. |

### Passo 2: Solicite ao usuário que insira o código do usuário em um navegador

Your device will show the user verification code and prompt the user to enter the code at {% data variables.product.device_authorization_url %}.

  ![Field to enter the user verification code displayed on your device](/assets/images/github-apps/device_authorization_page_for_user_code.png)

### Passo 3: O aplicativo solicita que o GitHub verifique se o usuário autorizou o dispositivo

    POST {% data variables.product.oauth_host_code %}/login/oauth/access_token

Your app will make device authorization requests that poll `POST {% data variables.product.oauth_host_code %}/login/oauth/access_token`, until the device and user codes expire or the user has successfully authorized the app with a valid user code. The app must use the minimum polling `interval` retrieved in step 1 to avoid rate limit errors. For more information, see "[Rate limits for the device flow](#rate-limits-for-the-device-flow)."

The user must enter a valid code within 15 minutes (or 900 seconds). After 15 minutes, you will need to request a new device authorization code with `POST {% data variables.product.oauth_host_code %}/login/device/code`.

Once the user has authorized, the app will receive an access token that can be used to make requests to the API on behalf of a user.

#### Parâmetros de entrada

| Nome          | Tipo     | Descrição                                                                                                                                                             |
| ------------- | -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `client_id`   | `string` | **Obrigatório.** O ID do cliente que você recebeu do {% data variables.product.product_name %} para o seu {% data variables.product.prodname_oauth_app %}.          |
| `device_code` | `string` | **Obrigatório.** O código de verificação do dispositivo que você recebeu da solicitação `POST {% data variables.product.oauth_host_code %}/login/dispositivo/código`. |
| `grant_type`  | `string` | **Obrigatório.** O tipo de concessão deve ser `urn:ietf:params:oauth:grant-type:device_code`.                                                                         |

#### Resposta

Por padrão, a resposta assume o seguinte formato:

```
access_token={% ifversion fpt or ghes > 3.1 or ghae or ghec %}gho_16C7e42F292c6912E7710c838347Ae178B4a{% else %}e72e16c7e42f292c6912e7710c838347ae178b4a{% endif %}&token_type=bearer&scope=repo%2Cgist
```

{% data reusables.apps.oauth-auth-vary-response %}

```json
Accept: application/json
{
 "access_token": "{% ifversion fpt or ghes > 3.1 or ghae or ghec %}gho_16C7e42F292c6912E7710c838347Ae178B4a{% else %}e72e16c7e42f292c6912e7710c838347ae178b4a{% endif %}",
  "token_type": "bearer",
  "scope": "repo,gist"
}
```

```xml
Accept: application/xml
<OAuth>
  <access_token>{% ifversion fpt or ghes > 3.1 or ghae or ghec %}gho_16C7e42F292c6912E7710c838347Ae178B4a{% else %}e72e16c7e42f292c6912e7710c838347ae178b4a{% endif %}</access_token>
  <token_type>bearer</token_type>
  <scope>gist,repo</scope>
</OAuth>
```

### Limites de taxa para o fluxo do dispositivo

When a user submits the verification code on the browser, there is a rate limit of 50 submissions in an hour per application.

If you make more than one access token request (`POST {% data variables.product.oauth_host_code %}/login/oauth/access_token`) within the required minimum timeframe between requests (or `interval`), you'll hit the rate limit and receive a `slow_down` error response. The `slow_down` error response adds 5 seconds to the last `interval`. For more information, see the [Errors for the device flow](#errors-for-the-device-flow).

### Códigos de erro para o fluxo do dispositivo

| Código do erro                 | Descrição                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `authorization_pending`        | Este erro ocorre quando a solicitação de autorização está pendente e o usuário ainda não inseriu o código do usuário. Espera-se que o aplicativo continue fazendo a sondagem da solicitação `POST {% data variables.product.oauth_host_code %}/login/oauth/oaccess_token` sem exceder o [`intervalo`](#response-parameters), que exige um número mínimo de segundos entre cada solicitação.                                                                                                                                                                                |
| `slow_down`                    | Ao receber o erro `slow_down`, são adicionados 5 segundos extras ao intervalo mínimo `` ou período de tempo necessário entre as suas solicitações usando `POST {% data variables.product.oauth_host_code %}/login/oauth/oaccess_token`. Por exemplo, se o intervalo inicial for necessário pelo menos 5 segundos entre as solicitações e você receber uma resposta de erro de `slow_down`, você deverá aguardar pelo menos 10 segundos antes de fazer uma nova solicitação para um token de acesso OAuth. A resposta de erro inclui o novo `intervalo` que você deve usar. |
| `expired_token`                | Se o código do dispositivo expirou, você verá o erro `token_expired`. Você deve fazer uma nova solicitação para um código de dispositivo.                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `unsupported_grant_type`       | O tipo de concessão deve ser `urn:ietf:params:oauth:grant-type:device_code` e incluído como um parâmetro de entrada quando você faz a sondagem da solicitação do token do OAuth `POST {% data variables.product.oauth_host_code %}/login/oauth/oaccess_token`.                                                                                                                                                                                                                                                                                                             |
| `incorrect_client_credentials` | Para o fluxo do dispositivo, você deve passar o ID de cliente do aplicativo, que pode ser encontrado na página de configurações do aplicativo. O `client_secret` não é necessário para o fluxo do dispositivo.                                                                                                                                                                                                                                                                                                                                                             |
| `incorrect_device_code`        | O device_code fornecido não é válido.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `access_denied`                | Quando um usuário clica em cancelar durante o processo de autorização, você receberá uma mensagem de erro de `access_denied` e o usuário não poderá usar o código de verificação novamente.                                                                                                                                                                                                                                                                                                                                                                                |

For more information, see the "[OAuth 2.0 Device Authorization Grant](https://tools.ietf.org/html/rfc8628#section-3.5)."

## Fluxo do aplicativo que não são da web

Non-web authentication is available for limited situations like testing. If you need to, you can use [Basic Authentication](/rest/overview/other-authentication-methods#basic-authentication) to create a personal access token using your [Personal access tokens settings page](/articles/creating-an-access-token-for-command-line-use). This technique enables the user to revoke access at any time.

{% ifversion fpt or ghes or ghec %}
{% note %}

**Note:** When using the non-web application flow to create an OAuth2 token, make sure to understand how to [work with two-factor authentication](/rest/overview/other-authentication-methods#working-with-two-factor-authentication) if you or your users have two-factor authentication enabled.

{% endnote %}
{% endif %}

## URLs de redirecionamento

The `redirect_uri` parameter is optional. If left out, GitHub will redirect users to the callback URL configured in the OAuth Application settings. If provided, the redirect URL's host and port must exactly match the callback URL. The redirect URL's path must reference a subdirectory of the callback URL.

    RETORNO DE CHAMADA: http://example.com/path
    
    GOOD: http://example.com/path
    GOOD: http://example.com/path/subdir/other
    BAD:  http://example.com/bar
    BAD:  http://example.com/
    BAD:  http://example.com:8080/path
    BAD:  http://oauth.example.com:8080/path
    BAD:  http://example.org

### URLs de redirecionamento do Localhost

The optional `redirect_uri` parameter can also be used for localhost URLs. If the application specifies a localhost URL and a port, then after authorizing the application users will be redirected to the provided URL and port. The `redirect_uri` does not need to match the port specified in the callback url for the app.

For the `http://127.0.0.1/path` callback URL, you can use this `redirect_uri`:

```
http://127.0.0.1:1234/path
```

## Criar vários tokens para aplicativos OAuth

You can create multiple tokens for a user/application/scope combination to create tokens for specific use cases.

This is useful if your OAuth App supports one workflow that uses GitHub for sign-in and only requires basic user information. Another workflow may require access to a user's private repositories. Using multiple tokens, your OAuth App can perform the web flow for each use case, requesting only the scopes needed. If a user only uses your application to sign in, they are never required to grant your OAuth App access to their private repositories.

{% data reusables.apps.oauth-token-limit %}

{% data reusables.apps.deletes_ssh_keys %}

## Direcionar os usuários para revisar seus acessos

You can link to authorization information for an OAuth App so that users can review and revoke their application authorizations.

To build this link, you'll need your OAuth Apps `client_id` that you received from GitHub when you registered the application.

```
{% data variables.product.oauth_host_code %}/settings/connections/applications/:client_id
```

{% tip %}

**Tip:** To learn more about the resources that your OAuth App can access for a user, see "[Discovering resources for a user](/rest/guides/discovering-resources-for-a-user)."

{% endtip %}

## Solução de Problemas

* "[Solucionando erros de solicitação de autorização](/apps/managing-oauth-apps/troubleshooting-authorization-request-errors)"
* "[Troubleshooting OAuth App access token request errors](/apps/managing-oauth-apps/troubleshooting-oauth-app-access-token-request-errors)"
* "[Device flow errors](#error-codes-for-the-device-flow)"{% ifversion fpt or ghae-issue-4374 or ghes > 3.2 or ghec %}
* "[Vencimento e revogação do Token](/github/authenticating-to-github/keeping-your-account-and-data-secure/token-expiration-and-revocation)"{% endif %}

## Leia mais

- "[Sobre a autenticação em {% data variables.product.prodname_dotcom %}](/github/authenticating-to-github/about-authentication-to-github)"
