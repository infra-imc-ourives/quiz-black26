# Entrega técnica — Quiz Primeiro Você

Quiz de 12 perguntas que mostra à pessoa "em que lugar da própria lista" ela está e leva para o evento do dia 28.10, às 20h, e para o grupo do WhatsApp.

## O que está neste pacote

- `dist/index.html`: página completa do quiz, com estilos, perguntas, cálculo do resultado e envio das respostas.
- `dist/assets/logo-primeiro-voce.webp`: logo da abertura.
- `dist/assets/selo.webp`: selo usado nas telas de transição, carregamento e resultado.
- `dist/teste-webhook.html`: página de apoio para testar o webhook antes da publicação.
- `exemplo-payload.json`: exemplo dos dados enviados ao concluir o quiz.

É uma página estática em HTML, CSS e JavaScript puro. Não exige Node.js, npm, banco de dados ou processo de compilação. A única dependência externa é a fonte Bodoni Moda, carregada do Google Fonts.

## Antes de publicar: duas linhas para preencher

No começo do `<script>` de `dist/index.html`:

```js
var WEBHOOK_URL="";
var WHATSAPP_URL="";
```

- `WEBHOOK_URL`: endpoint HTTPS que recebe as respostas (n8n, CRM ou outra automação). **Enquanto estiver vazio, o quiz funciona normalmente para a pessoa, mas nenhuma resposta é registrada.**
- `WHATSAPP_URL`: link de convite do grupo. Enquanto estiver vazio, o botão "ENTRAR NO GRUPO DO WHATSAPP" não leva a lugar nenhum. Preenchido, abre o link em nova aba.

## Formas de instalação

### Opção 1: publicar no domínio oficial

1. Copiar o conteúdo da pasta `dist` para uma pasta pública do servidor ou CDN.
2. Manter a pasta `assets` ao lado do `index.html`, pois as imagens usam caminhos relativos.
3. Definir a rota desejada, por exemplo `/primeiro-voce/`.
4. Servir exclusivamente por HTTPS.

### Opção 2: incorporar em uma página existente

```html
<iframe
  src="https://dominio-final/primeiro-voce/"
  title="Primeiro Você"
  loading="lazy"
  style="width:100%;min-height:100vh;border:0;display:block"
></iframe>
```

## Parâmetros de URL

- `?nome=Maria` já preenche o campo de nome na abertura.
- Todos os parâmetros da URL (UTMs, `email`, `phone`, `id` de contato etc.) seguem no campo `params` do payload.

**Ponto de atenção:** o quiz pede apenas o primeiro nome. Não há campo de e-mail nem de telefone. Para o registro virar contato no CRM, o link divulgado precisa carregar um identificador, por exemplo `?email=%EMAIL%` no disparo de e-mail ou o ID do contato no ActiveCampaign. Sem isso, o que chega é uma resposta anônima com UTMs.

## Envio das respostas

Ao chegar no resultado, a página faz um `POST` com `Content-Type: application/json` para o `WEBHOOK_URL`, no formato de `exemplo-payload.json`. O endpoint deve responder com status `2xx`.

O envio usa o mesmo mecanismo do Diagnóstico dos 13 Degraus:

1. Guarda a submissão numa fila no `localStorage` antes de tentar enviar.
2. Tenta o `POST` até três vezes, com intervalos de 1,2 e 2,4 segundos.
3. Só remove da fila quando o endpoint responde 2xx.
4. Mostra uma linha discreta no fim do resultado ("Respostas registradas." ou aviso de falha com botão "Tentar novamente").
5. Reenvia a fila quando a conexão volta, quando a pessoa retorna à aba e quando reabre a página.

A mesma submissão pode chegar mais de uma vez. Por isso o payload traz `submissionId`, gerado no navegador e igual em todas as tentativas. O workflow deve usá-lo como chave de idempotência: se o `submissionId` já existe, atualizar em vez de criar.

### Se o destino for o n8n

