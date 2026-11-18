# Worker de pagamentos

## Gateway Masterfy

Novas cobranças PIX são criadas pelo Worker em `POST https://api.masterfypagamentos.com/v1/payment`. A chave privada fica somente nos secrets do Worker Cloudflare. Configure `MASTERFY_API_KEY` e `MASTERFY_WEBHOOK_SECRET` no painel; não coloque esses valores no repositório, no frontend ou em comandos registrados.

No painel Masterfy, registre o webhook de pagamentos apontando para `https://marmita.laura66hg.workers.dev/api/masterfy/webhook` (ou para o domínio público atual do Worker) e habilite os eventos `payment.created`, `payment.paid`, `payment.refused` e `payment.refunded`. Configure o mesmo segredo de assinatura no Worker como `MASTERFY_WEBHOOK_SECRET`. O Worker verifica `X-Signature` como HMAC-SHA256 em Base64 sobre o corpo JSON original. Notificações enviadas por `notificationUrl` não carregam essa assinatura, por isso a integração usa o webhook registrado no painel e não envia `notificationUrl` na criação.

O adaptador mantém o contrato do site (`transactionId`, `status`, `amount` e `paymentData.copyPaste`), valida os valores no servidor e envia itens, dados do pagador, endereço de entrega e IP recebido pelo Worker. O QR é criado no site a partir do código PIX. Estados do pagamento são atualizados pelo webhook e salvos no Durable Object; `POST /api/pix/status` e sessões/taxas apenas leem o estado local e não consultam a API de pagamento. A Masterfy exige webhook para acompanhar mudanças; não implemente polling em `GET /v1/payment/:id`.

O identificador UUID do pedido é enviado como `externalRef`; a criação continua idempotente. Se houver erro de rede ou resposta inconclusiva, a cobrança não é criada novamente automaticamente. O estado é armazenado antes do rastreamento da Meta. Eventos repetidos são tratados de forma idempotente.

`POST /api/pix` recebe `{requestId, items:[{name,quantity,observation?}], customer:{name,email,phone,cpf,notes?,cpfOnReceipt?}, address:{street,number,complement,neighborhood,city,state,cep}, tracking?}`. Os preços e o total são calculados pelo catálogo do servidor, não pelo navegador. Pagamentos de pedidos e taxas adicionais usam PIX. As rotas de TMT e taxa de entrega mantêm seus valores e validações próprios.

### Publicação

- Preserve os bindings já declarados em `wrangler.toml`: `PIX_REQUESTS`, `PAYMENT_SESSIONS`, os rate limiters e o Secret Store da Meta.
- Configure os secrets `MASTERFY_API_KEY`, `MASTERFY_WEBHOOK_SECRET`, `META_ACCESS_TOKEN`, `TMT_CUSTOMER_KEY` e `SUPABASE_SECRET_KEY` diretamente no Cloudflare, conforme os recursos que já estão ativos.
- O destino do site é definido em `js/pix-config.js`; ele deve apontar para o Worker publicado.
- Registre o endpoint do webhook e confira a assinatura no painel Masterfy antes de aceitar pagamentos reais. Nenhum segredo, cadastro de webhook ou deploy remoto pode ser configurado por este repositório.
- Registros antigos da Blackcat/VenoPag continuam identificados no Durable Object para recuperação de pagamentos iniciados antes da troca. As novas cobranças usam exclusivamente Masterfy.

## TMT e taxa de entrega

O token de sessão continua sendo emitido pelo Worker e validado pelo Durable Object. A TMT custa R$ 9,90; a taxa de entrega custa R$ 7,90 e só pode ser gerada após a confirmação das etapas anteriores. Cada taxa tem identificador idempotente próprio e não altera o valor nem os itens da sacola original.

## Rastreamento

O Worker mantém o Pixel 1179633571678200 e a Conversions API. O evento Purchase do servidor usa o valor e o identificador do pedido confirmado, com dados de correspondência guardados no Durable Object. Dados de navegação e cliente continuam sujeitos às regras já implementadas em `worker.mjs`.

## Cadastro do pet

Antes de usar a gravação do cadastro, execute `supabase-pet-registrations.sql` no SQL Editor do Supabase e configure `SUPABASE_SECRET_KEY` como secret do Worker. A página envia os dados ao Worker, que valida e grava no Supabase. Nenhuma chave Supabase deve ser incluída no site.

## Catálogo

`generate-catalog.cjs` atualiza os preços do bloco gerado em `worker.mjs` a partir de `js/produto-data.js`. Rode `node worker-pix/generate-catalog.cjs` quando alterar preços ou produtos.
