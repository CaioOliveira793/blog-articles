# Idempotency

## Intro

Ao utilizar microsserviços, temos como objetivo criar serviços que possuem um escopo menor para colhermos os benefícios de aplicar este estilo de arquitetura. Dada a alta dependência de comunicação entre esses sistemas, é possível que chamadas a esses serviços falhem, resultando em aplicações em estado inconsistente.

### Network errors, timeout and retry

Quando lidamos com erros na rede, é comum e mais simples fazer o reenvio da requisição até que tenham sucesso. Entretanto, nem sempre as retentativas são descomplicadas, como nos erros de timeout, onde o serviço que não obteve a resposta não consegue determinar se a requisição já foi ou não realizada. Para superar esta limitação, é preciso que chamadas idênticas para o mesmo serviço apliquem o resultado apenas uma vez.

## Idempotency

Idempotência é a propriedade que se dá a uma operação que pode ser executada múltiplas vezes sem que resultado se altere após a aplicação inicial. Levando ao contexto de serviços, esta é a propriedade que procuramos para possibilitar a execução de retentativas que não produzam efeitos colaterais.

### API idempotency

No ambito de APIs, de acordo com o [MDN](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent) idempotência é relacionada a mutação do estado do servidor utilizando chamadas idênticas, entretanto é comum encontrarmos guias que apontam que a resposta do servidor também deve ser igual a que foi préviamente utilizada.

Apesar de não estar estritamente de acordo com a referência, o fundo prático desta afirmação está no objetivo de solucionar o problema das retentativas, no qual um dos requisitos é de que o cliente possa verificar se sua requisição foi ou não processada.

Casos em que a retentativa não apresenta uma resposta similar à execução anterior, podem tornar a verificação complexa por parte do cliente, impedindo por exemplo uma **política padrão de retentativas**.

#### Example of verification obstacles

No exemplo a seguir, o cliente deve tratar o erro `404` de endpoints com o método `DELETE` como um recurso não existente ou já deletado.

```
DELETE /posts/favorites/57e1a571af63 HTTP/1.1 -> 204 No Content
DELETE /posts/favorites/57e1a571af63 HTTP/1.1 -> 404 Not Found
DELETE /posts/favorites/57e1a571af63 HTTP/1.1 -> 404 Not Found
```

Em casos não triviais, como validações de regras de negócio, a lógica para determinar se o endpoint chamado já foi processado é consideravelmente mais complexa.

```
POST /posts/divulgations HTTP/1.1
{
	"post_id": "83bd0dd29b66",
	"emails": ["user@email.com"]
}

HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json
Content-Language: en
{
	"message": "Could not send same post to email address already send",
	"code": "divulgation_address_already_used"
}
```

## Achieving idempotency

Dado que para que uma API suporte retentativas de maneira prática seja necessário idempotência e a capacidade do cliente de reconhecer o estado atual sem muitas verificações, serão discutidas duas formas de atingir este objetivo, ambas com diferentes prós e contras.

### Idempotency-Key

O método através da chave de idempotência usa de um id único para toda requisição que represente uma nova ação (mesmo que sejam idênticas), onde caso seja necessário reenviar a mesma requisição representando a ação que pode ter sido tomada anteriormente, a mesma chave de idempotência deve ser utilizada, possibilitando a identificação pelo servidor se a requisição foi executada ou não.

```txt
POST /posts/divulgations HTTP/1.1
Idempotency-Key: 058494e50458
{
	"post_id": "83bd0dd29b66",
	"emails": ["user@email.com"]
}

HTTP/1.1 200 OK
// Client does not receive the response due timeout error

POST /posts/divulgations HTTP/1.1
Idempotency-Key: 058494e50458
{
	"post_id": "83bd0dd29b66",
	"emails": ["user@email.com"]
}

HTTP/1.1 200 OK
// Same response is given for the same idempotency key

POST /posts/divulgations HTTP/1.1
Idempotency-Key: 1b634c78964f
{
	"post_id": "ca0ebfa3b0fe",
	"emails": ["user@email.com"]
}

HTTP/1.1 200 OK
// New idempotency key, request is normally handled
```

Apesar da simplicidade, a sua implementação apresenta mais sobrecarga por necessitar de operações ACID entre as mutações de estado e a resposta produzida pela chave utilizada. Este nível de consistência é necessário para evitar situações onde a chave de idempotência é salva mas a persistência das ocorrências falham, ou o cenário contrário.

### Idempotency by design

Ao realizar o design de APIs, é possível notar que alguns endpoints são idempotentes por padrão, sem fazer uso de tokens por requisição.

```
PUT /posts/comments/61a4c1a3f06f HTTP/1.1
{
  "message": "Hasta la vista, baby",
  "post_id": "ca0ebfa3b0fe"
}
```

> Caso múltiplas requisições sejam feitas, o mesmo resultado será aplicado.

Utilizando o verbo `PUT`, é passado o id do recurso que será alterado, garantindo que, se a mesma requisição for feita múltiplas vezes, o servidor irá produzir o mesmo estado.

Dentro desta mesma lógica, em casos que é feita a criação do recurso, comumente utilizando o verbo `POST`, podemos construir um endpoint idempotente passando o id do recurso na requisição.

```
PUT /posts/comments/61a4c1a3f06f HTTP/1.1
If-None-Match: *
{
  "message": "I'll be back",
  "post_id": "ca0ebfa3b0fe"
}
```

> If-None-Match é um cabeçalho presente no protocolo HTTP que condiciona a requisição a ser executada se nenhum recurso for encontrado com o id enviado.

Ao utilizar o id do recurso no momento da criação é eliminada a possibilidade de alteração do estado com mais de uma chamada consecutiva.

Porém, aplicar idempotência utilizando o id do recurso na criação possui alguns desafios, como a produção de respostas semanticamente equivalentes para o cliente em caso de retentativas.

Embora possuindo maior complexidade, modelar recursos e endpoints idempotentes pode resultar em um tratamento mais escalável e que possua um encaixe melhor para o modelo de domínio.

---

Mesmo que a API construída não seja utilizada para comunicar com outros serviços, possuir a garantia de não produzir efeitos colaterais em retentativas facilita o tratamento de erros de comunicação, sendo aplicável mesmo que o único consumidor seja uma aplicação frontend.

## References

- https://aws.amazon.com/pt/builders-library/making-retries-safe-with-idempotent-APIs/
- https://developer.mozilla.org/pt-BR/docs/Glossary/Idempotent
- https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- https://www.w3.org/1999/04/Editing/#3.1
