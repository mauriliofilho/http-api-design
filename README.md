# Guia para Projeto de API HTTP

## Introdução

Este guia descreve uma série de boas práticas para o projeto de APIs HTTP+JSON, originalmente extraido de um trabalho na [API da Plataforma Heroku](https://devcenter.heroku.com/articles/platform-api-reference).

Este guia inclue adições a essa API e serve de guia para novas APIs Internas no Heroku. Esperamos que seja de interesse de outros que projetam APIs que não são do Heroku.

Nosso objetivo aqui são consistencia e foco nas regras de negocios evitando coisas supérfluas. Procuramos um jeito _bom, consistente e bem docuemntado_ de projetar APIs, não necessariamente _o método único/ideal_.

Assumimos que você já esteja farmiliarizado com o básico de APIs HTTP+JSON APIs e não cobriremos todos os fundamentos disto nesse guia.

Agradecemos [contribuições](CONTRIBUTING.md) a esse guia.

## Conteúdo

* [Fundamentos](#foundations)
  *  [Separe as Responsabilidades](#separate-concerns)
  *  [Exija Conexões Seguras](#require-secure-connections)
  *  [Exija Versionamento no Cabeçalho Accepts](#require-versioning-in-the-accepts-header)
  *  [Suporte ETags para Cacheamento](#support-etags-for-caching)
  *  [Forneça Request-Ids para Introspecção](#provide-request-ids-for-introspection)
  *  [Divida Respostas Longas Entre Requisições com Ranges](#divide-large-responses-across-requests-with-ranges)
* [Requisições](#requests)
  *  [Aceite JSON serializado no corpo das requisições](#accept-serialized-json-in-request-bodies)
  *  [Use formatos de rotas consistentes](#use-consistent-path-formats)
    *  [Rotas e atributos em letras minúsculas](#downcase-paths-and-attributes)
    *  [Suporte referencia com atributos que não sejam ID por conveniencia](#support-non-id-dereferencing-for-convenience)
    *  [Minimize a profundidade da rota](#minimize-path-nesting)
* [Respostas](#responses)
  *  [Retorne o código de estado apropriado](#return-appropriate-status-codes)
  *  [Prover recursos completos quando disponivel](#provide-full-resources-where-available)
  *  [Prover os (UU)IDs dos recursos](#provide-resource-uuids)
  *  [Prover timestamps padrões](#provide-standard-timestamps)
  *  [Use horários UTC em formato ISO8601](#use-utc-times-formatted-in-iso8601)
  *  [Aninhe as relações de chaves estrangeiras](#nest-foreign-key-relations)
  *  [Gere erros estruturados](#generate-structured-errors)
  *  [Mostre o estado limite de requisições](#show-rate-limit-status)
  *  [Manter o JSON minificado em todas as respostas](#keep-json-minified-in-all-responses)
* [Artefatos](#artifacts)
  *  [Prover um esquema JSON processavel](#provide-machine-readable-json-schema)
  *  [Prover documentação para leitura](#provide-human-readable-docs)
  *  [Prover exemplos executaveis](#provide-executable-examples)
  *  [Descreva a estabilidade](#describe-stability)
* [Traduções](#translations)

### Fundamentos

#### Separe as Responsabilidades

Mantenha as coisas simples quando for projetar separando as responsabilidades entre partes diferentes do ciclo de requisição e resposta. Mantendo regras simples aqui permite um foco maior em problemas maiores e mais complexos.

Requisições e respostas serão feitas para se dirigir a um recurso em particular ou coleção. Use o caminho para indicar a identidade, o corpo para transferir o conteúdo e os cabeçalhos para indicar metadados. Parametros podem ser usado como um meio de passar informações de cabeçalhos em casos extremos, mas cabeçalhos são preferidos já que são mais flexiveis e podem transmitir informações mais diversas.

#### Exija Conexões Seguras

Exija conexões seguras com TLS para acessar a API, sem excessões.
Não vale a pena ficar imaginando ou explicando quando se deve usar TLS e quando não. Simplesmente exija TLS para tudo.

O ideal é simplesmente rejeitar qualquer requisição não TLS não respondendo a requisição http ou na porta 80 para evitar troca de dados inseguros. Em ambientes que isto não é possivel, responda com `403 Forbidden`.

Redirecionamentos são desencorajados uma vez que eles permitem um comportamento incorreto do cliente sem oferecer nenhum ganho. Clientes que dependem de redirecionamentos dobram o tráfego do servidor e fazem com que o TLS seja inutil uma vez que dados sensiveis já terão sido expostos na primeira requisição.

#### Exija Versionamento no Cabeçalho Accepts

Versionamento e a transição entre versões podem ser um dos aspectos mais desafiadores de projetar e manter uma API. Por isso, é melhor empregar mecanismos para facilitar isto desde o começo.

Para prevenir surpresas, indisponibilidade para o usuário, é melhor exigir que a versão seja especificada com todas as requisições. Versões padrões devem ser evitadas já que, no melhor dos casos, são muito dificeis de mudar no futuro.

É melhor enviar especificação da versão no cabeçalho, com outros metadados, usando o cabeçalho `Accept` com um _content type_ personalizado, por exemplo:

```
Accept: application/vnd.heroku+json; version=3
```

#### Suporte ETags para Cacheamento

Inclua um cabeçalho `ETag` em todas as respostas, identificando a versão especifica do recurso retornado. Isto permite aos usuários cachear os recursos e usar requisições com este valor no cabeçalho `If-None-Match` para determinar se o cache deve ser atualizado.

#### Forneça Request-Ids para Introspecção

Inclua um cabeçalho `Request-Id` em cada resposta da API, populada com um valor UUID. Registrando esses valores no cliente, servidores e qualquer serviço adicional, oferece um mecanismo para rastrear, diagnosticar e depurar requisições.

#### Divida Respostas Longas Entre Requisições com Ranges

Respostas grandes devem ser quebradas entre multiplas requisições usando o cabeçalho `Range` para especificar quando mais dados estão disponiveis e como obter eles. Veja [Heroku Platform API discussion of Ranges](https://devcenter.heroku.com/articles/platform-api-reference#ranges) para os detalhes da requisição e dos cabeçalhos de respostas, códigos de estado, limites, organização e iteração.

### Requisições

#### Aceite JSON serializado no corpo das requisições

Aceite JSON serializado no corpo das requisiçõe `PUT`/`PATCH`/`POST` além de ou em conjunto a dados form-encoded. Isto cria uma simetria com o corpo das requisições JSON serializado, p.e.:

```bash
$ curl -X POST https://service.com/apps \
    -H "Content-Type: application/json" \
    -d '{"name": "demoapp"}'

{
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "name": "demoapp",
  "owner": {
    "email": "username@example.com",
    "id": "01234567-89ab-cdef-0123-456789abcdef"
  },
  ...
}
```

#### Use formatos de rotas consistentes

##### Nomes de recursos

Use a versão pluralizada de um nome de recurso a menos que o recurso em questão seja unico no sistema (por exemplo, na maioria dos sistemas, um dado usuario deve ter somente uma conta). Isto permite consistencia na forma que você se refere a um recurso em particular.

##### Ações

Prefira endpoint que não precisem de quaisquer ações especiais para recursos individuais. Em casos onde ações especiais são necessárias, coloque-as sobre um prefixo `actions` padrão, para diferencia-los com clareza:

```
/resources/:resource/actions/:action
```

p.e.

```
/runs/{run_id}/actions/stop
```

#### Rotas e atributos em letras minúsculas

Use rotas com letras minusculas e separadas por hifen, para que seja igual a nomes de dominio, p.e.:

```
service-api.com/users
service-api.com/app-setups
```

Também utilize letras minusculas para os atributos, mas use sublinhado como separador pois assim nomes de atributos podem ser digitados sem aspas em JavaScript, p.e.:

```
service_class: "first"
```

#### Suporte referencia com atributos que não sejam ID por conveniencia

Em alguns casos pode ser incoveniente para o usuario final oferecer IDs para identificar um recurso. Por exemplo, um usuario pode pensar no nome de Apps no Heroku, mas este app pode ser identificado por um UUID. Nestes casos você poderia aceitar ambos, p.e.:

```bash
$ curl https://service.com/apps/{app_id_or_name}
$ curl https://service.com/apps/97addcf0-c182
$ curl https://service.com/apps/www-prod
```

Não aceite somente nomes omitindo IDs.

#### Minimize a profundidade da rota

Em alguns modelos de dados com relacionamento de recursos pai/filho, as rotas podem acabar ficando muito profundas, p.e.:

```
/orgs/{org_id}/apps/{app_id}/dynos/{dyno_id}
```

Limite a profundidade excessiva preferindo localizar recursos na raiz da rota. Use nesting depth by preferring to locate resources at the root
path. Use aninhamento para indicarcoleções. Por exemplo, para o caso acima em que um _dyno_ pertence a um app, que pertence a uma organização:

```
/orgs/{org_id}
/orgs/{org_id}/apps
/apps/{app_id}
/apps/{app_id}/dynos
/dynos/{dyno_id}
```

### Respostas

#### Retorne o código de estado apropriado

Retorne o código de estado HTTP adequado com cada resposta. Respostas com exito devem retornar códigos de acordo com este guia:

* `200`: Requisição com exito para um pedido `GET`, `DELETE` ou
  `PATCH` que se completou de forma sincrona, ou para um pedido `PUT` que atualizou um recurso existente de forma sincrona
* `201`: Requisição com exito para um pedido `POST` que completou de forma sincrona, ou para um pedido `PUT` que completou de forma sincrona a criação de um novo recurso
* `202`: Requisição aceita para um pedido `POST`, `PUT`, `DELETE`, or `PATCH` call que será processada de forma assincrona
* `206`: Requisição com exito para um pedido `GET`, mas que retorna somente uma resposta parcial: veja acima sobre [respostas longas](#divide-large-responses-across-requests-with-ranges)

Preste atenção ao uso de códigos de erros para autenticação e autorização:

* `401 Unauthorized`: Requisição falhou por que o usuario não está autenticado
* `403 Forbidden`: Requisição falhou por que o usuario não tem autorização para acessar o recurso especifico

Retorne códigos adequadospara prover informações adicionais quando há erros:

* `422 Unprocessable Entity`: Sua requisição foi entendida, mas contém parametros inválidos
* `429 Too Many Requests`: Você superou o limete de consumo, tente novamente mais tarde
* `500 Internal Server Error`: Algo deu errado com o Servidor, confira o site do estado e/ou notifique o problema

Consulte a [Especificação de códigos de respostas HTTP](https://tools.ietf.org/html/rfc7231#section-6)
par um guia sobre código de estado para erros de usuario e servidor.

#### Prover recursos completos quando disponivel

Disponibilize a representação completa do recurso (p.e. o objeto com todos os atributos) quando possivel na resposta. Sempre disponibilize o recurso completo em respostas 200 e 201, includindo requisições `PUT`/`PATCH` e `DELETE` p.e.:

```bash
$ curl -X DELETE \  
  https://service.com/apps/1f9b/domains/0fd4

HTTP/1.1 200 OK
Content-Type: application/json;charset=utf-8
...
{
  "created_at": "2012-01-01T12:00:00Z",
  "hostname": "subdomain.example.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Respostas 202 não incluirão a representação total do recurso, p.e.:

```bash
$ curl -X DELETE \  
  https://service.com/apps/1f9b/dynos/05bd

HTTP/1.1 202 Accepted
Content-Type: application/json;charset=utf-8
...
{}
```

#### Prover os (UU)IDs dos recursos

Dê a cada recurso um atributo `id` por padrão. Use UUIDs a menos que você tenham uma razão muito boa para não fazê-lo. Não use IDs que não são unicos globalmente entre instancias de serviços ou outros recursos no serviço, especialmente IDs com auto incremento.

Mostre os UUIDs em letras minusculas no formato `8-4-4-4-12`, p.e.:

```
"id": "01234567-89ab-cdef-0123-456789abcdef"
```

#### Prover timestamps padrões

Disponibilize timestamps `created_at` e `updated_at` para recursos por padrão, p.e.:

```javascript
{
  // ...
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T13:00:00Z",
  // ...
}
```

Estes timestamps talvez não façam sentido para alguns recursos, neste caso, podem ser omitidos.

#### Use horários UTC em formato ISO8601

Aceite e retorne horários somente em UTC. Mostre horarios em formato ISO8601,
p.e.:

```
"finished_at": "2012-01-01T12:00:00Z"
```

#### Aninhe as relações de chaves estrangeiras

Serialize as referencias a chaves estrangeiras com um objeto aninhado, p.e.:

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0..."
  },
  // ...
}
```

Ao invés de p.e.:

```javascript
{
  "name": "service-production",
  "owner_id": "5d8201b0...",
  // ...
}
```

Este método possibilita mostrar mais informações sobre um recurso sem ter de mudar a estrutura de resposta ou introduzir mais campos de de resposta de alto nivel p.e.:

```javascript
{
  "name": "service-production",
  "owner": {
    "id": "5d8201b0...",
    "name": "Alice",
    "email": "alice@heroku.com"
  },
  // ...
}
```

#### Gere erros estruturados

Gere corpos de respostas de erros estruturados e consistentes. Inclua um `id`  legivel para máquinas, uma `message` de erro legivel para humanos e opcionalmente uma `url` que aponte para mais informações sobre o erro e como resolver isto, p.e.:

```
HTTP/1.1 429 Too Many Requests
```

```json
{
  "id":      "rate_limit",
  "message": "Account reached its API rate limit.",
  "url":     "https://docs.service.com/rate-limits"
}
```

Documente seus formatos de erros e possiveis `id`s de erros que os clientes possam encontrar.

#### Mostre o estado limite de requisições

Limite as requisições dos clientes para proteger o seu serviço e manter um serviço de alta qualidade  para outros cllientes. Voc6e pode usar um
[algoritimo de token bucket](http://en.wikipedia.org/wiki/Token_bucket) para calcular o limite de respostas.

Retorne o número restante de requisições no cabeçalho de resposta `RateLimit-Remaining`.

#### Manter o JSON minificado em todas as respostas

Espaços extras adicionam tamanho extra as respostas, e muitos clientes que mostram os dados para humanos "embelezam" a saída em JSON. É melhor manter as respostas JSON minificadas, p.e.:

```json
{"beta":false,"email":"alice@heroku.com","id":"01234567-89ab-cdef-0123-456789abcdef","last_login":"2012-01-01T12:00:00Z","created_at":"2012-01-01T12:00:00Z","updated_at":"2012-01-01T12:00:00Z"}
```

Ao invés de p.e.:

```json
{
  "beta": false,
  "email": "alice@heroku.com",
  "id": "01234567-89ab-cdef-0123-456789abcdef",
  "last_login": "2012-01-01T12:00:00Z",
  "created_at": "2012-01-01T12:00:00Z",
  "updated_at": "2012-01-01T12:00:00Z"
}
```

Você pode, opcionalmente, prover um jeito para os clietnes de receber uma respostas mais _verbosa_, tanto via um parametro query (p.e. `?pretty=true`)
como via um parametro do cabeçalho `Accept` (e.g.
`Accept: application/vnd.heroku+json; version=3; indent=4;`).

### Artefatos

#### Prover um esquema JSON processavel

Disponibilize um esquema processavel para especificar sua API. Use
[prmd](https://github.com/interagent/prmd) para gerenciar o seu esquema e asegure que está validado com `prmd verify`.

#### Prover documentação para leitura

Disponibilize uma documentação para que os desenvolvedores possam entender a sua API.

Se você criar um esquema com prmd, como descrito acima, você pode facilmente gerar documentação em Markdown para todos os endpoints com `prmd doc`.

Em adição aos detalhes do endpoint, disponibilize um resumo da APIcom informações sobre:

* Autenticação, includindo adquirir e usar tokens de autenticação.
* Estabilidade e versionamento da API, incluindo como selecionar a versão desejada da API.
* Cabeçalhos padrões de requisições e respostas.
* Formato de resposta de erro.
* Exemplos de utilização da APi com clientes em diferentes linguagens.

#### Prover exemplos executaveis

Disponibilize exemplos executaveis que os usuarios possam utilizar direto em seus terminais para ver como a API trabalha. Sempre que possivel, estes exemplos devem ser palavra por palavras, para minimizar o trabalho que um usuario precisa fazer para testar a API, p.e.:

```bash
$ export TOKEN=... # acquire from dashboard
$ curl -is https://$TOKEN@service.com/users
```

Se você usa [prmd](https://github.com/interagent/prmd) para gerar documentação em Markdown, você terá exemplos para cada endpoint inclusos.

#### Descreva a estabilidade

Descreva a estabilidade da sua API ou seus vários endpoints de acordo com a maturidade e estabilidade, p.e. com sinalização de prototipo/desenvolvimento/produção.

Veja a [politica de compatibilidade da API do Heroku](https://devcenter.heroku.com/articles/api-compatibility-policy)
para um exemplo de estabilidade e gerenciamento de mudanças.

Uma vez que sua API está declarada como estavel e pronta para produção, não faça mudanças incompativeis com a mesma versão de API. Se você precisar mudanças incompativeis, criue uma nova API com numero de versão superior.


### Traduções
 * [Versão em Português](https://github.com/Gutem/http-api-design/) (a partir de [fba98f08b5](https://github.com/interagent/http-api-design/commit/fba98f08b50acbb08b7b30c012a6d0ca795e29ee)), por [@Gutem](https://github.com/Gutem/)
 * [Versão em Espanhol](https://github.com/jmnavarro/http-api-design) (a partir de [2a74f45](https://github.com/interagent/http-api-design/commit/2a74f45b9afaf6c951352f36c3a4e1b0418ed10b)), por [@jmnavarro](https://github.com/jmnavarro/)
 * [Versão em Coreano](https://github.com/yoondo/http-api-design) (a partir de [f38dba6](https://github.com/interagent/http-api-design/commit/f38dba6fd8e2b229ab3f09cd84a8828987188863)), por [@yoondo](https://github.com/yoondo/)
 * [Versão em Chines Simplificado](https://github.com/ZhangBohan/http-api-design-ZH_CN) (a partir de [337c4a0](https://github.com/interagent/http-api-design/commit/337c4a05ad08f25c5e232a72638f063925f3228a)), por [@ZhangBohan](https://github.com/ZhangBohan/)
 * [Versão em Chinês Tradicional](https://github.com/kcyeu/http-api-design) (a partir de [232f8dc](https://github.com/interagent/http-api-design/commit/232f8dc6a941d0b25136bf64998242dae5575f66)), por [@kcyeu](https://github.com/kcyeu/)
 * [Versão em Turco](https://github.com/hkulekci/http-api-design/tree/master/tr) (a partir de [c03842f](https://github.com/interagent/http-api-design/commit/c03842fda80261e82860f6dc7e5ccb2b5d394d51)), por [@hkulekci](https://github.com/hkulekci/)
 * [Versão original em Inglês](https://github.com/interagent/http-api-design), por [@interagent](https://github.com/interagent)