# CLAUDE.md — Plataforma de Promoções em Tempo Real

## Visão Geral do Projeto

Plataforma web estilo Pelando.com que agrega promoções e cupons de desconto de múltiplos e-commerces brasileiros em tempo real. As promoções chegam de duas fontes: bot automatizado (workers de scraping + APIs de afiliados) e publicação manual por usuários cadastrados. Usuários podem se cadastrar, comentar, votar, salvar ofertas e publicar promoções.

**Stack:**
- Frontend: Angular (versão mais recente)
- Backend: Go / Golang (versão mais recente)
- Banco de dados: PostgreSQL
- Cache: Redis
- Mensageria: RabbitMQ (ou NATS)
- Containerização: Docker + Docker Compose

---

## Índice

1. [Regras de Negócio](#regras-de-negócio)
2. [Arquitetura de Software](#arquitetura-de-software)
3. [Fontes de Dados](#fontes-de-dados)
4. [Módulos do Sistema](#módulos-do-sistema)
5. [Modelo de Dados](#modelo-de-dados)
6. [APIs do Backend](#apis-do-backend)
7. [Frontend Angular](#frontend-angular)
8. [Infraestrutura](#infraestrutura)

---

## Regras de Negócio

### RN-001 — Cadastro de Usuário

O cadastro é dividido em três etapas:

**Etapa 1 — Formulário inicial (pré-confirmação)**
- O usuário informa apenas três campos obrigatórios: e-mail, CPF e senha.
- O e-mail deve ser único no sistema.
- O CPF deve ser único no sistema, validado no formato brasileiro (11 dígitos com dígitos verificadores válidos) e armazenado criptografado (AES-256) no banco de dados.
- A senha deve ter no mínimo 8 caracteres, com ao menos uma letra maiúscula e um número.
- O usuário deve aceitar explicitamente a Política de Privacidade e os Termos de Uso nesta etapa.
- Ao concluir, o sistema envia um e-mail de confirmação. A conta fica com status `pending_confirmation`.

**Etapa 2 — Confirmação de e-mail**
- O usuário clica no link recebido por e-mail.
- A conta passa para o status `pending_profile`.
- Usuários em `pending_confirmation` não conseguem acessar a plataforma.

**Etapa 3 — Completar perfil (pós-confirmação, obrigatório antes do primeiro acesso)**
- Após confirmar o e-mail, o usuário é redirecionado obrigatoriamente para a tela de completar perfil antes de acessar qualquer funcionalidade.
- Os seguintes campos são solicitados nesta etapa:
  - Nome completo (obrigatório)
  - Data de nascimento completa (obrigatório)
  - Gênero (obrigatório): `masculino`, `feminino`, `nao_binario`, `prefiro_nao_informar`
  - Telefone/WhatsApp (obrigatório)
  - Interesses/categorias favoritas (obrigatório): mínimo 1, máximo 10
  - Nome de exibição (obrigatório)
- A data de nascimento é usada para calcular a idade dinamicamente. O sistema bloqueia o cadastro de menores de 18 anos. A idade nunca é armazenada como campo fixo.
- O telefone é validado e armazenado no formato E.164 (ex: +5534999999999). O campo aceita entrada com ou sem máscara.
- Ao concluir, a conta passa para o status `active` e o usuário acessa a plataforma normalmente.

**Status possíveis da conta:**
- `pending_confirmation` → e-mail ainda não confirmado
- `pending_profile` → e-mail confirmado, perfil ainda não completado
- `active` → conta totalmente ativa
- `banned` → conta banida por moderação

### RN-002 — Autenticação

- Autenticação via JWT (Access Token + Refresh Token).
- Access Token expira em 15 minutos.
- Refresh Token expira em 7 dias.
- O sistema deve oferecer login via e-mail/senha e via OAuth2 (Google).
- Após 5 tentativas de login com senha errada, a conta é bloqueada por 15 minutos.

### RN-003 — Perfil de Usuário

- Cada usuário possui os seguintes dados de perfil:
  - Avatar (URL de imagem)
  - Nome de exibição (público)
  - Nome completo (privado, coletado no cadastro)
  - Biografia curta (público, opcional)
  - CPF (privado, criptografado, imutável após cadastro)
  - Data de nascimento (privada, imutável após cadastro)
  - Gênero (privado, editável)
  - Telefone/WhatsApp (privado, editável)
  - Interesses/categorias favoritas (privado, editável)
  - Reputação (público)
  - Data de cadastro (público)
- A reputação é calculada com base em votos recebidos nas promoções e comentários postados.
- Campos públicos (visíveis por outros usuários): avatar, nome de exibição, biografia, reputação, data de cadastro.
- Campos privados (visíveis apenas pelo próprio usuário e pelo sistema de dados): nome completo, CPF, data de nascimento, gênero, telefone, interesses.
- O usuário pode editar avatar, nome de exibição, biografia, gênero, telefone e interesses a qualquer momento.
- CPF e data de nascimento são imutáveis após o cadastro. Alterações nesses campos exigem contato com suporte.
- Usuários não podem alterar seu e-mail sem reconfirmação via link enviado ao novo endereço.

### RN-004 — Promoções

- Promoções podem ser publicadas por duas origens: bot automatizado da plataforma (workers de scraping e APIs de afiliados) ou por usuários cadastrados com status `active`.
- O administrador pode criar, editar e remover qualquer promoção via painel admin.
- Uma promoção contém: título, descrição, URL do produto, URL da imagem, preço original, preço promocional, percentual de desconto, loja de origem, categoria, data de expiração (opcional), cupom (opcional) e fonte (`api_mercadolivre`, `api_shopee`, `api_amazon`, `scraping_kabum`, `scraping_pichau`, `scraping_terabyte`, `user`, `admin_manual`).
- O percentual de desconto é sempre calculado automaticamente pelo sistema: `((preco_original - preco_promocional) / preco_original) * 100`.
- Promoções com desconto abaixo de 5% são rejeitadas automaticamente, seja pelo bot ou pela validação do endpoint de criação.
- Promoções publicadas por usuários entram com status `pending_review` e só ficam visíveis na listagem após aprovação automática (score mínimo de votos) ou manual pelo admin/moderador.
- Promoções expiradas são automaticamente arquivadas e não aparecem na listagem principal.
- Promoções podem ser marcadas como "Destaque" apenas pelo administrador.
- Uma promoção pode ter status: `pending_review`, `active`, `expired`, `archived`, `removed`.

### RN-005 — Votação (🚀 Foguete / 🗑️ Lixo)

- Usuários autenticados e confirmados podem votar "🚀 Foguete" ou "🗑️ Lixo" em uma promoção.
- Um usuário pode votar apenas uma vez por promoção.
- O usuário pode trocar seu voto a qualquer momento.
- O score de uma promoção é calculado por: `votos_rocket - votos_trash`.
- Promoções com score abaixo de -10 são automaticamente arquivadas.
- O criador da promoção não pode votar na própria promoção.

### RN-006 — Comentários

- Usuários autenticados e confirmados podem comentar em promoções.
- Comentários suportam texto simples (sem HTML, sem markdown).
- Comentários podem ser deletados pelo próprio autor ou por moderadores.
- Comentários com 3 ou mais denúncias entram em fila de revisão moderação.
- Respostas a comentários são suportadas (1 nível de profundidade apenas).

### RN-007 — Cupons de Desconto

- Um cupom possui: código, loja, percentual ou valor fixo de desconto, data de validade e categorias de produto aplicáveis.
- O sistema verifica periodicamente se um cupom ainda está ativo (via scraping ou API).
- Cupons expirados são automaticamente desativados.
- Usuários podem reportar um cupom como "não funcionou".
- Após 5 reports de "não funcionou", o cupom é automaticamente desativado e enviado para revisão.

### RN-008 — Categorias

- Categorias são gerenciadas apenas por administradores.
- Uma promoção deve pertencer a pelo menos uma categoria.
- Categorias possuem ícone, cor e slug para URL.
- Exemplos: Eletrônicos, Moda, Casa, Games, Beleza, Mercado, Viagens.

### RN-009 — Favoritos

- Usuários autenticados podem salvar promoções em favoritos.
- A lista de favoritos é privada por padrão, mas pode ser tornada pública pelo usuário.
- O sistema envia notificação (e-mail ou push) quando uma promoção favoritada está prestes a expirar (2 horas antes).

### RN-010 — Notificações

- Usuários podem configurar alertas por categoria. Ex: "Me notifique quando aparecer promoção de Games".
- Notificações são enviadas via e-mail e/ou push notification (web push).
- O usuário pode desativar notificações a qualquer momento.
- Máximo de 5 alertas por categoria por usuário.

### RN-011 — Moderação

- Administradores e moderadores podem: remover promoções, banir usuários, editar cupons e gerenciar categorias.
- Usuários podem denunciar promoções e comentários.
- Promoções denunciadas 5 vezes entram em fila de revisão.
- Moderadores recebem notificação interna ao entrar novo item na fila.

### RN-015 — Privacidade e LGPD

- O sistema coleta dados pessoais sensíveis (CPF, data de nascimento, gênero, telefone). O tratamento desses dados deve estar em conformidade com a **Lei Geral de Proteção de Dados (LGPD — Lei 13.709/2018)**.
- O usuário deve aceitar explicitamente a **Política de Privacidade** e os **Termos de Uso** durante o cadastro. Esse aceite é registrado com timestamp e versão do documento.
- O CPF é coletado com a finalidade declarada de: identificação única do usuário e enriquecimento do pipeline de dados analíticos. Essa finalidade deve constar na Política de Privacidade.
- O usuário tem direito a solicitar a exportação de todos os seus dados pessoais (portabilidade).
- O usuário tem direito a solicitar a exclusão da conta e anonimização de seus dados pessoais. Após a exclusão: CPF e dados sensíveis são deletados, demais dados são anonimizados (não deletados, para preservar integridade de comentários e histórico de votos).
- Os dados pessoais (CPF, telefone, data de nascimento, gênero) **nunca** são expostos em nenhum endpoint público da API.
- Logs de acesso a dados sensíveis devem ser mantidos por no mínimo 6 meses.
- O pipeline de engenharia de dados downstream deve acessar os dados através de views anonimizadas ou com CPF mascarado (ex: `***.***.***-00`), salvo quando a identificação for estritamente necessária e autorizada.

### RN-012 — Coleta Automática de Promoções (Scraping e APIs)

- O sistema coleta promoções automaticamente das seguintes fontes (detalhes na seção [Fontes de Dados](#fontes-de-dados)).
- Promoções coletadas automaticamente passam por validação antes de serem publicadas.
- Promoções duplicadas (mesmo produto, mesma loja, mesmo preço) são ignoradas.
- A frequência de coleta é configurável por fonte.
- Erros de coleta são registrados e alertados para administradores.

### RN-013 — Deduplicação

- O sistema identifica duplicatas comparando: URL do produto normalizada + loja + preço promocional.
- A URL é normalizada removendo parâmetros de rastreamento (UTM, etc.) antes da comparação.
- Caso a mesma promoção chegue de duas fontes diferentes, a mais completa é mantida e a outra é descartada silenciosamente.

### RN-014 — Expiração Automática

- Promoções com data de expiração definida são arquivadas automaticamente pelo sistema via job agendado (cron).
- Promoções sem data de expiração são verificadas periodicamente via scraping para confirmar se o preço ainda está ativo.
- Se o preço voltou ao normal, a promoção é arquivada automaticamente.

---

### RN-021 — Sistema de Notificações

O sistema oferece quatro canais de notificação independentes e combináveis. O usuário escolhe quais canais ativar e para quais contextos cada canal será usado.

---

#### RN-021.1 — Canais Disponíveis

| Canal | Identificador | Configuração necessária |
|---|---|---|
| Navegador (Web Push) | `browser` | Permissão de push no navegador |
| E-mail | `email` | E-mail confirmado no cadastro |
| WhatsApp | `whatsapp` | Número de telefone validado |
| Telegram | `telegram` | Conexão com bot via código de verificação |

- O usuário pode ativar um ou mais canais simultaneamente.
- Cada canal pode ser ativado ou desativado de forma independente.
- Para WhatsApp e Telegram, o usuário escolhe um ou ambos — não há obrigatoriedade de escolher apenas um.
- Canais não configurados não aparecem como opção ativa nas preferências.

---

#### RN-021.2 — Notificações por Navegador (Web Push)

- Utiliza a **Web Push API** padrão (protocolo VAPID).
- O usuário deve conceder permissão explícita no navegador ao ativar o canal.
- Se a permissão for negada, o sistema orienta o usuário a habilitá-la manualmente nas configurações do navegador.
- O subscription token do navegador é armazenado na tabela `push_subscriptions` e vinculado ao usuário.
- Um usuário pode ter múltiplos tokens (diferentes dispositivos/navegadores), todos recebem a notificação.
- Notificações push exibem: título da promoção, desconto, loja, imagem do produto e link direto.

---

#### RN-021.3 — Notificações por E-mail

- Utiliza o e-mail confirmado no cadastro.
- E-mails de notificação são disparados via serviço SMTP (ex: SendGrid, Resend ou Amazon SES).
- Cada e-mail de notificação contém no máximo 5 promoções por envio para não sobrecarregar o usuário.
- O usuário pode definir uma frequência de agrupamento: `imediato`, `resumo_diario` (1x/dia às 9h) ou `resumo_semanal` (toda segunda às 9h).
- E-mails de notificação possuem sempre um link de descadastro (unsubscribe) em conformidade com a LGPD.

---

#### RN-021.4 — Notificações por WhatsApp

- Utiliza a **API oficial do WhatsApp Business (Meta Cloud API)** com templates aprovados.
- O número de telefone cadastrado pelo usuário é usado para envio.
- O usuário deve confirmar o número respondendo a uma mensagem de verificação enviada pelo sistema antes de ativar o canal.
- Mensagens seguem templates pré-aprovados pela Meta contendo: nome do produto, desconto, loja e link.
- Limite de 1 mensagem por promoção por usuário (sem agrupamento, pois o WhatsApp já tem interface de chat).
- O usuário pode pausar o canal enviando "PAUSAR" para o número do bot, e reativar enviando "ATIVAR".

---

#### RN-021.5 — Notificações por Telegram

- Utiliza a **API de Bots do Telegram**.
- O usuário conecta o canal acessando o bot da plataforma no Telegram e enviando um código de verificação gerado pelo sistema.
- O `chat_id` retornado pelo Telegram é armazenado na tabela `telegram_subscriptions` e vinculado ao usuário.
- Mensagens do Telegram suportam formatação Markdown, imagem inline e botão de ação ("Ver oferta").
- O usuário pode pausar enviando `/pausar` para o bot e reativar com `/ativar`.
- Limite de 1 mensagem por promoção por usuário.

---

#### RN-021.5b — Notificações de Itens Favoritados

Usuários que salvaram promoções nos favoritos recebem notificações automáticas sobre o estado dessas promoções, independentemente das preferências de alerta por categoria/loja.

**Eventos que geram notificação para o usuário que favoritou:**

| Evento | Gatilho | Quando |
|---|---|---|
| `favorite_expiring_soon` | Promoção favoritada prestes a expirar | 2 horas antes do `expires_at` |
| `favorite_flash_expiring_soon` | Promoção relâmpago favoritada prestes a encerrar | 30 minutos antes do `flash_ends_at` |
| `favorite_flash_low_stock` | Estoque da promoção relâmpago favoritada chegando ao fim | Quando `flash_stock_remaining` ≤ 10% do `flash_stock` original |
| `favorite_expired` | Promoção favoritada expirou | No momento do encerramento (`expires_at` atingido) |
| `favorite_flash_ended` | Promoção relâmpago favoritada encerrou | No momento do encerramento por prazo ou estoque |
| `favorite_removed` | Promoção favoritada foi removida da plataforma | No momento da remoção por moderação ou invalidação |
| `favorite_price_dropped` | Preço de uma promoção favoritada baixou ainda mais | Quando `promo_price` é atualizado para um valor menor que o anterior |

**Regras específicas:**

- As notificações de favoritos são configuráveis pelo usuário: pode ativar/desativar cada evento individualmente e escolher o canal (browser, e-mail, WhatsApp, Telegram).
- Notificações de `favorite_flash_expiring_soon` e `favorite_flash_low_stock` têm prioridade `high` na fila — são disparadas antes das demais.
- O evento `favorite_flash_low_stock` é disparado apenas uma vez por promoção por usuário, mesmo que o estoque continue caindo.
- O evento `favorite_price_dropped` só é disparado se a nova redução for de pelo menos 5% em relação ao preço anterior registrado.
- Para o e-mail de `favorite_expiring_soon`, o template utilizado é o **TEMPLATE-06** já documentado.
- Para os demais eventos de favorito, os e-mails seguem o mesmo padrão visual do TEMPLATE-06, com conteúdo adaptado ao evento:
  - `favorite_flash_expiring_soon`: destaque de urgência com contador textual de minutos restantes
  - `favorite_flash_low_stock`: badge "⚠️ Estoque acabando" com quantidade restante
  - `favorite_expired`: tom informativo — "Esta promoção encerrou. Veja ofertas similares." + 3 sugestões de promoções da mesma categoria
  - `favorite_flash_ended`: igual ao anterior, com indicação se encerrou por prazo ou por estoque esgotado
  - `favorite_removed`: tom informativo — "Esta promoção foi removida da plataforma." + link para ver outras promoções
  - `favorite_price_dropped`: badge "📉 Ficou ainda mais barato!" com preço anterior e novo preço em destaque

**Sugestões de promoções similares:**
- Os e-mails de `favorite_expired`, `favorite_flash_ended` e `favorite_removed` incluem automaticamente 3 promoções ativas da mesma categoria da promoção encerrada, ordenadas por `trend_score`.

#### RN-021.6 — Preferências de Notificação por Categoria e Loja

- O usuário configura independentemente para cada canal quais categorias e lojas deseja receber notificações.
- Exemplo: receber push no navegador para "Eletrônicos" e "Games", mas WhatsApp apenas para "Relâmpago".
- Configurações disponíveis por canal:
  - **Categorias de interesse**: lista de categorias (já definidas no perfil, mas editáveis aqui separadamente)
  - **Lojas de interesse**: Amazon, Mercado Livre, Shopee, Kabum, etc.
  - **Apenas relâmpago**: receber notificação somente de promoções relâmpago
  - **Desconto mínimo**: só notificar se o desconto for acima de X% (configurável de 5% a 90%)
  - **Faixa de preço**: só notificar se o preço promocional estiver dentro de uma faixa (opcional)
  - **Notificações de favoritos**: ativar/desativar cada evento individualmente (`favorite_expiring_soon`, `favorite_flash_expiring_soon`, `favorite_flash_low_stock`, `favorite_expired`, `favorite_flash_ended`, `favorite_removed`, `favorite_price_dropped`)
- Uma notificação de alerta por categoria/loja só é disparada para um usuário se a promoção atender a **todos** os critérios configurados para aquele canal.
- Notificações de favoritos são disparadas independentemente dos critérios de categoria/loja — basta o item estar nos favoritos e o evento correspondente estar ativo.
- Máximo de 20 notificações por canal por dia para evitar spam. Notificações de favoritos com prioridade `high` (`favorite_flash_expiring_soon`, `favorite_flash_low_stock`) **não** contam para esse limite.

---

#### RN-021.7 — Newsletter "Promoções em Alta" (Digest por IA)

- Usuários podem se inscrever em uma newsletter periódica chamada **"Em Alta"**.
- A newsletter é gerada automaticamente pela **API da Anthropic (Claude)** com base nas 10 promoções com maior trend_score do período.
- **Frequências disponíveis:** diária (enviada às 9h) ou semanal (toda segunda às 9h). O usuário escolhe uma das duas.
- **Conteúdo gerado pela IA:**
  - Título criativo e contextualizado para o período (ex: "As melhores ofertas dessa segunda")
  - Parágrafo introdutório curto e envolvente
  - Para cada uma das 10 promoções: título, descrição curta destacando o benefício, preço, desconto e CTA ("Ver oferta")
  - Parágrafo de encerramento com chamada para acessar a plataforma
- **Prompt enviado ao Claude:** as 10 promoções são enviadas como dados estruturados (JSON) e o Claude retorna o e-mail formatado em HTML responsivo pronto para envio.
- O HTML gerado passa por sanitização antes do envio para garantir que não há scripts ou conteúdo malicioso.
- A newsletter é gerada uma única vez por período e o mesmo HTML é enviado para todos os inscritos daquela frequência.
- Usuários podem se inscrever e desinscrever da newsletter de forma independente dos outros canais de notificação.
- Todo e-mail de newsletter possui link de descadastro (unsubscribe) em conformidade com a LGPD.
- A inscrição na newsletter não requer nenhuma configuração de categoria — ela sempre reflete as top 10 promoções globais em alta.

---

### RN-022 — Templates de E-mail HTML

Todos os e-mails enviados pela plataforma utilizam templates HTML responsivos, padronizados e personalizados por tipo de ação. Nenhum e-mail é enviado em texto puro.

---

#### RN-022.1 — Estrutura Base (Layout Padrão)

Todos os templates compartilham a mesma estrutura base, garantindo consistência visual. Os valores de identidade visual (nome da plataforma, cor primária, logo, URL base) são injetados via variáveis no momento do render.

```
┌─────────────────────────────────────┐
│           HEADER                    │
│  [Logo da Plataforma]               │
│  [Nome da Plataforma]               │
├─────────────────────────────────────┤
│           HERO (opcional)           │
│  Ícone ou imagem contextual         │
│  Título principal do e-mail         │
├─────────────────────────────────────┤
│           BODY                      │
│  Saudação personalizada             │
│  Conteúdo específico do template    │
│  Botão de ação principal (CTA)      │
├─────────────────────────────────────┤
│           FOOTER                    │
│  Links: Plataforma | Perfil |       │
│         Preferências | Unsubscribe  │
│  Aviso legal LGPD                   │
│  © Ano - Nome da Plataforma         │
└─────────────────────────────────────┘
```

**Variáveis globais injetadas em todos os templates:**

| Variável | Descrição |
|---|---|
| `{{platform_name}}` | Nome da plataforma |
| `{{platform_url}}` | URL base da plataforma |
| `{{platform_logo_url}}` | URL do logo |
| `{{platform_primary_color}}` | Cor primária em hex |
| `{{user_display_name}}` | Nome de exibição do destinatário |
| `{{unsubscribe_url}}` | Link de descadastro com token único |
| `{{current_year}}` | Ano atual para o rodapé |

**Regras gerais de todos os templates:**
- HTML responsivo compatível com os principais clientes de e-mail (Gmail, Outlook, Apple Mail, Samsung Mail).
- CSS inline obrigatório (clientes de e-mail não suportam `<style>` externo de forma confiável).
- Largura máxima do container: 600px, centralizado.
- Fontes seguras para e-mail: `Arial, Helvetica, sans-serif` como fallback universal.
- Imagens sempre com atributo `alt` preenchido.
- Todos os links com `target="_blank"` e `rel="noopener"`.
- Versão texto puro (plain text) gerada automaticamente como fallback.
- Pré-cabeçalho (preview text) configurado individualmente por template — é o texto exibido na caixa de entrada antes de abrir o e-mail.

---

#### RN-022.2 — Catálogo de Templates

---

##### TEMPLATE-01 — Confirmação de Cadastro

**Gatilho:** imediatamente após o usuário completar o formulário de cadastro (Etapa 1).
**Assunto:** `Confirme seu e-mail para acessar a {{platform_name}}`
**Preview text:** `Falta só um passo! Clique para confirmar seu cadastro.`

**Conteúdo:**
- Saudação com o e-mail cadastrado (nome ainda não disponível nessa etapa)
- Mensagem explicando que o link expira em 24 horas
- Botão CTA primário: **"Confirmar meu e-mail"** → link de confirmação
- Aviso: se não foi você quem se cadastrou, ignorar o e-mail

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{confirmation_url}}` | Link de confirmação com token |
| `{{token_expiry_hours}}` | Horas até expirar (padrão: 24) |

---

##### TEMPLATE-02 — Boas-vindas

**Gatilho:** imediatamente após o usuário confirmar o e-mail e completar o perfil (status → `active`).
**Assunto:** `Bem-vindo(a) à {{platform_name}}, {{user_display_name}}! 🎉`
**Preview text:** `Sua conta está ativa. Veja o que está em alta agora.`

**Conteúdo:**
- Saudação personalizada com nome de exibição
- Breve apresentação da plataforma (2-3 linhas)
- 3 cards das promoções mais em alta no momento (geradas dinamicamente)
- Botão CTA primário: **"Ver todas as promoções"** → feed principal
- Seção secundária: "Configure suas notificações" → link para preferências

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{top_promotions}}` | Array com 3 promoções em alta (título, loja, desconto, imagem, URL) |
| `{{preferences_url}}` | Link para a página de preferências de notificação |

---

##### TEMPLATE-03 — Recuperação de Senha

**Gatilho:** usuário solicita redefinição de senha.
**Assunto:** `Redefinição de senha — {{platform_name}}`
**Preview text:** `Recebemos um pedido para redefinir sua senha.`

**Conteúdo:**
- Saudação com nome de exibição
- Informação de que o link expira em 1 hora
- Botão CTA primário: **"Redefinir minha senha"** → link de redefinição
- Aviso de segurança: se não foi você, ignorar o e-mail e considerar trocar a senha
- IP e horário da solicitação (para fins de segurança)

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{reset_url}}` | Link de redefinição com token |
| `{{token_expiry_minutes}}` | Minutos até expirar (padrão: 60) |
| `{{request_ip}}` | IP de onde veio a solicitação |
| `{{request_datetime}}` | Data e hora da solicitação |

---

##### TEMPLATE-04 — Alerta de Promoção por Categoria/Loja

**Gatilho:** nova promoção publicada que atende aos critérios de alerta do usuário (canal e-mail com frequência `instant`).
**Assunto:** `🔥 {{discount_percent}}% OFF em {{store}} — {{promotion_title}}`
**Preview text:** `{{original_price}} por apenas {{promo_price}}. Corre que pode acabar!`

**Conteúdo:**
- Saudação curta
- Card grande da promoção: imagem do produto, título, loja, preço original riscado, preço promocional em destaque, percentual de desconto em badge, cupom (se houver)
- Botão CTA primário: **"Ver oferta"** → URL da promoção na plataforma
- Informação de expiração (se houver)
- Rodapé secundário: "Este alerta foi enviado porque você segue [categoria/loja]. Gerenciar alertas."

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{promotion_title}}` | Título da promoção |
| `{{promotion_url}}` | URL da promoção na plataforma |
| `{{product_image_url}}` | Imagem do produto |
| `{{store}}` | Nome da loja |
| `{{original_price}}` | Preço original formatado |
| `{{promo_price}}` | Preço promocional formatado |
| `{{discount_percent}}` | Percentual de desconto |
| `{{coupon_code}}` | Cupom (opcional, omitido se vazio) |
| `{{expires_at_formatted}}` | Data de expiração formatada (opcional) |
| `{{alert_context}}` | Contexto do alerta: "Eletrônicos" ou "Kabum" |
| `{{manage_alerts_url}}` | Link para gerenciar alertas |

---

##### TEMPLATE-05 — Promoção Relâmpago Detectada

**Gatilho:** nova promoção relâmpago publicada que atende aos critérios do usuário.
**Assunto:** `⚡ RELÂMPAGO: {{discount_percent}}% OFF em {{store}} — Encerra em {{time_remaining}}`
**Preview text:** `Corre! Essa oferta é por tempo limitado.`

**Conteúdo:**
- Banner de urgência no topo: fundo colorido com "⚡ OFERTA RELÂMPAGO"
- Card da promoção igual ao TEMPLATE-04 com destaque visual maior
- Contador textual de tempo restante (ex: "Encerra em 3h 45min")
- Estoque restante (se informado): "Apenas 23 unidades disponíveis"
- Botão CTA primário: **"Garantir agora"** → URL da promoção
- Aviso: "Este e-mail foi gerado automaticamente. O preço pode ter mudado."

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{flash_ends_at_formatted}}` | Data/hora de encerramento formatada |
| `{{time_remaining}}` | Tempo restante em texto (ex: "3h 45min") |
| `{{flash_stock_remaining}}` | Estoque restante (opcional, omitido se não informado) |
| (+ todas as variáveis do TEMPLATE-04) | |

---

##### TEMPLATE-06 — Favorito Prestes a Expirar

**Gatilho:** promoção favoritada pelo usuário com menos de 2 horas para expirar.
**Assunto:** `⏰ Sua promoção favorita encerra em breve — {{promotion_title}}`
**Preview text:** `Você salvou essa oferta. Ela encerra em {{time_remaining}}.`

**Conteúdo:**
- Saudação com nome
- Mensagem: "Você salvou esta promoção nos favoritos. Ela está prestes a encerrar."
- Card compacto da promoção com preço e desconto
- Botão CTA primário: **"Ver oferta antes que acabe"** → URL da promoção
- Link secundário: "Remover dos favoritos"

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{promotion_title}}` | Título da promoção |
| `{{promotion_url}}` | URL da promoção na plataforma |
| `{{product_image_url}}` | Imagem do produto |
| `{{promo_price}}` | Preço promocional |
| `{{discount_percent}}` | Percentual de desconto |
| `{{time_remaining}}` | Tempo restante em texto |
| `{{remove_favorite_url}}` | Link para remover dos favoritos |

---

##### TEMPLATE-06b — Eventos de Favorito (variantes do TEMPLATE-06)

**Gatilho:** qualquer evento de favorito além do `favorite_expiring_soon` (já coberto pelo TEMPLATE-06).
**Base visual:** mesmo layout do TEMPLATE-06, com adaptações por variante.

| Variante | Assunto | Preview text | Diferencial visual |
|---|---|---|---|
| `flash_expiring_soon` | `⚡ Sua oferta favorita encerra em {{time_remaining}}!` | `Apenas {{flash_stock_remaining}} unidades restantes.` | Banner de urgência vermelho + contador em minutos |
| `flash_low_stock` | `⚠️ Estoque acabando na sua promoção favorita` | `Restam apenas {{flash_stock_remaining}} unidades.` | Badge "⚠️ ESTOQUE CRÍTICO" + barra de progresso do estoque |
| `expired` | `Sua promoção favorita encerrou — veja similares` | `Mas separamos 3 ofertas parecidas pra você.` | Tom informativo + seção de 3 sugestões da mesma categoria |
| `flash_ended` | `⚡ A oferta relâmpago que você salvou acabou` | `Encerrou por {{end_reason}}. Veja outras promoções.` | Indicação de motivo (prazo/estoque) + seção de 3 sugestões |
| `removed` | `Uma promoção dos seus favoritos foi removida` | `Encontramos outras ofertas que podem te interessar.` | Tom neutro + seção de 3 sugestões da mesma categoria |
| `price_dropped` | `📉 Ficou ainda mais barato! Preço baixou na sua favorita` | `De {{old_price}} por {{promo_price}}. Corre!` | Comparativo visual: preço antigo riscado → novo preço em destaque com badge "BAIXOU" |

**Variáveis específicas adicionais (além das do TEMPLATE-06):**

| Variável | Usado em | Descrição |
|---|---|---|
| `{{flash_stock_remaining}}` | flash_expiring_soon, flash_low_stock | Estoque restante |
| `{{end_reason}}` | flash_ended | `prazo` ou `estoque esgotado` |
| `{{old_price}}` | price_dropped | Preço promocional anterior |
| `{{price_drop_percent}}` | price_dropped | Percentual adicional de redução |
| `{{similar_promotions}}` | expired, flash_ended, removed | Array com 3 promoções: título, loja, desconto, imagem, URL |

##### TEMPLATE-07 — Conta Banida

**Gatilho:** moderador ou admin bane a conta do usuário.
**Assunto:** `Sua conta na {{platform_name}} foi suspensa`
**Preview text:** `Informações importantes sobre sua conta.`

**Conteúdo:**
- Saudação com nome
- Informação objetiva: conta suspensa
- Motivo do banimento (se fornecido pelo moderador)
- Data de início da suspensão
- Tipo: permanente ou temporária (com data de término se temporária)
- Link para contestar: "Se acredita que houve um erro, entre em contato." → link de suporte
- Sem botão CTA principal — tom sóbrio e informativo

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{ban_reason}}` | Motivo do banimento |
| `{{ban_type}}` | `permanent` ou `temporary` |
| `{{ban_ends_at_formatted}}` | Data de término (apenas se temporário) |
| `{{ban_datetime}}` | Data e hora do banimento |
| `{{support_url}}` | Link para contestação |

---

##### TEMPLATE-08 — Promoção Removida (Publicador)

**Gatilho:** promoção do usuário é removida por qualquer motivo.
**Assunto:** `Sua promoção foi removida — {{promotion_title}}`
**Preview text:** `Veja o motivo e como publicar novamente.`

**Conteúdo:**
- Saudação com nome
- Identificação da promoção removida (título + loja)
- Motivo da remoção em destaque, diferenciado visualmente por tipo:
  - 🔴 Cupom inválido: "O cupom foi reportado como inválido por X usuários"
  - 🟠 Preço expirado: "O preço promocional não está mais disponível na loja"
  - ⚠️ Moderação: "Removida por [nome do moderador]. Motivo: [motivo]"
- Orientação: o que o usuário pode fazer (ex: verificar e republicar se ainda válida)
- Botão CTA: **"Publicar nova promoção"** → página de publicação
- Link secundário: "Falar com suporte" (apenas para remoções por moderação)

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{promotion_title}}` | Título da promoção removida |
| `{{store}}` | Loja da promoção |
| `{{removal_type}}` | `invalid_coupon`, `expired`, `moderation` |
| `{{removal_reason}}` | Descrição do motivo |
| `{{moderator_name}}` | Nome do moderador (apenas se `moderation`) |
| `{{report_count}}` | Número de reports (apenas se `invalid_coupon`) |
| `{{publish_url}}` | Link para publicar nova promoção |
| `{{support_url}}` | Link de suporte (apenas se `moderation`) |

---

##### TEMPLATE-09 — Promoção em Destaque (Publicador)

**Gatilho:** admin marca a promoção do usuário como destaque (`is_featured = true`).
**Assunto:** `🌟 Sua promoção foi colocada em destaque!`
**Preview text:** `Os editores escolheram sua oferta para o destaque da plataforma.`

**Conteúdo:**
- Saudação com nome
- Mensagem de parabéns com tom entusiasmado
- Card da promoção em destaque com badge "⭐ Em Destaque"
- Métricas atuais: visualizações, cliques, votos 🚀
- Botão CTA: **"Ver sua promoção em destaque"** → URL da promoção
- Mensagem de incentivo: "Continue publicando boas promoções!"

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{promotion_title}}` | Título da promoção |
| `{{promotion_url}}` | URL da promoção |
| `{{product_image_url}}` | Imagem do produto |
| `{{current_views}}` | Visualizações atuais |
| `{{current_clicks}}` | Cliques atuais |
| `{{current_rocket_votes}}` | Votos 🚀 Foguete atuais |

---

##### TEMPLATE-10 — Novo Comentário na Promoção (Publicador)

**Gatilho:** usuário comenta na promoção do publicador. Agrupado se mais de 3 em 10 minutos.
**Assunto:** `💬 {{commenter_name}} comentou na sua promoção` (unitário) ou `💬 {{comment_count}} pessoas comentaram na sua promoção` (agrupado)
**Preview text:** `"{{comment_preview}}"` (unitário) ou `Veja o que estão falando sobre sua oferta.` (agrupado)

**Conteúdo (unitário):**
- Saudação
- Identificação da promoção comentada (título compacto)
- Avatar + nome do comentarista
- Texto completo do comentário em destaque (caixa com borda)
- Botão CTA: **"Responder comentário"** → URL da promoção com âncora no comentário

**Conteúdo (agrupado):**
- Saudação
- Identificação da promoção
- Lista compacta dos últimos comentários (máx 3) com avatar, nome e prévia do texto
- Botão CTA: **"Ver todos os comentários"** → URL da promoção

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{is_grouped}}` | Boolean: unitário ou agrupado |
| `{{comment_count}}` | Quantidade de comentários (agrupado) |
| `{{commenter_name}}` | Nome do comentarista (unitário) |
| `{{commenter_avatar_url}}` | Avatar do comentarista (unitário) |
| `{{comment_text}}` | Texto do comentário (unitário) |
| `{{comment_preview}}` | Prévia do comentário para preview text |
| `{{comments}}` | Array de comentários (agrupado): nome, avatar, prévia |
| `{{promotion_title}}` | Título da promoção |
| `{{promotion_url}}` | URL da promoção com âncora |

---

##### TEMPLATE-11 — Resumo de Encerramento Relâmpago (Publicador)

**Gatilho:** promoção relâmpago do publicador encerra (por prazo ou estoque).
**Assunto:** `⚡ Sua promoção relâmpago encerrou — veja o desempenho`
**Preview text:** `{{total_clicks}} cliques em {{duration_formatted}}. Confira os números.`

**Conteúdo:**
- Saudação
- Identificação da promoção (título, loja, desconto)
- Motivo do encerramento: prazo atingido ou estoque esgotado (com tom positivo se estoque esgotado)
- **Painel de métricas** com cards individuais:
  - 👁️ Visualizações totais
  - 🖱️ Cliques no link
  - 👍 Votos 🚀 Foguete / 👎 Votos 🗑️ Lixo
  - ⭐ Salvamentos em favoritos
  - ⏱️ Tempo ativo (ex: "4h 32min")
- Botão CTA: **"Publicar nova promoção relâmpago"** → página de publicação

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{promotion_title}}` | Título da promoção |
| `{{store}}` | Loja |
| `{{discount_percent}}` | Percentual de desconto |
| `{{end_reason}}` | `expired` ou `out_of_stock` |
| `{{total_views}}` | Total de visualizações |
| `{{total_clicks}}` | Total de cliques |
| `{{total_rocket_votes}}` | Total de votos 🚀 |
| `{{total_trash_votes}}` | Total de votos 🗑️ |
| `{{total_saves}}` | Total de salvamentos |
| `{{duration_formatted}}` | Duração em texto (ex: "4h 32min") |
| `{{flash_started_at}}` | Quando iniciou |
| `{{flash_ended_at}}` | Quando encerrou |
| `{{publish_flash_url}}` | Link para publicar nova relâmpago |

---

##### TEMPLATE-12 — Canal de Notificação com Problema (Unreachable)

**Gatilho:** envio por WhatsApp ou push falha 2 vezes consecutivas e o canal é marcado como `unreachable`.
**Assunto:** `⚠️ Problema no envio de notificações — ação necessária`
**Preview text:** `Seu canal de {{channel_name}} não está recebendo notificações.`

**Conteúdo:**
- Saudação
- Identificação do canal com problema (WhatsApp ou Push)
- Explicação simples do problema (sem jargão técnico)
- Passo a passo visual de como reconfigurar o canal
- Botão CTA: **"Reconfigurar notificações"** → link direto para o painel do canal específico
- Link secundário: "Desativar este canal" (caso o usuário não queira mais usar)

**Variáveis específicas:**

| Variável | Descrição |
|---|---|
| `{{channel_name}}` | Nome amigável do canal (ex: "WhatsApp", "Notificações do navegador") |
| `{{channel_type}}` | `whatsapp` ou `browser` |
| `{{failure_reason}}` | Motivo técnico simplificado |
| `{{reconfigure_url}}` | Link direto para reconfiguração do canal |
| `{{disable_channel_url}}` | Link para desativar o canal |

---

##### TEMPLATE-13 — Newsletter "Em Alta" (Paper de Notícias)

**Gatilho:** job agendado diário (9h) ou semanal (segunda às 9h). HTML gerado pelo Claude.
**Assunto:** gerado pelo Claude (ex: `🔥 As 10 melhores ofertas dessa segunda, {{date_formatted}}`)
**Preview text:** gerado pelo Claude

**Estrutura que o Claude deve gerar (passada no prompt):**

```
┌─────────────────────────────────────┐
│  HEADER padrão da plataforma        │
├─────────────────────────────────────┤
│  HERO da newsletter                 │
│  Título criativo da edição          │
│  Parágrafo de introdução (2-3 lin.) │
├─────────────────────────────────────┤
│  CARD DESTAQUE (promoção #1)        │
│  Imagem grande + título + loja      │
│  Preço + desconto + botão CTA       │
├─────────────────────────────────────┤
│  GRID 2 COLUNAS (promoções #2-#5)  │
│  Card compacto por promoção         │
├─────────────────────────────────────┤
│  DIVISOR "Mais em Alta"             │
├─────────────────────────────────────┤
│  GRID 2 COLUNAS (promoções #6-#10) │
│  Card compacto por promoção         │
├─────────────────────────────────────┤
│  PARÁGRAFO de encerramento          │
│  CTA: "Ver todas no site"           │
├─────────────────────────────────────┤
│  FOOTER padrão + unsubscribe        │
└─────────────────────────────────────┘
```

**Prompt enviado ao Claude para geração:**
```
Você é o redator da newsletter "Em Alta" da plataforma {{platform_name}}.
Gere um e-mail HTML responsivo completo seguindo EXATAMENTE esta estrutura:
1. Hero com título criativo para {{frequency}} de {{date_formatted}} e parágrafo de introdução (2-3 linhas, tom informal e animado)
2. Card destaque para a promoção #1 (imagem grande, preço, desconto, botão "Ver oferta")
3. Grid 2 colunas para as promoções #2 a #5 (cards compactos)
4. Seção "Mais em Alta" com grid 2 colunas para as promoções #6 a #10
5. Parágrafo de encerramento com CTA para o site

Use CSS inline. Largura máxima 600px. Cores da marca: primária {{platform_primary_color}}.
Retorne APENAS o HTML do body (sem <html>, <head> ou <body> — apenas o conteúdo interno).
Não inclua explicações, apenas o HTML.

Promoções (em ordem de trend_score):
{{promotions_json}}
```

**Variáveis injetadas no prompt:**

| Variável | Descrição |
|---|---|
| `{{frequency}}` | `edição diária` ou `edição semanal` |
| `{{date_formatted}}` | Data formatada (ex: "segunda-feira, 16 de junho") |
| `{{promotions_json}}` | JSON com as 10 promoções: título, loja, preço original, preço promo, desconto, imagem, URL |
| `{{platform_primary_color}}` | Cor primária da plataforma |

**Pós-processamento do HTML gerado:**
- Sanitização: remoção de qualquer `<script>`, `onclick`, `onerror` e atributos de evento
- Injeção do header e footer padrão da plataforma (não gerados pelo Claude)
- Injeção do link de unsubscribe no footer
- Armazenamento do HTML final em `newsletter_editions.html_content`

---

#### RN-022.3 — Implementação Técnica dos Templates

**Engine de template:** os templates HTML são armazenados como arquivos `.html` no repositório com placeholders no formato `{{variavel}}`. A substituição é feita pelo backend Go em tempo de envio.

**Estrutura de arquivos:**
```
backend/
└── internal/
    └── infrastructure/
        └── email/
            ├── templates/
            │   ├── base/
            │   │   ├── layout.html       ← estrutura base reutilizável
            │   │   ├── header.html       ← header com logo
            │   │   └── footer.html       ← footer com links e unsubscribe
            │   ├── 01_confirm.html
            │   ├── 02_welcome.html
            │   ├── 03_reset_password.html
            │   ├── 04_promotion_alert.html
            │   ├── 05_flash_alert.html
            │   ├── 06_favorite_expiring.html
            │   ├── 06b_favorite_flash_expiring.html
            │   ├── 06b_favorite_flash_low_stock.html
            │   ├── 06b_favorite_expired.html
            │   ├── 06b_favorite_flash_ended.html
            │   ├── 06b_favorite_removed.html
            │   └── 06b_favorite_price_dropped.html
            │   ├── 07_account_banned.html
            │   ├── 08_promotion_removed.html
            │   ├── 09_promotion_featured.html
            │   ├── 10_new_comment.html
            │   ├── 11_flash_summary.html
            │   ├── 12_channel_unreachable.html
            │   └── 13_newsletter.html    ← template do wrapper (header+footer)
            ├── renderer.go               ← lógica de render e substituição de variáveis
            └── sender.go                 ← lógica de envio via provider SMTP
```

**Responsabilidades do `renderer.go`:**
- Carregar o template pelo identificador
- Injetar variáveis globais automaticamente
- Injetar variáveis específicas do template
- Gerar versão plain text automaticamente (strip de tags HTML)
- Retornar struct com: subject, html, plain_text, preview_text

**Regra de identidade visual:**
- Enquanto a identidade visual não estiver definida, os templates usam placeholders de cor (`{{platform_primary_color}}`) e logo (`{{platform_logo_url}}`) que são substituídos em tempo de render pelas variáveis de ambiente `PLATFORM_PRIMARY_COLOR` e `PLATFORM_LOGO_URL`.
- Ao definir a identidade visual, basta atualizar essas variáveis de ambiente e todos os templates refletem a mudança automaticamente.

### RN-021.7b — Notificações para Publicadores de Promoções

Usuários que publicam promoções na plataforma recebem notificações exclusivas sobre suas próprias promoções. Essas notificações são independentes das preferências de canal configuradas para alertas de promoções — o usuário configura separadamente quais canais deseja receber notificações de publicador.

**Eventos que geram notificação para o publicador:**

| Evento | Gatilho | Mensagem exemplo |
|---|---|---|
| `publisher_comment` | Alguém comentou na promoção | "Lucas comentou na sua promoção: 'SSD Kingston'" |
| `publisher_reply` | Alguém respondeu um comentário do publicador | "Maria respondeu seu comentário na promoção 'Notebook Dell'" |
| `publisher_promotion_removed_invalid_coupon` | Promoção removida por cupom inválido (5+ reports) | "Sua promoção foi removida: o cupom PROMO10 foi reportado como inválido por vários usuários" |
| `publisher_promotion_removed_expired` | Promoção removida por expiração confirmada | "Sua promoção 'TV Samsung' foi removida pois o preço retornou ao normal" |
| `publisher_promotion_removed_moderation` | Promoção removida manualmente por moderador | "Sua promoção foi removida por um moderador. Motivo: [motivo]" |
| `publisher_promotion_featured` | Admin marcou a promoção como destaque | "🌟 Sua promoção 'SSD Kingston' foi colocada em destaque pelos editores!" |
| `publisher_promotion_top_rated` | Promoção se tornou a mais bem avaliada da plataforma | "🏆 Sua promoção 'SSD Kingston' é agora a mais bem avaliada da plataforma!" |
| `publisher_promotion_trending` | Promoção entrou no top 3 do trend_score global | "🔥 Sua promoção está bombando! Entrou no top 3 em alta agora" |
| `publisher_flash_ended_expired` | Promoção relâmpago encerrada por prazo | "⚡ Sua promoção relâmpago 'SSD Kingston' encerrou. Foram X cliques em Y horas" |
| `publisher_flash_ended_out_of_stock` | Promoção relâmpago encerrada por estoque esgotado | "⚡ Sua promoção relâmpago esgotou o estoque em X minutos! Parabéns" |

**Regras específicas:**

- Notificações de comentário (`publisher_comment`) são agrupadas quando há mais de 3 comentários em menos de 10 minutos no mesmo post, para evitar spam ao publicador (ex: "5 pessoas comentaram na sua promoção").
- A notificação `publisher_promotion_top_rated` é disparada apenas uma vez por promoção, quando ela assume a primeira posição no ranking de `score`. Se perder e reconquistar a posição, não notifica novamente.
- A notificação `publisher_promotion_trending` é disparada apenas quando a promoção entra no **top 3** do trend_score global (não por loja ou categoria), e no máximo uma vez por hora para a mesma promoção.
- Notificações de remoção sempre incluem o motivo detalhado. Quando removida por moderação, o publicador recebe também o nome do moderador responsável (somente o nome de exibição).
- Para promoções relâmpago encerradas, a notificação inclui um **resumo de desempenho**: total de visualizações, cliques, votos 🚀 Foguete / 🗑️ Lixo e tempo que ficou ativa.
- O publicador pode configurar individualmente quais eventos de publicador deseja receber e em quais canais, de forma independente das preferências gerais de notificação.
- Notificações de publicador têm prioridade `high` na fila RabbitMQ — são processadas antes de notificações comuns de promoção.

**Configuração de canais para notificações de publicador:**

O usuário pode ativar/desativar cada tipo de evento individualmente e escolher o canal (browser, e-mail, WhatsApp, Telegram) para cada grupo:

| Grupo de eventos | Eventos incluídos |
|---|---|
| `comments` | `publisher_comment`, `publisher_reply` |
| `removals` | `publisher_promotion_removed_*` (todos os tipos de remoção) |
| `achievements` | `publisher_promotion_featured`, `publisher_promotion_top_rated`, `publisher_promotion_trending` |
| `flash_summary` | `publisher_flash_ended_expired`, `publisher_flash_ended_out_of_stock` |

#### RN-021.8 — Regras Gerais de Notificação

- Nenhum canal envia notificação para promoções com status `pending_review` — apenas promoções `active` ou `flash_active` disparam notificações.
- Notificações de promoções relâmpago têm prioridade de envio sobre notificações comuns na fila.
- Se um canal falhar no envio (ex: token de push expirado, número de WhatsApp inválido), o sistema tenta novamente 1 vez após 5 minutos. Se falhar novamente, o canal é marcado como `unreachable` e o usuário é notificado por e-mail para reconfigurar.
- O usuário pode desativar todos os canais de uma vez com o botão "Pausar todas as notificações" no painel de preferências.
- Logs de envio (sucesso/falha por canal e por notificação) são mantidos por 30 dias para fins de diagnóstico.

### RN-019 — Promoções Relâmpago

**Definição:**
Promoções relâmpago são ofertas com duração máxima de 24 horas, estoque limitado ou ambos. Elas possuem tratamento especial na plataforma: destaque visual diferenciado, contador regressivo em tempo real e notificação imediata para usuários interessados.

**Criação:**
- Promoções relâmpago podem ser criadas pelo bot automatizado (quando a fonte original já as classifica como relâmpago, ex: "Oferta do Dia" do Mercado Livre, "Flash Sale" da Shopee) ou por usuários/admin manualmente.
- Para marcar uma promoção como relâmpago, os seguintes campos adicionais são obrigatórios:
  - `is_flash = true`
  - `flash_ends_at`: data/hora de encerramento (obrigatório, máximo 24h a partir da criação)
  - `flash_stock`: quantidade de unidades disponíveis (opcional — quando informado, a promoção encerra ao esgotar o estoque)

**Comportamento:**
- Promoções relâmpago aparecem em uma seção exclusiva no topo do feed principal: **"⚡ Relâmpago"**.
- O contador regressivo é exibido em tempo real via WebSocket no card e na página de detalhe.
- Quando `flash_stock` é informado, o estoque restante é exibido ao lado do contador (ex: "12 restantes").
- Quando o estoque chega a 0 ou `flash_ends_at` é atingido, a promoção é encerrada automaticamente com status `flash_ended` e removida da seção relâmpago.
- Promoções com menos de 1 hora restante recebem destaque visual adicional (ex: contador vermelho piscando).
- O encerramento é processado pelo job de expiração (a cada 1 minuto para promoções relâmpago, mais frequente que o job padrão).

**Notificações:**
- Usuários que configuraram alerta para a categoria ou loja da promoção recebem notificação imediata ao surgir uma promoção relâmpago.
- Usuários que salvaram a promoção nos favoritos recebem notificação 30 minutos antes do encerramento.
- Usuários podem se inscrever em "alertas relâmpago" por categoria ou loja para receber push/e-mail sempre que uma flash sale surgir naquele contexto.

**Estoque:**
- O campo `flash_stock` representa o estoque declarado pelo criador. O sistema não integra com estoque real das lojas.
- O sistema decrementa `flash_stock_remaining` a cada clique no botão "Ver oferta" como estimativa de demanda (não é estoque real garantido).
- Quando `flash_stock_remaining` chega a 0, a promoção é marcada como `flash_ended` automaticamente.
- O criador da promoção (usuário ou admin) pode atualizar o estoque manualmente se necessário.

**Status exclusivos de promoções relâmpago:**

| Status | Descrição |
|---|---|
| `flash_active` | Relâmpago ativa, dentro do prazo e com estoque |
| `flash_ended` | Encerrada por prazo ou estoque esgotado |

> Promoções relâmpago também passam pelos status normais (`pending_review`, `removed`) antes de se tornarem `flash_active`.

### RN-023 — Bot de Consulta via Telegram

O bot do Telegram serve dois propósitos distintos e independentes:

1. **Canal de notificações** (já documentado na RN-021.5) — recebe alertas automáticos de promoções conforme as preferências configuradas pelo usuário na plataforma.
2. **Canal de consulta interativa** (esta regra) — o usuário conversa com o bot para buscar promoções sob demanda, sem precisar acessar o site.

---

#### RN-023.1 — Acesso e Autenticação no Bot

- O bot é acessado pelo username configurado (ex: `@EmAltaPromocoesBot`).
- Usuários **não cadastrados** na plataforma podem usar o bot em modo limitado: apenas busca por palavra-chave, sem personalização.
- Usuários **cadastrados** conectam o bot à sua conta via código de verificação (mesmo fluxo da RN-021.5). Após conectados, o bot acessa suas categorias de interesse e histórico.
- O comando `/conectar` inicia o fluxo de vinculação com a conta da plataforma.
- O comando `/desconectar` desvincula o bot da conta, mantendo apenas o modo limitado.

---

#### RN-023.2 — Comandos Disponíveis

| Comando | Descrição |
|---|---|
| `/start` | Mensagem de boas-vindas + lista de comandos disponíveis |
| `/conectar` | Inicia fluxo de vinculação com a conta da plataforma |
| `/desconectar` | Desvincula a conta |
| `/hoje` | Retorna as melhores promoções do dia (top 10 por trend_score) |
| `/relampago` | Lista as promoções relâmpago ativas no momento |
| `/categoria [nome]` | Busca promoções por categoria (ex: `/categoria eletronicos`) |
| `/loja [nome]` | Busca promoções por loja (ex: `/loja amazon`) |
| `/meus` | Retorna promoções das categorias de interesse do usuário conectado |
| `/ajuda` | Lista todos os comandos com exemplos de uso |
| `/pausar` | Pausa todas as notificações automáticas (não afeta consultas) |
| `/ativar` | Reativa as notificações automáticas |

---

#### RN-023.3 — Busca por Palavra-chave (Digitação Livre)

- Qualquer mensagem enviada ao bot que **não seja um comando** (não começa com `/`) é interpretada como uma busca por palavra-chave.
- O bot realiza uma busca full-text no título e descrição das promoções ativas usando o termo digitado.
- Exemplos de entrada: `teclado`, `notebook gamer`, `fone bluetooth`, `tv 55 polegadas`.
- A busca é case-insensitive e sem acento (utiliza a mesma função `TRANSLATE` já implementada no backend para accent-insensitive search).
- Resultado mínimo: 1 promoção. Resultado máximo por resposta: 5 promoções.
- Se não houver resultados: o bot responde com "Nenhuma promoção encontrada para *[termo]*. Tente outro termo ou use /hoje para ver as melhores ofertas do momento."

---

#### RN-023.4 — Paginação das Respostas

- Resultados são enviados em blocos de **5 promoções por vez**.
- Após cada bloco, o bot exibe um botão inline **"Ver mais 5 →"** caso existam mais resultados.
- O usuário clica no botão para receber o próximo bloco — sem precisar redigitar o comando.
- Máximo de **3 páginas** (15 promoções) por consulta. Após a terceira página, o bot sugere refinar a busca.
- Cada página usa `callback_query` do Telegram para manter o contexto da busca sem poluir o chat com mensagens repetidas.

---

#### RN-023.5 — Formato das Respostas

Cada promoção é enviada como uma **mensagem individual formatada em Markdown** com:

```
🏷️ *[Título da promoção]*
🏪 Loja: [Nome da loja]
💰 De: ~~R$ [preço original]~~ por *R$ [preço promocional]*
🔥 Desconto: *[X]% OFF*
🎟️ Cupom: `[CODIGO]`  ← omitido se não houver cupom
⏰ Encerra em: [tempo restante]  ← omitido se não tiver prazo
🔗 [Ver oferta](URL da promoção na plataforma)
```

- Promoções relâmpago recebem o badge `⚡ RELÂMPAGO` no título.
- Promoções em destaque recebem o badge `⭐ DESTAQUE` no título.
- O link "Ver oferta" redireciona para a página da promoção na plataforma (não direto para a loja), registrando o clique nas métricas.
- Após o bloco de 5 promoções, uma mensagem de resumo: "Mostrando [1-5] de [total] resultados para *[termo]*."

---

#### RN-023.6 — Comando `/meus` (Usuário Conectado)

- Retorna as melhores promoções ativas das categorias de interesse do usuário, ordenadas por `trend_score`.
- Se o usuário não tiver categorias configuradas, o bot sugere: "Você ainda não configurou suas categorias. Acesse [link] para configurar."
- Funciona como um feed personalizado via Telegram — equivalente ao feed da plataforma filtrado pelas preferências do usuário.

---

#### RN-023.7 — Comando `/categoria` e `/loja`

- Aceita o nome completo ou parcial: `/categoria eletro` retorna promoções de "Eletrônicos".
- Se o termo for ambíguo (ex: `/categoria mo` pode ser "Moda" ou "Móveis"), o bot exibe botões inline para o usuário escolher qual categoria deseja.
- `/loja` funciona da mesma forma para filtrar por loja.
- Ambos os comandos combinam com paginação (RN-023.4).

---

#### RN-023.8 — Rate Limiting do Bot

- Máximo de **20 consultas por usuário por hora** para evitar abuso.
- Ao atingir o limite: "Você atingiu o limite de consultas por hora. Tente novamente em [X] minutos."
- Notificações automáticas (RN-021.5) não contam para esse limite — são independentes.
- Usuários não autenticados têm limite menor: **10 consultas por hora** por `chat_id`.

---

#### RN-023.9 — Fluxo de Conversa Completo

```
Usuário: teclado
Bot: 🔍 Buscando promoções para "teclado"...

Bot: ⭐ DESTAQUE
     🏷️ *Teclado Mecânico Redragon K552*
     🏪 Loja: Kabum
     💰 De: ~~R$ 299,90~~ por *R$ 149,90*
     🔥 Desconto: *50% OFF*
     🔗 Ver oferta

[... mais 4 promoções individuais ...]

Bot: Mostrando 1-5 de 12 resultados para *teclado*.
     [Ver mais 5 →]

Usuário: [clica em "Ver mais 5 →"]
Bot: [envia as próximas 5 promoções]
```

```
Usuário: /categoria
Bot: Qual categoria você quer ver?
     [Eletrônicos] [Games] [Moda] [Casa] [Mercado]
     [Beleza] [Esportes] [Viagens] [Outros]

Usuário: [clica em "Eletrônicos"]
Bot: 🔍 Promoções em *Eletrônicos*...
     [5 promoções formatadas + paginação]
```

### RN-020 — Promoções por Tempo Limitado (não relâmpago)

- Promoções comuns (não relâmpago) também podem ter prazo de validade definido via `expires_at`, sem as restrições de 24h e sem a seção especial de relâmpago.
- Essas promoções são exibidas normalmente no feed, mas com um indicador de prazo quando faltarem menos de 6 horas para o encerramento.
- A ordenação `expiring_soon` lista todas as promoções com `expires_at` definido, ordenadas pelo prazo mais próximo, incluindo as relâmpago.

### RN-016 — Métricas de Engajamento

- Toda interação do usuário com uma promoção é rastreada para alimentar o sistema de destaques e rankings. As métricas rastreadas são:
  - **Visualizações** (`views`): incrementada toda vez que a página de detalhe da promoção é aberta.
  - **Cliques no link** (`clicks`): incrementada toda vez que o usuário clica no botão "Ver oferta" (redirecionamento para o site externo).
  - **Votos 🚀 Foguete / 🗑️ Lixo** (`rocket_votes`, `trash_votes`): já existentes.
  - **Salvamentos em favoritos** (`saves`): toda vez que um usuário adiciona a promoção aos favoritos.
  - **Compartilhamentos** (`shares`): toda vez que o usuário usa o botão de compartilhar a promoção.
- Visualizações e cliques são registrados de forma assíncrona via fila (RabbitMQ) para não impactar a performance da requisição principal.
- Visualizações de um mesmo usuário autenticado na mesma promoção dentro de 1 hora são contadas apenas uma vez.
- Visualizações de visitantes anônimos são contadas por IP + User-Agent com janela de 30 minutos.
- Todas as métricas são armazenadas em duas camadas:
  - **Redis**: contadores em tempo real (janela de 24h e 7d), usados para o cálculo de tendência.
  - **PostgreSQL**: totais históricos acumulados na tabela `promotions` e série temporal na tabela `promotion_metrics`.

### RN-017 — Destaque Automático por Tendência

- O sistema calcula automaticamente um **score de tendência** (`trend_score`) para cada promoção ativa, atualizado a cada 5 minutos via job agendado.
- A fórmula do trend_score considera:
  - Cliques nas últimas 1h (peso 5)
  - Visualizações nas últimas 1h (peso 2)
  - Votos 🚀 Foguete nas últimas 1h (peso 4)
  - Salvamentos nas últimas 1h (peso 3)
  - Compartilhamentos nas últimas 1h (peso 3)
- O trend_score decai ao longo do tempo (decaimento logarítmico): promoções mais antigas precisam de mais engajamento para manter a posição.
- As **N promoções com maior trend_score** (configurável, padrão: 10) recebem automaticamente a flag `is_trending = true`.
- `is_trending` é diferente de `is_featured`: `is_trending` é calculado automaticamente pelo sistema; `is_featured` é definido manualmente pelo admin.
- Uma promoção pode ser `is_trending = true` e `is_featured = true` ao mesmo tempo.
- O trend_score é calculado globalmente e também **por loja** e **por categoria**, permitindo que cada contexto de filtro tenha seu próprio ranking de tendência.

### RN-018 — Ordenações e Filtros

**Ordenações disponíveis** (aplicáveis em qualquer contexto de listagem):

| Ordenação | Descrição | Critério |
|---|---|---|
| `trending` | Em alta agora | Maior `trend_score` nas últimas 1h |
| `top_rated` | Mais bem avaliadas | Maior `score` (rocket - trash) de todos os tempos |
| `most_clicked` | Mais clicadas | Maior `clicks` total |
| `newest` | Mais recentes | Maior `created_at` |
| `biggest_discount` | Maior desconto | Maior `discount_percent` |
| `expiring_soon` | Prestes a expirar | Menor `expires_at` (apenas promoções com data definida) |

**Filtros disponíveis** (combináveis entre si e com qualquer ordenação):

| Filtro | Parâmetro | Valores |
|---|---|---|
| Por loja | `store` | `mercadolivre`, `amazon`, `shopee`, `kabum`, `pichau`, `terabyte`, `magalu`, `americanas` |
| Por categoria | `category` | slug da categoria (ex: `eletronicos`, `roupas`, `games`) |
| Por faixa de desconto | `min_discount` / `max_discount` | 0 a 100 (%) |
| Por faixa de preço | `min_price` / `max_price` | valor decimal |
| Apenas com cupom | `has_coupon` | `true` / `false` |
| Apenas em destaque | `featured` | `true` / `false` |
| Apenas em tendência | `trending` | `true` / `false` |

**Combinações esperadas de uso:**
- Feed principal → sem filtro, ordenação `trending`
- Seção "Mais bem avaliadas" → sem filtro, ordenação `top_rated`
- Amazon em alta → `store=amazon`, ordenação `trending`
- Eletrônicos mais votados → `category=eletronicos`, ordenação `top_rated`
- Roupas com maior desconto → `category=roupas`, ordenação `biggest_discount`
- Kabum mais clicadas → `store=kabum`, ordenação `most_clicked`

**Regras de combinação:**
- Todos os filtros são combináveis entre si.
- Todas as ordenações funcionam dentro de qualquer filtro ativo.
- A paginação é aplicada após filtragem e ordenação.
- O trend_score usado na ordenação `trending` dentro de um filtro de loja ou categoria é o trend_score **específico daquele contexto**, não o global.

---

## Arquitetura de Software

### Monorepo

O projeto é organizado como um **monorepo** contendo todos os projetos de forma independente, cada um com seu próprio toolchain, dependências e ciclo de build. A raiz do repositório contém apenas arquivos de orquestração (Docker Compose, CI/CD, scripts compartilhados).

```
promo-platform/                         # raiz do monorepo
├── apps/
│   ├── backend/                        # API Go (DDD)
│   ├── frontend/                       # SPA Angular
│   ├── workers/                        # Workers de coleta e scraping (Go)
│   └── scheduler/                      # Jobs agendados: trend, newsletter, expiração (Go)
├── infra/
│   ├── docker/                         # Dockerfiles por app
│   ├── nginx/                          # Configuração do API Gateway
│   └── migrations/                     # Migrations SQL (compartilhadas)
├── .github/
│   └── workflows/                      # CI/CD pipelines por app
├── docker-compose.yml                  # Orquestração local completa
├── docker-compose.prod.yml             # Orquestração produção
└── Makefile                            # Comandos utilitários (make dev, make migrate, etc.)
```

---

### Visão Geral da Infraestrutura

```
┌──────────────────────────────────────────────────────────┐
│                      CLIENTES                            │
│              Browser (Angular SPA)                       │
└───────────────────────┬──────────────────────────────────┘
                        │ HTTPS
┌───────────────────────▼──────────────────────────────────┐
│                 NGINX (API Gateway)                      │
│         Rate limiting, SSL termination, proxy reverso    │
└──────┬────────────────────────────────────┬──────────────┘
       │ /api/*                             │ /ws/*
┌──────▼──────────┐               ┌─────────▼──────────────┐
│   apps/backend  │               │   WebSocket Server      │
│   Go / Gin      │               │   (apps/backend)        │
│   :8080         │               │   :8081                 │
└──────┬──────────┘               └────────────────────────┘
       │
┌──────▼─────────────────────────────────────────────────┐
│              Bounded Contexts (DDD)                    │
│  identity │ catalog │ engagement │ notification │ ...  │
└──────┬────────────────────┬──────────────────────────┬─┘
       │                    │                          │
┌──────▼──────┐  ┌──────────▼──────┐  ┌───────────────▼──┐
│ PostgreSQL  │  │     Redis        │  │    RabbitMQ       │
│ (dados)     │  │ (cache/sessões/  │  │ (eventos de       │
│             │  │  contadores)     │  │  domínio)         │
└─────────────┘  └─────────────────┘  └──────────┬────────┘
                                                  │
                              ┌───────────────────▼────────┐
                              │   apps/workers             │
                              │   Coleta: ML, Shopee,      │
                              │   Amazon, Kabum...         │
                              └────────────────────────────┘
                              ┌─────────────────────────────┐
                              │   apps/scheduler            │
                              │   trend_score, newsletter,  │
                              │   expiração, flash_check    │
                              └─────────────────────────────┘
```

---

### Domain-Driven Design — Bounded Contexts

A aplicação é estruturada em torno de **Bounded Contexts** independentes. Cada contexto possui seu próprio modelo de domínio, linguagem ubíqua e fronteiras claras. A comunicação entre contextos ocorre **exclusivamente via eventos de domínio** publicados no RabbitMQ — nunca por chamada direta entre repositórios de contextos distintos.

#### Mapa de Contextos

| Bounded Context | Aggregate Roots | Responsabilidade |
|---|---|---|
| `identity` | `User` | Cadastro, autenticação, perfil, dados pessoais |
| `catalog` | `Promotion`, `Coupon`, `Category` | Promoções, cupons, categorias |
| `engagement` | `Vote`, `Comment`, `Favorite` | Votos, comentários, favoritos, métricas |
| `notification` | `NotificationPreference` | Preferências e envio de notificações |
| `newsletter` | `NewsletterSubscription`, `NewsletterEdition` | Geração e envio do digest por IA |
| `trending` | `TrendScore` | Cálculo de rankings e scores de tendência |
| `collection` | `CollectionJob` | Workers de scraping e APIs de afiliados |
| `moderation` | `ModerationTicket` | Fila de moderação e denúncias |

#### Fluxo de Eventos entre Contextos

```
identity       → UserRegistered       → notification (envia TEMPLATE-01)
identity       → UserConfirmed        → notification (envia TEMPLATE-02)
identity       → UserBanned           → notification (envia TEMPLATE-07)

catalog        → PromotionPublished   → trending, notification, moderation
catalog        → PromotionApproved    → notification (dispara alertas de categoria/loja)
catalog        → PromotionFeatured    → notification (TEMPLATE-09 para publicador)
catalog        → PromotionRemoved     → notification (TEMPLATE-08 para publicador + favoritos)
catalog        → PromotionExpired     → notification (favoritos), trending
catalog        → PromotionPriceUpdated→ notification (favorite_price_dropped)
catalog        → FlashStarted         → notification (TEMPLATE-05 para inscritos)
catalog        → FlashEnded           → notification (TEMPLATE-11 para publicador + favoritos)
catalog        → FlashStockUpdated    → notification (favorite_flash_low_stock)

engagement     → VoteCast             → catalog (atualiza score), trending
engagement     → CommentPosted        → notification (TEMPLATE-10 para publicador)
engagement     → FavoriteAdded        → notification (cria favorite_notification_preferences)
engagement     → PromotionClicked     → trending (incrementa contador Redis)

trending       → TrendScoreUpdated    → catalog (atualiza trend_score + is_trending em batch)
trending       → Top3Reached          → notification (publisher_promotion_trending)

collection     → PromotionCollected   → catalog (publica promoção via command)

moderation     → TicketResolved       → catalog (remove ou aprova promoção)
```

---

### Estrutura Detalhada — `apps/backend`

A estrutura é **orientada por domínio/entidade**. Cada pasta representa um domínio do negócio e contém tudo que pertence a ele: controller, service, repository, model, DTO e eventos. Nenhum arquivo de um domínio fica espalhado fora de sua pasta.

```
apps/backend/
├── cmd/
│   └── api/
│       └── main.go                     # bootstrap: DI, rotas, middlewares, servidor HTTP
│
├── internal/
│   │
│   ├── users/                          # Domínio: Usuário
│   │   ├── user.go                     # Model / Entidade: User
│   │   ├── user_status.go              # Value Object: pending_confirmation | pending_profile | active | banned
│   │   ├── cpf.go                      # Value Object: validação + hash SHA-256 + criptografia AES-256
│   │   ├── email.go                    # Value Object: validação de formato
│   │   ├── password.go                 # Value Object: bcrypt hash
│   │   ├── phone.go                    # Value Object: formato E.164
│   │   ├── gender.go                   # Value Object: enum
│   │   ├── user_dto.go                 # DTOs: RegisterRequest, CompleteProfileRequest, UserResponse, etc.
│   │   ├── user_repository.go          # Interface: UserRepository
│   │   ├── user_repository_pg.go       # Implementação PostgreSQL
│   │   ├── user_service.go             # Serviço: regras de negócio de usuário
│   │   ├── user_controller.go          # Controller HTTP: rotas /auth/*, /users/*
│   │   ├── jwt_service.go              # Serviço: geração e validação de JWT
│   │   └── events/
│   │       ├── user_registered.go      # Evento de domínio
│   │       ├── user_confirmed.go
│   │       ├── user_profile_completed.go
│   │       └── user_banned.go
│   │
│   ├── promotions/                     # Domínio: Promoção
│   │   ├── promotion.go                # Model / Entidade: Promotion
│   │   ├── promotion_status.go         # Value Object: pending_review | active | flash_active | flash_ended | expired | archived | removed
│   │   ├── flash_config.go             # Value Object: flash_ends_at + flash_stock
│   │   ├── discount.go                 # Value Object: calcula discount_percent automaticamente
│   │   ├── store.go                    # Value Object: nome da loja
│   │   ├── promotion_dto.go            # DTOs: CreatePromotionRequest, PromotionResponse, PromotionListResponse, etc.
│   │   ├── promotion_repository.go     # Interface: PromotionRepository
│   │   ├── promotion_repository_pg.go  # Implementação PostgreSQL
│   │   ├── promotion_service.go        # Serviço: regras de negócio de promoção
│   │   ├── promotion_controller.go     # Controller HTTP: rotas /promotions/*
│   │   └── events/
│   │       ├── promotion_published.go
│   │       ├── promotion_approved.go
│   │       ├── promotion_featured.go
│   │       ├── promotion_removed.go
│   │       ├── promotion_expired.go
│   │       ├── promotion_price_updated.go
│   │       ├── flash_started.go
│   │       ├── flash_ended.go
│   │       └── flash_stock_updated.go
│   │
│   ├── categories/                     # Domínio: Categoria
│   │   ├── category.go                 # Model / Entidade: Category
│   │   ├── category_dto.go             # DTOs: CreateCategoryRequest, CategoryResponse, etc.
│   │   ├── category_repository.go      # Interface: CategoryRepository
│   │   ├── category_repository_pg.go   # Implementação PostgreSQL
│   │   ├── category_service.go         # Serviço
│   │   └── category_controller.go      # Controller HTTP: rotas /categories/* e /admin/categories/*
│   │
│   ├── coupons/                        # Domínio: Cupom
│   │   ├── coupon.go                   # Model / Entidade: Coupon
│   │   ├── coupon_dto.go               # DTOs: CouponResponse, ReportCouponRequest, etc.
│   │   ├── coupon_repository.go        # Interface: CouponRepository
│   │   ├── coupon_repository_pg.go     # Implementação PostgreSQL
│   │   ├── coupon_service.go           # Serviço: validação, reports, desativação automática
│   │   ├── coupon_controller.go        # Controller HTTP: rotas /coupons/*
│   │   └── events/
│   │       └── coupon_invalidated.go
│   │
│   ├── votes/                          # Domínio: Voto (🚀 Foguete / 🗑️ Lixo)
│   │   ├── vote.go                     # Model / Entidade: Vote
│   │   ├── vote_type.go                # Value Object: rocket | trash
│   │   ├── vote_dto.go                 # DTOs: CastVoteRequest, VoteResponse
│   │   ├── vote_repository.go          # Interface: VoteRepository
│   │   ├── vote_repository_pg.go       # Implementação PostgreSQL
│   │   ├── vote_service.go             # Serviço: regras de votação (unicidade, sem auto-voto)
│   │   ├── vote_controller.go          # Controller HTTP: rotas /promotions/:id/vote
│   │   └── events/
│   │       ├── vote_cast.go
│   │       └── vote_removed.go
│   │
│   ├── comments/                       # Domínio: Comentário
│   │   ├── comment.go                  # Model / Entidade: Comment
│   │   ├── comment_dto.go              # DTOs: CreateCommentRequest, CommentResponse, etc.
│   │   ├── comment_repository.go       # Interface: CommentRepository
│   │   ├── comment_repository_pg.go    # Implementação PostgreSQL
│   │   ├── comment_service.go          # Serviço: criação, deleção, agrupamento de notificações
│   │   ├── comment_controller.go       # Controller HTTP: rotas /promotions/:id/comments
│   │   └── events/
│   │       ├── comment_posted.go
│   │       └── comment_reported.go
│   │
│   ├── favorites/                      # Domínio: Favorito
│   │   ├── favorite.go                 # Model / Entidade: Favorite
│   │   ├── favorite_dto.go             # DTOs: FavoriteResponse, etc.
│   │   ├── favorite_repository.go      # Interface: FavoriteRepository
│   │   ├── favorite_repository_pg.go   # Implementação PostgreSQL
│   │   ├── favorite_service.go         # Serviço: add, remove, busca de similares ao expirar
│   │   ├── favorite_controller.go      # Controller HTTP: rotas /users/me/favorites/*
│   │   └── events/
│   │       ├── favorite_added.go
│   │       └── favorite_removed.go
│   │
│   ├── metrics/                        # Domínio: Métricas de Engajamento
│   │   ├── promotion_metrics.go        # Model / Entidade: PromotionMetrics
│   │   ├── metrics_dto.go              # DTOs: MetricsResponse
│   │   ├── metrics_repository.go       # Interface: MetricsRepository
│   │   ├── metrics_repository_pg.go    # Implementação PostgreSQL (série temporal)
│   │   ├── metrics_counter_redis.go    # Implementação Redis (contadores em tempo real)
│   │   ├── metrics_service.go          # Serviço: incremento de views/clicks/shares
│   │   └── metrics_controller.go       # Controller HTTP: rotas /promotions/:id/click, /share
│   │
│   ├── trending/                       # Domínio: Trend Score
│   │   ├── trend_score.go              # Model / Entidade: TrendScore
│   │   ├── trend_context.go            # Value Object: global | by_store | by_category
│   │   ├── trending_dto.go             # DTOs: TrendingResponse
│   │   ├── trend_repository.go         # Interface: TrendRepository
│   │   ├── trend_repository_pg.go      # Implementação PostgreSQL
│   │   └── trend_service.go            # Serviço: fórmula de cálculo + batch update
│   │
│   ├── notifications/                  # Domínio: Notificação
│   │   ├── notification_preference.go          # Model: preferências por canal
│   │   ├── favorite_notification_preference.go # Model: preferências de favoritos
│   │   ├── publisher_notification_preference.go# Model: preferências de publicador
│   │   ├── push_subscription.go               # Model: token Web Push
│   │   ├── telegram_subscription.go           # Model: chat_id do Telegram
│   │   ├── notification_log.go                # Model: log de envio
│   │   ├── channel.go                         # Value Object: browser | email | whatsapp | telegram
│   │   ├── notification_dto.go                # DTOs: PreferenceRequest, PreferenceResponse, etc.
│   │   ├── notification_repository.go         # Interface: NotificationRepository
│   │   ├── notification_repository_pg.go      # Implementação PostgreSQL
│   │   ├── notification_service.go            # Serviço: orquestra envio por canal
│   │   ├── notification_controller.go         # Controller HTTP: rotas /notifications/*
│   │   └── senders/                           # Adaptadores de envio por canal
│   │       ├── email_sender.go                # Adaptador SMTP (Resend/SendGrid/SES)
│   │       ├── web_push_sender.go             # Adaptador VAPID
│   │       ├── whatsapp_sender.go             # Adaptador Meta Cloud API
│   │       └── telegram_sender.go             # Adaptador Telegram Bot API
│   │
│   ├── newsletter/                     # Domínio: Newsletter
│   │   ├── newsletter_subscription.go  # Model: inscrição
│   │   ├── newsletter_edition.go       # Model: edição gerada
│   │   ├── frequency.go                # Value Object: daily | weekly
│   │   ├── newsletter_dto.go           # DTOs: SubscribeRequest, EditionResponse, etc.
│   │   ├── newsletter_repository.go    # Interface: NewsletterRepository
│   │   ├── newsletter_repository_pg.go # Implementação PostgreSQL
│   │   ├── newsletter_service.go       # Serviço: geração via Claude API + envio em massa
│   │   ├── newsletter_controller.go    # Controller HTTP: rotas /newsletter/*
│   │   └── anthropic_generator.go      # Adaptador Claude API (geração do HTML)
│   │
│   ├── moderation/                     # Domínio: Moderação
│   │   ├── moderation_ticket.go        # Model: ticket de moderação
│   │   ├── ticket_type.go              # Value Object: promotion_report | comment_report | coupon_report
│   │   ├── ticket_status.go            # Value Object: open | in_review | resolved | dismissed
│   │   ├── moderation_dto.go           # DTOs: TicketResponse, ResolveTicketRequest, etc.
│   │   ├── moderation_repository.go    # Interface: ModerationRepository
│   │   ├── moderation_repository_pg.go # Implementação PostgreSQL
│   │   ├── moderation_service.go       # Serviço: abertura, resolução e descarte de tickets
│   │   ├── moderation_controller.go    # Controller HTTP: rotas /admin/moderation/*
│   │   └── events/
│   │       └── ticket_resolved.go
│   │
│   ├── telegrambot/                    # Domínio: Bot de Consulta Telegram
│   │   ├── bot_session.go              # Model: sessão de conversa (contexto de busca + página atual)
│   │   ├── bot_command.go              # Value Object: enum de comandos (/start, /hoje, /relampago, etc.)
│   │   ├── bot_dto.go                  # DTOs internos: SearchRequest, PromotionBotResponse
│   │   ├── bot_repository.go           # Interface: BotSessionRepository
│   │   ├── bot_repository_redis.go     # Implementação Redis (sessões temporárias + rate limit)
│   │   ├── bot_service.go              # Serviço: orquestra comandos, busca, paginação, rate limit
│   │   ├── bot_controller.go           # Webhook handler: recebe updates do Telegram e roteia
│   │   ├── bot_formatter.go            # Formata promoções em Markdown para o Telegram
│   │   └── bot_keyboard.go             # Gera teclados inline (paginação, seleção de categoria)
│   │
│   └── collection/                     # Domínio: Coleta de Promoções
│       ├── collection_job.go            # Model: job de coleta
│       ├── collected_promotion.go       # Model: dado bruto pré-normalização
│       ├── source.go                    # Value Object: api_mercadolivre | scraping_kabum | ...
│       ├── collection_repository.go     # Interface: CollectionRepository
│       ├── collection_repository_pg.go  # Implementação PostgreSQL
│       ├── collection_service.go        # Serviço: normalização + deduplicação + publicação
│       └── adapters/                    # Adaptadores por fonte
│           ├── mercadolivre_adapter.go
│           ├── shopee_adapter.go
│           ├── amazon_adapter.go
│           ├── kabum_scraper.go
│           ├── pichau_scraper.go
│           └── terabyte_scraper.go
│
├── middleware/                         # Middlewares HTTP globais
│   ├── auth.go                         # valida JWT, injeta user no contexto
│   ├── rate_limit.go
│   └── cors.go
│
├── websocket/                          # WebSocket
│   └── promotion_hub.go                # hub de conexões e broadcast de eventos
│
├── shared/                             # Código compartilhado entre domínios
│   ├── domain_event.go                 # Interface base para eventos de domínio
│   ├── event_bus.go                    # Interface do barramento de eventos
│   ├── rabbitmq_event_bus.go           # Implementação RabbitMQ
│   ├── pagination.go                   # Struct de paginação reutilizável
│   ├── crypto.go                       # AES-256 para CPF
│   ├── validator.go                    # Validações customizadas (CPF, E.164, etc.)
│   └── logger.go
│
└── email/
    └── templates/                      # Templates HTML de e-mail
        ├── base/
        │   ├── layout.html
        │   ├── header.html
        │   └── footer.html
        ├── 01_confirm.html
        ├── 02_welcome.html
        ├── 03_reset_password.html
        ├── 04_promotion_alert.html
        ├── 05_flash_alert.html
        ├── 06_favorite_expiring.html
        ├── 06b_favorite_flash_expiring.html
        ├── 06b_favorite_flash_low_stock.html
        ├── 06b_favorite_expired.html
        ├── 06b_favorite_flash_ended.html
        ├── 06b_favorite_removed.html
        ├── 06b_favorite_price_dropped.html
        ├── 07_account_banned.html
        ├── 08_promotion_removed.html
        ├── 09_promotion_featured.html
        ├── 10_new_comment.html
        ├── 11_flash_summary.html
        ├── 12_channel_unreachable.html
        └── 13_newsletter.html
```

**Convenção por domínio:**

| Arquivo | Responsabilidade |
|---|---|
| `{domain}.go` | Model / Entidade principal + Value Objects internos |
| `{domain}_dto.go` | Todos os DTOs do domínio (request + response) |
| `{domain}_repository.go` | Interface do repositório (contrato) |
| `{domain}_repository_pg.go` | Implementação PostgreSQL do repositório |
| `{domain}_service.go` | Regras de negócio, orquestração, validações |
| `{domain}_controller.go` | Handlers HTTP: recebe request, chama service, retorna response |
| `events/` | Eventos de domínio publicados no RabbitMQ |

**Regras de dependência:**
- `controller` depende de `service`
- `service` depende de `repository` (interface) e pode publicar eventos via `event_bus`
- `repository_pg` implementa `repository` — é o único arquivo que conhece SQL
- Domínios **não importam** packages de outros domínios diretamente — comunicação sempre via eventos
- `shared/` é o único package importável por todos os domínios

---

### Estrutura Detalhada — `apps/workers`

```
apps/workers/
├── cmd/
│   └── main.go                         # bootstrap dos workers
├── internal/
│   └── collection/                     # reutiliza o BC collection do backend via módulo Go
└── Dockerfile
```

---

### Estrutura Detalhada — `apps/scheduler`

```
apps/scheduler/
├── cmd/
│   └── main.go                         # bootstrap com cron
├── jobs/
│   ├── trend_score_job.go              # a cada 5min: recalcula trend_score
│   ├── flash_check_job.go              # a cada 1min: verifica flash expiradas
│   ├── expiration_check_job.go         # a cada 10min: verifica promoções expiradas
│   ├── newsletter_daily_job.go         # diariamente às 8h50: gera edição diária
│   └── newsletter_weekly_job.go        # segundas às 8h50: gera edição semanal
└── Dockerfile
```

---

### Estrutura Detalhada — `apps/frontend`

```
apps/frontend/
├── src/
│   ├── app/
│   │   ├── core/                       # singletons globais (instanciados uma vez)
│   │   │   ├── auth/
│   │   │   │   ├── auth.service.ts     # login, logout, refresh token
│   │   │   │   └── auth.store.ts       # Signal store: usuário autenticado + tokens
│   │   │   ├── interceptors/
│   │   │   │   ├── auth.interceptor.ts        # injeta Bearer Token
│   │   │   │   └── refresh.interceptor.ts     # renova token ao receber 401
│   │   │   └── guards/
│   │   │       ├── auth.guard.ts
│   │   │       ├── admin.guard.ts
│   │   │       ├── pending-profile.guard.ts
│   │   │       ├── complete-profile.guard.ts
│   │   │       └── confirmed.guard.ts
│   │   │
│   │   ├── shared/                     # componentes, pipes e diretivas reutilizáveis
│   │   │   ├── components/
│   │   │   │   ├── promotion-card/
│   │   │   │   ├── coupon-card/
│   │   │   │   ├── vote-buttons/       # 🚀 Foguete / 🗑️ Lixo
│   │   │   │   ├── flash-badge/
│   │   │   │   ├── flash-countdown/
│   │   │   │   ├── flash-stock-bar/
│   │   │   │   ├── trending-badge/
│   │   │   │   ├── user-avatar/
│   │   │   │   └── notification-bell/
│   │   │   ├── pipes/
│   │   │   │   ├── currency-br.pipe.ts
│   │   │   │   ├── time-ago.pipe.ts
│   │   │   │   └── discount-percent.pipe.ts
│   │   │   └── directives/
│   │   │       └── promotion-click.directive.ts  # intercepta clique + registra via POST /click
│   │   │
│   │   ├── features/                   # módulos lazy loaded por contexto
│   │   │   ├── home/                   # feed principal + trending carousel
│   │   │   ├── promotions/             # detalhe de promoção + comentários
│   │   │   ├── flash/                  # seção ⚡ Relâmpago
│   │   │   ├── coupons/                # listagem de cupons
│   │   │   ├── categories/             # promoções por categoria
│   │   │   ├── auth/                   # login, cadastro, confirm, complete-profile
│   │   │   ├── profile/                # perfil, favoritos, alertas, notificações
│   │   │   ├── newsletter/             # inscrição/gerenciamento da newsletter
│   │   │   └── admin/                  # painel admin + moderação + workers
│   │   │
│   │   ├── layout/
│   │   │   ├── header/
│   │   │   ├── footer/
│   │   │   └── sidebar/
│   │   │
│   │   └── store/                      # Signal stores globais
│   │       ├── promotion.store.ts      # lista de promoções, filtros, paginação
│   │       └── notification.store.ts   # notificações não lidas
│   │
│   ├── environments/
│   │   ├── environment.ts
│   │   └── environment.prod.ts
│   └── styles/                         # tema global SCSS
│
├── angular.json
├── Dockerfile
└── package.json
```

**Padrões Angular utilizados:**
- Standalone Components (sem NgModules)
- Signals para gerenciamento de estado reativo
- Lazy loading por feature
- HTTP Interceptors para JWT (auth + refresh)
- Angular Router com Guards por status de conta
- WebSocket service para atualizações em tempo real

### Fluxo de Dados em Tempo Real

```
Worker coleta promoção
        ↓
Publica evento no RabbitMQ (exchange: promotions)
        ↓
API consome evento → salva no PostgreSQL → invalida cache Redis
        ↓
API emite evento via WebSocket para clientes conectados
        ↓
Angular recebe via WebSocket → atualiza lista de promoções em tela
```

---

## Fontes de Dados

### APIs Oficiais de Afiliados

| Loja | Programa | API Disponível | Tipo | Observações |
|---|---|---|---|---|
| Mercado Livre | Programa de Afiliados ML | Sim (REST) | Afiliados + Promoções | Bearer Token OAuth2 |
| Shopee | Shopee Affiliate Program | Sim (GraphQL) | Afiliados | App ID + Secret, solicitar credenciais |
| Amazon | Amazon Associates (Creators API) | Sim (REST) | Afiliados | PA-API foi depreciado em abril/2026, migrar para Creators API |
| Magazine Luiza | Magalu Afiliados | Parcial | Afiliados | Verificar disponibilidade da API |
| Americanas | Afiliados Americanas | Parcial | Afiliados | Verificar disponibilidade da API |

> **Importante:** A Amazon PA-API foi depreciada em abril de 2026. O projeto deve usar a nova **Creators API** da Amazon Associates.

### Estratégia de Scraping (fallback)

Para lojas sem API pública, utilizar scraping com as seguintes regras:

- **Respeitar robots.txt** de cada site antes de iniciar scraping.
- Usar delays aleatórios entre requisições (mínimo 2s, máximo 10s).
- Rotacionar User-Agent para simular navegador real.
- Usar proxies residenciais para evitar bloqueio de IP.
- **Nunca** realizar scraping em endpoints autenticados sem autorização.
- Lojas alvo para scraping: Kabum, Pichau, Terabyte, Casas Bahia.

### Frequência de Coleta por Fonte

| Fonte | Frequência | Método |
|---|---|---|
| Mercado Livre | A cada 5 minutos | API REST |
| Shopee | A cada 10 minutos | GraphQL API |
| Amazon | A cada 15 minutos | Creators API |
| Kabum | A cada 30 minutos | Scraping |
| Pichau | A cada 30 minutos | Scraping |
| Terabyte | A cada 30 minutos | Scraping |

---

## Módulos do Sistema

### Módulo de Autenticação (`auth`)

**Responsabilidades:**
- Registro de usuário com confirmação de e-mail
- Login com e-mail/senha
- Login OAuth2 (Google)
- Geração e renovação de JWT
- Logout com invalidação de Refresh Token no Redis

### Módulo de Promoções (`promotions`)

**Responsabilidades:**
- CRUD de promoções (criação apenas via workers automáticos)
- Listagem com paginação, filtros e ordenação
- Votação (🚀/🗑️)
- Denúncia
- Arquivamento automático

**Ordenações disponíveis:**
- Mais recentes
- Mais votadas (score)
- Maior desconto (%)
- Prestes a expirar

**Filtros disponíveis:**
- Por categoria
- Por loja
- Por faixa de desconto (%)
- Por faixa de preço

### Módulo de Cupons (`coupons`)

**Responsabilidades:**
- Listagem de cupons ativos
- Validação periódica de cupons
- Report de cupom inválido
- Desativação automática por reports

### Módulo de Usuário (`users`)

**Responsabilidades:**
- Gerenciamento de perfil
- Histórico de promoções vistas
- Lista de favoritos
- Configuração de alertas/notificações
- Dashboard de reputação

### Módulo de Notificações (`notifications`)

**Responsabilidades:**
- Gerenciamento de preferências de notificação por canal e usuário
- Web Push: registro de tokens VAPID, envio via protocolo Web Push
- E-mail: envio via SMTP (SendGrid/Resend/SES), agrupamento por frequência (instant/daily/weekly)
- WhatsApp: envio via Meta Cloud API com templates aprovados, verificação de número
- Telegram: bot integrado via Telegram Bot API, geração de código de verificação, comandos /pausar e /ativar
- Fila de envio assíncrona via RabbitMQ com retry automático (1 tentativa após 5 minutos em caso de falha)
- Controle de limite diário (máx 20 notificações/canal/usuário/dia)
- Log de envio na tabela `notification_logs` com retenção de 30 dias
- Descadastro via token para links de unsubscribe nos e-mails (LGPD)
- Notificações internas para moderadores (fila de moderação)
- Notificações de favoritos: escuta eventos internos (`promotion_expiring`, `flash_expiring`, `flash_low_stock`, `promotion_expired`, `flash_ended`, `promotion_removed`, `promotion_price_updated`) e dispara notificações para todos os usuários que têm aquela promoção nos favoritos e com o evento ativo nas preferências
- Para `favorite_expired`, `favorite_flash_ended` e `favorite_removed`: busca 3 promoções ativas da mesma categoria ordenadas por `trend_score` para incluir como sugestões no e-mail
- Criação automática de `favorite_notification_preferences` com todos os eventos ativos em `browser` e `email` quando o usuário salva o primeiro favorito
- Notificações de publicador: escuta eventos internos (`comment_created`, `promotion_removed`, `promotion_featured`, `trend_top3_reached`, `flash_ended`) e dispara notificações para o autor da promoção conforme suas preferências de publicador
- Agrupamento de comentários: se mais de 3 `publisher_comment` ocorrerem em menos de 10 minutos para a mesma promoção, agrupa em uma única notificação
- Geração de resumo de desempenho para promoções relâmpago encerradas (views, clicks, votos, tempo ativo)
- Criação automática de `publisher_notification_preferences` com defaults ao usuário publicar pela primeira vez

### Módulo de Newsletter (`newsletter`)

**Responsabilidades:**
- Job agendado diariamente às 8h50 para geração da edição diária
- Job agendado toda segunda às 8h50 para geração da edição semanal
- Consulta as 10 promoções com maior `trend_score` do período via banco
- Monta payload JSON com os dados das promoções e envia para a **API da Anthropic (Claude Sonnet)**
- Recebe HTML responsivo gerado pelo Claude, sanitiza e armazena em `newsletter_editions`
- Às 9h dispara envio em massa para todos os inscritos da frequência correspondente via fila RabbitMQ
- Gerencia inscrições, frequências e descadastros
- Rastreia `recipients_count` e atualiza `sent_at` após conclusão do envio

**Prompt base enviado ao Claude para geração da newsletter:**
```
Você é o redator de uma newsletter de promoções chamada "Em Alta".
Gere um e-mail HTML responsivo completo com as seguintes promoções em alta do dia.
O e-mail deve ter: título criativo, parágrafo de introdução envolvente (2-3 linhas),
cards individuais para cada promoção com nome, descrição curta do benefício, preço promocional,
percentual de desconto e botão "Ver oferta", e parágrafo de encerramento.
Use linguagem informal, animada e direta. Retorne apenas o HTML, sem explicações.

Promoções: [JSON com as 10 promoções]
```

### Módulo de Workers / Scrapers (`workers`)

**Responsabilidades:**
- Coleta periódica de promoções por fonte
- Normalização dos dados coletados para o modelo interno
- Deduplicação antes de publicar no RabbitMQ
- Verificação de promoções expiradas
- Log de erros e alertas de falha

### Módulo de Trend Score (`trend`)

**Responsabilidades:**
- Job agendado a cada 5 minutos que recalcula o `trend_score` de todas as promoções ativas
- Job agendado a cada 1 minuto que verifica promoções relâmpago expiradas por prazo ou estoque e atualiza status para `flash_ended`
- Lê contadores das últimas 1h do Redis (views, clicks, votes, saves, shares) por promoção
- Aplica a fórmula de tendência com decaimento por tempo
- Atualiza `trend_score` e `is_trending` na tabela `promotions` em batch
- Calcula trend_score por contexto: global, por loja e por categoria
- Persiste os contadores da hora atual na tabela `promotion_metrics` via upsert (a cada hora)
- Zera os contadores do Redis após persistência na série temporal

### Módulo Admin (`admin`)

**Responsabilidades:**
- Dashboard com métricas (promoções ativas, usuários, fontes de erro)
- Gerenciamento de categorias
- Fila de moderação (promoções e comentários denunciados)
- Gerenciamento de usuários (banir, promover a moderador)
- Configuração das frequências de scraping

---

## Modelo de Dados

### Tabela `users`

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| email | VARCHAR(255) | UNIQUE, NOT NULL | E-mail de acesso |
| password_hash | VARCHAR(255) | NULLABLE | Nulo para usuários OAuth |
| cpf_encrypted | VARCHAR(500) | UNIQUE, NOT NULL | CPF criptografado AES-256. O índice UNIQUE é aplicado sobre o hash do CPF (cpf_hash), não sobre o valor criptografado |
| cpf_hash | VARCHAR(64) | UNIQUE, NOT NULL | SHA-256 do CPF puro, usado para garantia de unicidade sem expor o dado |
| full_name | VARCHAR(255) | NOT NULL | Nome completo (privado) |
| display_name | VARCHAR(100) | NOT NULL | Nome exibido publicamente |
| birth_date | DATE | NOT NULL | Data de nascimento. Idade calculada dinamicamente |
| gender | ENUM | NOT NULL | `masculino`, `feminino`, `nao_binario`, `prefiro_nao_informar` |
| phone | VARCHAR(20) | NOT NULL | Telefone no formato E.164 (ex: +5534999999999) |
| avatar_url | TEXT | NULLABLE | URL do avatar |
| bio | TEXT | NULLABLE | Biografia curta pública |
| role | ENUM | NOT NULL, DEFAULT `user` | `user` (visualiza, vota, comenta, publica promoções), `moderator` (modera comentários/denúncias/promoções), `admin` (acesso total + painel) |
| reputation | INTEGER | NOT NULL, DEFAULT 0 | Score de reputação |
| status | ENUM | NOT NULL, DEFAULT `pending_confirmation` | `pending_confirmation`, `pending_profile`, `active`, `banned` |
| created_at | TIMESTAMP | NOT NULL | Data de criação |
| updated_at | TIMESTAMP | NOT NULL | Data de atualização |

> **Nota sobre CPF:** O CPF nunca é armazenado em texto puro. O fluxo é: CPF recebido → validado → `cpf_hash` = SHA-256(CPF) para checagem de unicidade → `cpf_encrypted` = AES-256(CPF) para recuperação quando necessário pelo pipeline de dados.

### Tabela `user_interests`

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| user_id | UUID | FK → users, NOT NULL | Usuário |
| category_id | UUID | FK → categories, NOT NULL | Categoria de interesse |
| created_at | TIMESTAMP | NOT NULL | Data de registro |

> **Constraint:** UNIQUE(user_id, category_id). Máximo de 10 registros por usuário (validado na camada de serviço).

### Tabela `notification_preferences`

Armazena as preferências de notificação por canal de cada usuário.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| user_id | UUID | FK → users, NOT NULL | Usuário |
| channel | ENUM | NOT NULL | `browser`, `email`, `whatsapp`, `telegram` |
| is_active | BOOLEAN | NOT NULL, DEFAULT false | Canal ativo |
| only_flash | BOOLEAN | NOT NULL, DEFAULT false | Apenas promoções relâmpago |
| min_discount | INTEGER | NOT NULL, DEFAULT 5 | Desconto mínimo para notificar (%) |
| min_price | DECIMAL(10,2) | NULLABLE | Preço mínimo (opcional) |
| max_price | DECIMAL(10,2) | NULLABLE | Preço máximo (opcional) |
| frequency | ENUM | NOT NULL, DEFAULT `instant` | `instant`, `daily_digest`, `weekly_digest` (apenas para e-mail) |
| created_at | TIMESTAMP | NOT NULL | Data de criação |
| updated_at | TIMESTAMP | NOT NULL | Data de atualização |

> **Constraint:** UNIQUE(user_id, channel).

### Tabela `notification_preference_stores`

Lojas de interesse por canal de notificação do usuário.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| notification_preference_id | UUID | FK → notification_preferences, NOT NULL | Preferência de canal |
| store | VARCHAR(100) | NOT NULL | Nome da loja |

> **Constraint:** UNIQUE(notification_preference_id, store).

### Tabela `notification_preference_categories`

Categorias de interesse por canal de notificação do usuário.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| notification_preference_id | UUID | FK → notification_preferences, NOT NULL | Preferência de canal |
| category_id | UUID | FK → categories, NOT NULL | Categoria |

> **Constraint:** UNIQUE(notification_preference_id, category_id).

### Tabela `push_subscriptions`

Tokens de Web Push por dispositivo/navegador.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| user_id | UUID | FK → users, NOT NULL | Usuário |
| endpoint | TEXT | NOT NULL | Endpoint do serviço de push |
| p256dh | TEXT | NOT NULL | Chave pública do cliente |
| auth | TEXT | NOT NULL | Token de autenticação |
| user_agent | VARCHAR(255) | NULLABLE | Navegador/dispositivo identificado |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Token ainda válido |
| created_at | TIMESTAMP | NOT NULL | Data de criação |

### Tabela `telegram_subscriptions`

Conexão entre usuário e chat do Telegram.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| user_id | UUID | FK → users, UNIQUE, NOT NULL | Usuário (1 chat por usuário) |
| chat_id | BIGINT | UNIQUE, NOT NULL | ID do chat no Telegram |
| verification_code | VARCHAR(10) | NULLABLE | Código temporário de verificação |
| is_verified | BOOLEAN | NOT NULL, DEFAULT false | Conexão confirmada |
| is_paused | BOOLEAN | NOT NULL, DEFAULT false | Usuário pausou via /pausar |
| created_at | TIMESTAMP | NOT NULL | Data de criação |

### Tabela `newsletter_subscriptions`

Inscrições na newsletter "Em Alta".

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| user_id | UUID | FK → users, UNIQUE, NOT NULL | Usuário |
| frequency | ENUM | NOT NULL | `daily`, `weekly` |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Inscrito ativo |
| created_at | TIMESTAMP | NOT NULL | Data de criação |
| updated_at | TIMESTAMP | NOT NULL | Data de atualização |

### Tabela `newsletter_editions`

Edições geradas da newsletter com o HTML produzido pelo Claude.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| frequency | ENUM | NOT NULL | `daily`, `weekly` |
| subject | VARCHAR(255) | NOT NULL | Assunto do e-mail gerado pela IA |
| html_content | TEXT | NOT NULL | HTML completo gerado pelo Claude |
| promotion_ids | UUID[] | NOT NULL | IDs das 10 promoções incluídas |
| sent_at | TIMESTAMP | NULLABLE | Quando foi enviada (null = ainda não enviada) |
| recipients_count | INTEGER | NOT NULL, DEFAULT 0 | Quantidade de destinatários |
| created_at | TIMESTAMP | NOT NULL | Data de criação |

### Tabela `favorite_notification_preferences`

Preferências de notificação de eventos de favoritos por usuário e canal.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| user_id | UUID | FK → users, NOT NULL | Usuário |
| channel | ENUM | NOT NULL | `browser`, `email`, `whatsapp`, `telegram` |
| favorite_expiring_soon | BOOLEAN | NOT NULL, DEFAULT true | Promoção prestes a expirar (2h antes) |
| favorite_flash_expiring_soon | BOOLEAN | NOT NULL, DEFAULT true | Relâmpago prestes a encerrar (30min antes) |
| favorite_flash_low_stock | BOOLEAN | NOT NULL, DEFAULT true | Estoque do relâmpago chegando ao fim |
| favorite_expired | BOOLEAN | NOT NULL, DEFAULT true | Promoção expirou |
| favorite_flash_ended | BOOLEAN | NOT NULL, DEFAULT true | Relâmpago encerrado |
| favorite_removed | BOOLEAN | NOT NULL, DEFAULT true | Promoção removida da plataforma |
| favorite_price_dropped | BOOLEAN | NOT NULL, DEFAULT true | Preço baixou ainda mais |
| created_at | TIMESTAMP | NOT NULL | Data de criação |
| updated_at | TIMESTAMP | NOT NULL | Data de atualização |

> **Constraint:** UNIQUE(user_id, channel). Criado automaticamente com todos os eventos ativos em `browser` e `email` quando o usuário salva seu primeiro favorito.

### Tabela `publisher_notification_preferences`

Preferências de notificação exclusivas para publicadores, por grupo de evento e canal.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| user_id | UUID | FK → users, NOT NULL | Usuário publicador |
| event_group | ENUM | NOT NULL | `comments`, `removals`, `achievements`, `flash_summary` |
| channel | ENUM | NOT NULL | `browser`, `email`, `whatsapp`, `telegram` |
| is_active | BOOLEAN | NOT NULL, DEFAULT true | Notificação ativa para este grupo/canal |
| created_at | TIMESTAMP | NOT NULL | Data de criação |
| updated_at | TIMESTAMP | NOT NULL | Data de atualização |

> **Constraint:** UNIQUE(user_id, event_group, channel). Criado automaticamente com todos os grupos ativos no canal `browser` e `email` quando o usuário publica sua primeira promoção.

### Tabela `notification_logs`

Log de envio de notificações por canal.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| user_id | UUID | FK → users, NOT NULL | Usuário |
| promotion_id | UUID | FK → promotions, NULLABLE | Promoção relacionada (null para newsletter) |
| newsletter_edition_id | UUID | FK → newsletter_editions, NULLABLE | Edição da newsletter (null para notificações comuns) |
| channel | ENUM | NOT NULL | `browser`, `email`, `whatsapp`, `telegram`, `newsletter` |
| event_type | VARCHAR(100) | NULLABLE | Tipo do evento (ex: `publisher_comment`, `publisher_flash_ended_expired`) |
| status | ENUM | NOT NULL | `sent`, `failed`, `retrying`, `discarded` |
| error_message | TEXT | NULLABLE | Mensagem de erro em caso de falha |
| sent_at | TIMESTAMP | NULLABLE | Timestamp do envio bem-sucedido |
| created_at | TIMESTAMP | NOT NULL | Data de criação do log |

### Tabela `promotions`

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| title | VARCHAR(255) | NOT NULL | Título da promoção |
| description | TEXT | NULLABLE | Descrição detalhada |
| product_url | TEXT | NOT NULL | URL do produto normalizada (sem UTMs) |
| product_url_hash | VARCHAR(64) | NOT NULL | SHA-256 da URL normalizada, usado para deduplicação |
| image_url | TEXT | NULLABLE | URL da imagem do produto |
| store | VARCHAR(100) | NOT NULL | Nome da loja (ex: `Mercado Livre`, `Kabum`) |
| original_price | DECIMAL(10,2) | NOT NULL | Preço original |
| promo_price | DECIMAL(10,2) | NOT NULL | Preço promocional |
| discount_percent | DECIMAL(5,2) | NOT NULL | Calculado automaticamente pelo sistema |
| coupon_code | VARCHAR(100) | NULLABLE | Cupom associado, se houver |
| category_id | UUID | FK → categories | Categoria da promoção |
| user_id | UUID | NULLABLE, FK → users | Autor da promoção. NULL quando publicada pelo bot |
| source | ENUM | NOT NULL | `api_mercadolivre`, `api_shopee`, `api_amazon`, `scraping_kabum`, `scraping_pichau`, `scraping_terabyte`, `user`, `admin_manual` |
| status | ENUM | NOT NULL, DEFAULT `active` | `active`, `expired`, `archived`, `removed` |
| is_featured | BOOLEAN | NOT NULL, DEFAULT false | Destaque manual definido pelo admin |
| is_trending | BOOLEAN | NOT NULL, DEFAULT false | Destaque automático calculado pelo sistema |
| rocket_votes | INTEGER | NOT NULL, DEFAULT 0 | Votos positivos |
| trash_votes | INTEGER | NOT NULL, DEFAULT 0 | Votos negativos |
| score | INTEGER | NOT NULL, DEFAULT 0 | Calculado: rocket_votes - trash_votes |
| views | INTEGER | NOT NULL, DEFAULT 0 | Total histórico de visualizações |
| clicks | INTEGER | NOT NULL, DEFAULT 0 | Total histórico de cliques no link |
| saves | INTEGER | NOT NULL, DEFAULT 0 | Total histórico de salvamentos em favoritos |
| shares | INTEGER | NOT NULL, DEFAULT 0 | Total histórico de compartilhamentos |
| trend_score | DECIMAL(10,4) | NOT NULL, DEFAULT 0 | Score de tendência calculado a cada 5min |
| is_flash | BOOLEAN | NOT NULL, DEFAULT false | Indica se é promoção relâmpago |
| flash_ends_at | TIMESTAMP | NULLABLE | Data/hora de encerramento do relâmpago (obrigatório se is_flash = true) |
| flash_stock | INTEGER | NULLABLE | Estoque declarado pelo criador (opcional) |
| flash_stock_remaining | INTEGER | NULLABLE | Estoque restante estimado, decrementado a cada clique |
| expires_at | TIMESTAMP | NULLABLE | Data de expiração para promoções comuns com prazo |
| created_at | TIMESTAMP | NOT NULL | Data de criação |
| updated_at | TIMESTAMP | NOT NULL | Data de atualização |

> Quando a promoção é publicada pelo bot, `user_id` é `NULL`. Quando publicada por um usuário, `user_id` referencia o autor.

### Tabela `promotion_metrics`

Armazena a série temporal de métricas por hora, usada pelo pipeline de engenharia de dados e pelo cálculo de trend_score.

| Campo | Tipo | Restrições | Descrição |
|---|---|---|---|
| id | UUID | PK | Chave primária |
| promotion_id | UUID | FK → promotions, NOT NULL | Promoção |
| store | VARCHAR(100) | NOT NULL | Loja da promoção (desnormalizado para queries analíticas) |
| category_id | UUID | FK → categories, NOT NULL | Categoria (desnormalizado) |
| hour | TIMESTAMP | NOT NULL | Hora do registro (truncado para hora cheia) |
| views | INTEGER | NOT NULL, DEFAULT 0 | Visualizações nessa hora |
| clicks | INTEGER | NOT NULL, DEFAULT 0 | Cliques nessa hora |
| rocket_votes | INTEGER | NOT NULL, DEFAULT 0 | Votos 🚀 Foguete nessa hora |
| trash_votes | INTEGER | NOT NULL, DEFAULT 0 | Votos 🗑️ Lixo nessa hora |
| saves | INTEGER | NOT NULL, DEFAULT 0 | Salvamentos nessa hora |
| shares | INTEGER | NOT NULL, DEFAULT 0 | Compartilhamentos nessa hora |

> **Constraint:** UNIQUE(promotion_id, hour). Registros são inseridos via upsert (INSERT ON CONFLICT DO UPDATE). Esta tabela é a principal fonte para o pipeline de engenharia de dados downstream.

### Tabela `coupons`

| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Chave primária |
| code | VARCHAR(100) | Código do cupom |
| store | VARCHAR(100) | Loja |
| discount_type | ENUM | `percent`, `fixed` |
| discount_value | DECIMAL(10,2) | Valor do desconto |
| min_order_value | DECIMAL(10,2) | Valor mínimo de pedido |
| categories | TEXT[] | Categorias aplicáveis |
| is_active | BOOLEAN | Cupom ativo |
| invalid_reports | INTEGER | Qtd de reports de inválido |
| expires_at | TIMESTAMP | Data de expiração |
| created_at | TIMESTAMP | Data de criação |

### Tabela `categories`

| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Chave primária |
| name | VARCHAR(100) | Nome |
| slug | VARCHAR(100) | Slug para URL |
| icon | VARCHAR(50) | Ícone (ex: nome do ícone) |
| color | VARCHAR(7) | Cor hexadecimal |

### Tabela `votes`

| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Chave primária |
| user_id | UUID | FK para users |
| promotion_id | UUID | FK para promotions |
| type | ENUM | `rocket`, `trash` |
| created_at | TIMESTAMP | Data do voto |

> **Constraint:** UNIQUE(user_id, promotion_id)

### Tabela `comments`

| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Chave primária |
| user_id | UUID | FK para users |
| promotion_id | UUID | FK para promotions |
| parent_id | UUID | FK para comments (nullable, respostas) |
| content | TEXT | Texto do comentário |
| reports | INTEGER | Quantidade de denúncias |
| is_deleted | BOOLEAN | Soft delete |
| created_at | TIMESTAMP | Data de criação |

### Tabela `favorites`

| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Chave primária |
| user_id | UUID | FK para users |
| promotion_id | UUID | FK para promotions |
| created_at | TIMESTAMP | Data de criação |

> **Constraint:** UNIQUE(user_id, promotion_id)

### Tabela `alerts`

| Campo | Tipo | Descrição |
|---|---|---|
| id | UUID | Chave primária |
| user_id | UUID | FK para users |
| category_id | UUID | FK para categories |
| min_discount | INTEGER | Desconto mínimo para alertar (%) |
| notify_email | BOOLEAN | Notificar via e-mail |
| notify_push | BOOLEAN | Notificar via push |
| created_at | TIMESTAMP | Data de criação |

---

## APIs do Backend

### Convenções

- Base URL: `/api/v1`
- Autenticação: `Authorization: Bearer <access_token>`
- Respostas paginadas: `{ data: [], meta: { page, limit, total } }`
- Erros: `{ error: { code, message } }`

### Endpoints de Autenticação

```
POST   /api/v1/auth/register          → Etapa 1: cadastro inicial (e-mail, CPF, senha)
GET    /api/v1/auth/confirm/:token    → Etapa 2: confirmação de e-mail
POST   /api/v1/auth/complete-profile  → Etapa 3: completar perfil pós-confirmação
POST   /api/v1/auth/login             → Login e-mail/senha
POST   /api/v1/auth/refresh           → Renovar Access Token
POST   /api/v1/auth/logout            → Logout
GET    /api/v1/auth/google            → Iniciar OAuth2 Google
GET    /api/v1/auth/google/callback   → Callback OAuth2 Google
```

**Body — `POST /api/v1/auth/register` (Etapa 1):**
```json
{
  "email": "lucas@email.com",
  "cpf": "123.456.789-09",
  "password": "Senha123",
  "accept_terms": true,
  "accept_privacy_policy": true
}
```

**Body — `POST /api/v1/auth/complete-profile` (Etapa 3 — requer token JWT da conta `pending_profile`):**
```json
{
  "full_name": "Lucas Silva",
  "display_name": "Lucas",
  "birth_date": "1998-05-20",
  "gender": "masculino",
  "phone": "+5534999999999",
  "category_ids": ["uuid-1", "uuid-2", "uuid-3"]
}
```

> O endpoint `/complete-profile` só aceita tokens de contas com status `pending_profile`. Contas `active` são redirecionadas. Contas `pending_confirmation` recebem erro 403.

### Endpoints de Promoções

```
# Públicos
GET    /api/v1/promotions                    → Listar promoções com filtros e paginação
GET    /api/v1/promotions/:id                → Detalhe de promoção
GET    /api/v1/promotions/:id/comments       → Comentários de uma promoção
POST   /api/v1/promotions/:id/click          → Registrar clique no link (anônimo ou autenticado)
POST   /api/v1/promotions/:id/share          → Registrar compartilhamento (anônimo ou autenticado)
GET    /api/v1/promotions/flash              → Listar apenas promoções relâmpago ativas (ordenadas por flash_ends_at ASC)

# Autenticados (usuários ativos)
POST   /api/v1/promotions                    → Publicar nova promoção
PUT    /api/v1/promotions/:id                → Editar própria promoção (somente autor, somente status pending_review)
DELETE /api/v1/promotions/:id                → Remover própria promoção (somente autor)
POST   /api/v1/promotions/:id/vote           → Votar 🚀 Foguete ou 🗑️ Lixo
DELETE /api/v1/promotions/:id/vote           → Remover voto
POST   /api/v1/promotions/:id/report         → Denunciar promoção inválida/expirada
POST   /api/v1/promotions/:id/comments       → Adicionar comentário
```

**Query params — `GET /api/v1/promotions`:**
```
sort         → trending | top_rated | most_clicked | newest | biggest_discount | expiring_soon | flash_ending_soon
                (default: trending)
store        → mercadolivre | amazon | shopee | kabum | pichau | terabyte | magalu | americanas
category     → slug da categoria (ex: eletronicos, roupas, games)
min_discount → número inteiro 0-100
max_discount → número inteiro 0-100
min_price    → decimal
max_price    → decimal
has_coupon   → true | false
featured     → true | false
trending     → true | false
flash        → true | false (filtra apenas promoções relâmpago ativas)
page         → número inteiro (default: 1)
limit        → número inteiro (default: 20, max: 50)
```

**Exemplo de chamadas:**
```
GET /api/v1/promotions?sort=trending
GET /api/v1/promotions?store=amazon&sort=trending
GET /api/v1/promotions?category=eletronicos&sort=top_rated
GET /api/v1/promotions?store=kabum&sort=most_clicked
GET /api/v1/promotions?category=roupas&sort=biggest_discount
GET /api/v1/promotions?sort=top_rated&min_discount=30&has_coupon=true
```

**Body — `POST /api/v1/promotions` (publicar promoção):**
```json
{
  "title": "Título da promoção",
  "description": "Descrição opcional",
  "product_url": "https://www.loja.com.br/produto",
  "image_url": "https://...",
  "store": "Kabum",
  "original_price": 199.90,
  "promo_price": 99.90,
  "coupon_code": "PROMO10",
  "category_id": "uuid-categoria",
  "expires_at": "2025-12-31T23:59:59Z",
  "is_flash": false,
  "flash_ends_at": null,
  "flash_stock": null
}
```

**Body — promoção relâmpago (is_flash = true):**
```json
{
  "title": "⚡ SSD 1TB Kingston por R$ 199!",
  "description": "Apenas hoje, quantidade limitada!",
  "product_url": "https://www.kabum.com.br/produto/...",
  "image_url": "https://...",
  "store": "Kabum",
  "original_price": 399.90,
  "promo_price": 199.90,
  "category_id": "uuid-eletronicos",
  "is_flash": true,
  "flash_ends_at": "2025-06-15T23:59:59Z",
  "flash_stock": 50
}
```

> `flash_ends_at` deve ser no máximo 24h após o momento da criação, validado pelo backend. `discount_percent` é calculado automaticamente. `source` é definido como `user` automaticamente. Promoções de usuários entram como `pending_review` mesmo sendo relâmpago.

### Endpoints de Cupons

```
GET    /api/v1/coupons                → Listar cupons ativos (público)
GET    /api/v1/coupons/:id            → Detalhe do cupom (público)
POST   /api/v1/coupons/:id/report     → Reportar cupom inválido (autenticado)
```

### Endpoints de Usuário

```
GET    /api/v1/users/me                          → Perfil próprio completo (autenticado)
PUT    /api/v1/users/me                          → Editar perfil (autenticado)
GET    /api/v1/users/me/favorites                → Lista de favoritos (autenticado)
POST   /api/v1/users/me/favorites/:promotionId   → Adicionar favorito
DELETE /api/v1/users/me/favorites/:promotionId   → Remover favorito
GET    /api/v1/users/me/alerts                   → Alertas configurados (autenticado)
POST   /api/v1/users/me/alerts                   → Criar alerta (autenticado)
DELETE /api/v1/users/me/alerts/:id               → Remover alerta (autenticado)
PUT    /api/v1/users/me/interests                → Atualizar interesses/categorias
GET    /api/v1/users/me/export                   → Exportar todos os dados pessoais (LGPD)
DELETE /api/v1/users/me                          → Solicitar exclusão de conta (LGPD)
GET    /api/v1/users/:id                         → Perfil público de outro usuário
```

**Campos editáveis em `PUT /api/v1/users/me`:**
```json
{
  "display_name": "Lucas",
  "avatar_url": "https://...",
  "bio": "Caçador de promoções",
  "gender": "masculino",
  "phone": "+5534999999999"
}
```

> CPF, data de nascimento, e-mail e nome completo **não** são editáveis por este endpoint.

**Body em `PUT /api/v1/users/me/interests`:**
```json
{
  "category_ids": ["uuid-1", "uuid-2"]
}
```

**Response de `GET /api/v1/users/me`** (dados privados retornados apenas para o próprio usuário):
```json
{
  "id": "uuid",
  "email": "lucas@email.com",
  "full_name": "Lucas Silva",
  "display_name": "Lucas",
  "birth_date": "1998-05-20",
  "age": 27,
  "gender": "masculino",
  "phone": "+5534999999999",
  "avatar_url": "https://...",
  "bio": "...",
  "reputation": 120,
  "role": "user",
  "is_confirmed": true,
  "interests": [{ "id": "uuid", "name": "Eletrônicos" }],
  "created_at": "2025-01-01T00:00:00Z"
}
```

> O campo `age` é calculado dinamicamente pelo backend com base em `birth_date`. O CPF **nunca** é retornado em nenhum endpoint.

### Endpoints de Notificações

```
# Preferências por canal (autenticado)
GET    /api/v1/notifications/preferences                        → Listar preferências de todos os canais
PUT    /api/v1/notifications/preferences/:channel               → Atualizar preferências de um canal
DELETE /api/v1/notifications/preferences/:channel               → Desativar canal

# Web Push (autenticado)
POST   /api/v1/notifications/push/subscribe                     → Registrar token de push do navegador
DELETE /api/v1/notifications/push/subscribe                     → Remover token de push

# Telegram (autenticado)
POST   /api/v1/notifications/telegram/generate-code            → Gerar código de verificação para conectar bot
POST   /api/v1/notifications/telegram/verify                   → Confirmar código e vincular chat_id

# WhatsApp (autenticado)
POST   /api/v1/notifications/whatsapp/verify                   → Disparar mensagem de verificação para o número

# Newsletter (autenticado)
GET    /api/v1/newsletter/subscription                         → Status da inscrição do usuário
POST   /api/v1/newsletter/subscribe                            → Inscrever-se na newsletter
PUT    /api/v1/newsletter/subscription                         → Alterar frequência (daily/weekly)
DELETE /api/v1/newsletter/unsubscribe                          → Descadastrar da newsletter

# Webhook do Bot Telegram (sem autenticação — validado via secret token do Telegram)
POST   /api/v1/telegram/webhook                                → Recebe updates do Telegram (mensagens, callbacks, comandos)

# Unsubscribe público (sem autenticação, via link do e-mail)
GET    /api/v1/newsletter/unsubscribe/:token                   → Descadastro via link do e-mail
GET    /api/v1/notifications/unsubscribe/:token                → Descadastro de notificações por e-mail via link

# Notificações de favoritos (autenticado)
GET    /api/v1/notifications/favorites/preferences             → Listar preferências de notificações de favoritos
PUT    /api/v1/notifications/favorites/preferences             → Atualizar preferências de notificações de favoritos

# Notificações de publicador (autenticado)
GET    /api/v1/notifications/publisher/preferences             → Listar preferências de publicador por grupo e canal
PUT    /api/v1/notifications/publisher/preferences             → Atualizar preferências de publicador
```

**Body — `PUT /api/v1/notifications/publisher/preferences`:**
```json
{
  "preferences": [
    { "event_group": "comments",     "channel": "browser",  "is_active": true },
    { "event_group": "comments",     "channel": "email",    "is_active": true },
    { "event_group": "comments",     "channel": "whatsapp", "is_active": false },
    { "event_group": "removals",     "channel": "browser",  "is_active": true },
    { "event_group": "removals",     "channel": "email",    "is_active": true },
    { "event_group": "achievements", "channel": "browser",  "is_active": true },
    { "event_group": "achievements", "channel": "telegram", "is_active": true },
    { "event_group": "flash_summary","channel": "email",    "is_active": true }
  ]
}
```

**Body — `PUT /api/v1/notifications/preferences/:channel`:**
```json
{
  "is_active": true,
  "only_flash": false,
  "min_discount": 20,
  "min_price": null,
  "max_price": 500.00,
  "frequency": "instant",
  "category_ids": ["uuid-eletronicos", "uuid-games"],
  "stores": ["amazon", "kabum"]
}
```

### Endpoints Admin (requer role admin/moderator)

```
# Promoções
GET    /api/v1/admin/promotions                  → Listar todas (incluindo arquivadas/removidas)
POST   /api/v1/admin/promotions                  → Criar promoção manualmente
PUT    /api/v1/admin/promotions/:id              → Editar qualquer promoção
DELETE /api/v1/admin/promotions/:id              → Remover qualquer promoção
PUT    /api/v1/admin/promotions/:id/feature      → Marcar/desmarcar como destaque
PUT    /api/v1/admin/promotions/:id/approve      → Aprovar promoção em pending_review
GET    /api/v1/admin/promotions/queue            → Fila de moderação (pending_review + denunciadas)

# Categorias
GET    /api/v1/admin/categories                  → Listar categorias
POST   /api/v1/admin/categories                  → Criar categoria
PUT    /api/v1/admin/categories/:id              → Editar categoria
DELETE /api/v1/admin/categories/:id              → Remover categoria

# Usuários
GET    /api/v1/admin/users                       → Listar usuários
PUT    /api/v1/admin/users/:id/ban               → Banir usuário
PUT    /api/v1/admin/users/:id/role              → Alterar role do usuário

# Workers / Bot
GET    /api/v1/admin/workers/status              → Status de todos os workers
POST   /api/v1/admin/workers/:name/run           → Forçar execução manual de um worker
PUT    /api/v1/admin/workers/:name/config        → Configurar frequência e parâmetros
GET    /api/v1/admin/workers/logs                → Logs de execução e erros dos workers
```

### WebSocket

```
WS /ws/promotions   → Canal de novas promoções em tempo real
```

**Eventos emitidos pelo servidor:**
```json
{ "event": "new_promotion",         "data": { ...promotion } }
{ "event": "new_flash_promotion",   "data": { ...promotion } }
{ "event": "promotion_updated",     "data": { "id": "...", "score": 42 } }
{ "event": "promotion_expired",     "data": { "id": "..." } }
{ "event": "flash_countdown",       "data": { "id": "...", "seconds_remaining": 3540, "stock_remaining": 38 } }
{ "event": "flash_stock_update",    "data": { "id": "...", "stock_remaining": 12 } }
{ "event": "flash_ended",           "data": { "id": "...", "reason": "expired" | "out_of_stock" } }
{ "event": "trending_updated",      "data": { "promotion_ids": ["uuid1", "uuid2", ...] } }
```

> O evento `flash_countdown` é emitido a cada 60 segundos para promoções relâmpago ativas. Quando faltam menos de 10 minutos, passa a ser emitido a cada 10 segundos.

---

## Frontend Angular

### Rotas

```
/                       → Home (feed de promoções)
/promocoes/:id          → Detalhe da promoção
/cupons                 → Listagem de cupons
/relampago              → Seção exclusiva de promoções relâmpago ativas
/categorias/:slug       → Promoções por categoria
/auth/login                 → Login
/auth/register              → Cadastro (Etapa 1: e-mail, CPF, senha)
/auth/confirm/:token        → Confirmação de e-mail (Etapa 2)
/completar-perfil           → Completar perfil (Etapa 3 — guard: pending_profile)
/aguardando-confirmacao     → Tela informativa pós-cadastro
/perfil                 → Perfil próprio (autenticado)
/perfil/favoritos       → Favoritos (autenticado)
/perfil/alertas         → Alertas (autenticado)
/usuario/:id            → Perfil público
/admin                  → Painel admin (admin/moderator)
/admin/moderacao        → Fila de moderação
/admin/categorias       → Gerenciar categorias
/perfil/notificacoes    → Painel de preferências de notificação (autenticado)
/perfil/notificacoes/publicador → Painel de preferências de notificações de publicador (autenticado, só exibido se usuário já publicou ao menos 1 promoção)
/newsletter             → Página de inscrição/gerenciamento da newsletter
```

### Signals e Estado Global

- `AuthStore` (Signal): usuário autenticado, tokens
- `PromotionStore` (Signal): lista de promoções, filtros, paginação
- `NotificationStore` (Signal): notificações não lidas

### Componentes Principais

| Componente | Descrição |
|---|---|
| `PromotionCardComponent` | Card de promoção com votação e favorito |
| `PromotionListComponent` | Feed com scroll infinito |
| `PromotionDetailComponent` | Página de detalhe com comentários |
| `CouponCardComponent` | Card de cupom com botão de copiar código |
| `FilterBarComponent` | Barra de filtros e ordenação |
| `VoteButtonsComponent` | Botões 🚀/🗑️ com score |
| `CommentSectionComponent` | Seção de comentários com resposta |
| `UserAvatarComponent` | Avatar com reputação |
| `NotificationBellComponent` | Sino de notificações com badge |
| `TrendingBadgeComponent` | Badge "🔥 Em alta" exibido em promoções com is_trending = true |
| `FilterBarComponent` | Barra de filtros: loja, categoria, desconto, preço, cupom, destaque, tendência |
| `SortBarComponent` | Seletor de ordenação: trending, top_rated, most_clicked, newest, biggest_discount, expiring_soon |
| `PromotionClickDirective` | Diretiva que intercepta o clique no botão "Ver oferta", registra o evento via POST /click e redireciona |
| `TrendingCarouselComponent` | Carrossel horizontal das top 10 promoções em tendência no momento |
| `TopRatedSectionComponent` | Seção de promoções mais bem avaliadas de todos os tempos |
| `FlashSectionComponent` | Seção ⚡ Relâmpago no topo do feed com carrossel de flash sales ativas |
| `FlashCountdownComponent` | Contador regressivo em tempo real (dias, horas, minutos, segundos) via WebSocket |
| `FlashStockBarComponent` | Barra de progresso visual do estoque restante |
| `FlashBadgeComponent` | Badge ⚡ exibido em cards de promoções relâmpago |
| `NotificationPreferencesComponent` | Painel completo de configuração de canais e preferências |
| `PushPermissionPromptComponent` | Prompt solicitando permissão de push no navegador |
| `TelegramConnectComponent` | Fluxo de conexão com o bot do Telegram (exibe código + instruções) |
| `WhatsappVerifyComponent` | Fluxo de verificação do número para WhatsApp |
| `NewsletterSubscribeComponent` | Card de inscrição na newsletter com seleção de frequência |
| `NotificationChannelCardComponent` | Card individual de configuração de canal (ativo/inativo + filtros) |
| `PublisherNotificationPreferencesComponent` | Painel de configuração de notificações exclusivas para publicadores (por grupo de evento e canal) |
| `PromotionPerformanceSummaryComponent` | Resumo de desempenho exibido na notificação de encerramento de promoção relâmpago (views, clicks, votos, tempo ativo) |

### Interceptors

- `AuthInterceptor`: adiciona Bearer Token em todas as requisições autenticadas
- `RefreshInterceptor`: ao receber 401, tenta renovar o token automaticamente antes de redirecionar para login

### Guards

- `AuthGuard`: bloqueia rotas que exigem autenticação
- `AdminGuard`: bloqueia rotas de admin para usuários sem a role adequada
- `PendingProfileGuard`: redireciona para `/completar-perfil` se o status da conta for `pending_profile`. Aplicado em todas as rotas protegidas exceto a própria rota de completar perfil.
- `CompleteProfileGuard`: impede que usuários com status `active` acessem a rota `/completar-perfil`

**Fluxo de redirecionamento por status:**
- `pending_confirmation` → redireciona para `/aguardando-confirmacao` (página informativa)
- `pending_profile` → redireciona para `/completar-perfil` (obrigatório)
- `active` → acesso normal à plataforma
- `banned` → redireciona para `/conta-suspensa`

---

## Infraestrutura

### Docker Compose (desenvolvimento local)

```yaml
# docker-compose.yml na raiz do monorepo
services:
  # Apps
  backend:      # apps/backend   — Go API REST + WebSocket (:8080, :8081)
  workers:      # apps/workers   — Go Workers de coleta e scraping
  scheduler:    # apps/scheduler — Go Jobs agendados (trend, newsletter, expiração)
  frontend:     # apps/frontend  — Angular via ng serve (:4200)

  # Infraestrutura
  postgres:     # PostgreSQL (:5432)
  redis:        # Redis (:6379)
  rabbitmq:     # RabbitMQ (:5672) + painel admin (:15672)
  nginx:        # API Gateway / proxy reverso (:80, :443)
```

**Comandos via Makefile:**
```
make dev          → sobe toda a stack local (docker-compose up)
make backend      → sobe apenas o backend + dependências
make frontend     → sobe apenas o frontend
make migrate      → executa migrations pendentes
make migrate-down → reverte última migration
make test         → roda testes de todos os apps
make lint         → lint em todos os apps
```

### Variáveis de Ambiente (`.env`)

```
DATABASE_URL=postgres://user:pass@postgres:5432/promodb
REDIS_URL=redis://redis:6379
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672
JWT_SECRET=...
JWT_REFRESH_SECRET=...
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...

# APIs de Afiliados
ML_APP_ID=...
ML_CLIENT_SECRET=...
SHOPEE_APP_ID=...
SHOPEE_SECRET=...
AMAZON_ACCESS_KEY=...
AMAZON_SECRET_KEY=...
AMAZON_PARTNER_TAG=...

# E-mail
SMTP_PROVIDER=resend          # resend | sendgrid | ses
SMTP_HOST=...
SMTP_PORT=587
SMTP_USER=...
SMTP_PASS=...
EMAIL_FROM=noreply@suaplataforma.com.br
EMAIL_FROM_NAME=Em Alta Promoções

# Web Push (VAPID)
VAPID_PUBLIC_KEY=...
VAPID_PRIVATE_KEY=...
VAPID_SUBJECT=mailto:contato@suaplataforma.com.br

# WhatsApp (Meta Cloud API)
WHATSAPP_API_URL=https://graph.facebook.com/v18.0
WHATSAPP_PHONE_NUMBER_ID=...
WHATSAPP_ACCESS_TOKEN=...
WHATSAPP_TEMPLATE_PROMOTION=promo_alert    # nome do template aprovado
WHATSAPP_TEMPLATE_FLASH=flash_alert

# Telegram
TELEGRAM_BOT_TOKEN=...
TELEGRAM_BOT_USERNAME=@SuaPlataformaBot
TELEGRAM_WEBHOOK_SECRET=...          # token secreto para validar requisições do Telegram
TELEGRAM_BOT_QUERY_RATE_LIMIT=20     # consultas por hora para usuários autenticados
TELEGRAM_BOT_ANON_RATE_LIMIT=10      # consultas por hora para usuários não autenticados
TELEGRAM_BOT_PAGE_SIZE=5             # promoções por página de resultado
TELEGRAM_BOT_MAX_PAGES=3             # máximo de páginas por consulta

# Anthropic (newsletter IA)
ANTHROPIC_API_KEY=...
ANTHROPIC_MODEL=claude-sonnet-4-20250514
NEWSLETTER_MAX_PROMOTIONS=10
```

### Checklist de Implementação

#### Fase 1 — Base
- [ ] Setup Go com Gin + Clean Architecture
- [ ] Setup Angular com Standalone Components e Signals
- [ ] Docker Compose com PostgreSQL, Redis, RabbitMQ
- [ ] Migrations iniciais
- [ ] Módulo de autenticação completo (JWT + Google OAuth2)

#### Fase 2 — Core
- [ ] Worker Mercado Livre (API oficial)
- [ ] Worker Shopee (GraphQL API)
- [ ] Worker Amazon (Creators API)
- [ ] Listagem de promoções com filtros e paginação
- [ ] Sistema de votação
- [ ] Comentários

#### Fase 3 — Engagement
- [ ] Favoritos e alertas por categoria/loja
- [ ] Web Push Notifications (VAPID)
- [ ] Notificações por e-mail (instant + digest diário/semanal)
- [ ] Integração WhatsApp Business API (templates + verificação de número)
- [ ] Bot Telegram (verificação por código + comandos /pausar /ativar)
- [ ] Painel de preferências de notificação por canal
- [ ] WebSocket para atualizações em tempo real
- [ ] Sistema de reputação de usuário

#### Fase 3.4 — Templates de E-mail
- [ ] Estrutura base (layout.html, header.html, footer.html)
- [ ] renderer.go com substituição de variáveis e geração de plain text
- [ ] TEMPLATE-01: Confirmação de cadastro
- [ ] TEMPLATE-02: Boas-vindas
- [ ] TEMPLATE-03: Recuperação de senha
- [ ] TEMPLATE-04: Alerta de promoção
- [ ] TEMPLATE-05: Promoção relâmpago
- [ ] TEMPLATE-06: Favorito prestes a expirar
- [ ] TEMPLATE-06b: Variantes de favorito (flash_expiring_soon, flash_low_stock, expired, flash_ended, removed, price_dropped)
- [ ] TEMPLATE-07: Conta banida
- [ ] TEMPLATE-08: Promoção removida (publicador)
- [ ] TEMPLATE-09: Promoção em destaque (publicador)
- [ ] TEMPLATE-10: Novo comentário (publicador, unitário + agrupado)
- [ ] TEMPLATE-11: Resumo de encerramento relâmpago (publicador)
- [ ] TEMPLATE-12: Canal unreachable
- [ ] TEMPLATE-13: Wrapper da newsletter (header + footer para HTML do Claude)

#### Fase 3.6 — Bot de Consulta Telegram
- [ ] Registro do webhook no Telegram (`POST /setWebhook`)
- [ ] Handler de webhook com validação do secret token
- [ ] Roteamento de comandos: /start, /ajuda, /conectar, /desconectar
- [ ] Comando /hoje (top 10 por trend_score)
- [ ] Comando /relampago (flash sales ativas)
- [ ] Comando /categoria com seleção via teclado inline
- [ ] Comando /loja com seleção via teclado inline
- [ ] Comando /meus (feed personalizado por categorias de interesse)
- [ ] Busca por palavra-chave (mensagem livre)
- [ ] Paginação via callback_query (botão "Ver mais 5 →")
- [ ] Formatador Markdown de promoções
- [ ] Rate limiting por chat_id no Redis
- [ ] Sessão de contexto de busca no Redis (para paginação)
- [ ] Vinculação de conta via código de verificação (/conectar)

#### Fase 3.5 — Newsletter IA
- [ ] Job de geração de newsletter (diária e semanal)
- [ ] Integração com API do Claude (Anthropic) para geração do HTML
- [ ] Sanitização do HTML gerado
- [ ] Envio em massa via fila RabbitMQ
- [ ] Página de inscrição/gerenciamento da newsletter
- [ ] Links de unsubscribe com token (LGPD)

#### Fase 4 — Admin & Scrapers
- [ ] Painel admin com fila de moderação
- [ ] Workers de scraping (Kabum, Pichau, etc.)
- [ ] Dashboard de métricas
- [ ] Verificação periódica de promoções expiradas

#### Fase 5 — Polimento
- [ ] Testes unitários e de integração (Go)
- [ ] Testes de componente (Angular)
- [ ] Rate limiting na API
- [ ] Monitoramento (logs estruturados + alertas)
- [ ] CI/CD pipeline