1. O workflow precisa estar **ativo**. A URL `/webhook-test/` só funciona com a tela de execução aberta e não serve para produção.
2. O nó Webhook precisa liberar **CORS** para o domínio do quiz (Options → Allowed Origins). Como o envio é JSON, o navegador dispara um preflight `OPTIONS` antes do `POST`. Sem CORS, a pessoa vê o resultado normalmente e a resposta não chega, sem erro visível.

## Como testar o webhook

Publique `dist/teste-webhook.html` junto do quiz e abra **no mesmo domínio em que o quiz ficará no ar**. Cole o endpoint no campo e clique em enviar. A página manda um payload igual ao real, com `modo: "teste"` para ser filtrado no workflow, e diz onde está o problema:

| O que a página mostra | O que está acontecendo | O que fazer |
| --- | --- | --- |
| Entrega confirmada | O endpoint recebeu e respondeu 2xx | Conferir se o registro chegou completo |
| Servidor acessível, mas respondeu 404 | Workflow inativo ou caminho divergente | Ativar o workflow e conferir o caminho |
| O webhook respondeu HTTP 5xx | A requisição chegou, algum nó falhou | Abrir as execuções e ver o nó com erro |
| O CORS está bloqueando o envio | Origem não autorizada | Liberar o domínio no nó Webhook |
| Não foi possível alcançar o servidor | Endereço errado ou instância fora do ar | Conferir URL e disponibilidade |

Depois que o quiz entrar no ar, `teste-webhook.html` pode ser removida.

## Dados enviados

- `quiz`: `primeiro_voce_2026`.
- `version`: versão do payload (`1.0`).
- `submissionId`: identificador único da submissão.
- `nome`: primeiro nome digitado (pode vir vazio).
- `perfil`: resultado exibido. Um de `Fim da fila` (índice ≥ 75), `Sempre para depois` (≥ 50), `Na porta da sua vez` (≥ 25) ou `Quase em primeiro` (< 25).
- `indice_geral`: média de 0 a 100 das quatro áreas. Quanto maior, mais a pessoa se coloca para depois.
- `indices`: percentual de espera por área (`dinheiro`, `tempo`, `sonhos`, `permissao`).
- `area_maior_espera`: área com o maior índice.
- `respostas`: chave curta de cada resposta, boa para tags e campos no CRM.
- `respostas_texto`: o texto exato da alternativa escolhida.
- `completedAt`: data e hora da conclusão em ISO.
- `params`: parâmetros da URL.

Perguntas que não pontuam, mas valem para segmentação: `area` (área que quer mudar), `sonho` (sonho adiado), `relacao` (história com a Elainne: `nova`, `redes`, `aluno_evento`, `holo`) e `momento` (intenção para o dia 28.10: `agora`, `receio`, `curioso`, `nao_agora`).

Sugestão de campos ou tags no CRM:

- `primeiro_voce_concluido`
- `primeiro_voce_perfil`
- `primeiro_voce_indice`
- `primeiro_voce_area_maior_espera`
- `primeiro_voce_momento`
- `primeiro_voce_relacao`
- `primeiro_voce_sonho`
- UTMs de origem

## Privacidade e LGPD

A página não pede e-mail nem telefone, mas as respostas são pessoais e, se o link carregar identificação por URL, passam a estar ligadas a um contato. Antes da publicação, a equipe responsável deve confirmar finalidade e base legal, definir retenção e acesso, e avaliar se é preciso exibir aviso de privacidade na abertura.

## Checklist de homologação

- Preencher `WEBHOOK_URL` e `WHATSAPP_URL`.
- Abrir o quiz no computador e no celular.
- Responder até o fim, usando também o botão de voltar.
- Conferir as duas telas de transição (depois da pergunta 1 e da pergunta 6).
- Testar "Eu" e outra pessoa na primeira pergunta: a leitura final muda.
- Confirmar que o registro chegou ao destino com todas as respostas e os `params`.
- Simular falha de rede ao concluir e confirmar que o reenvio recupera o registro.
- Confirmar que o botão do WhatsApp abre o grupo certo.
- Testar `?nome=` e as UTMs no link.
