# Automação n8n — Instagram: Auto-resposta Comentário + Direct (DM)

## Visão geral
Este workflow recebe eventos do Instagram via **Webhook**, normaliza o payload, identifica se a solicitação é **Comentário** ou **Direct (DM)** e executa uma resposta automática com:
- **classificação por palavra-chave** (template / ia / dm / default)
- **anti-repetição** de copy (memória de última variação)
- **delay aleatório** para simular comportamento humano
- **disparo via HTTP** para endpoints do Graph API (responder comentário e/ou enviar DM)

> Objetivo de negócio: reduzir tempo de resposta, entregar materiais automaticamente e manter consistência de atendimento sem vibe de bot.

---

## Arquitetura (macrofluxo)
1) **Webhook1** recebe evento  
2) **Edit Fields1** normaliza dados (campos obrigatórios)  
3) **Switch** roteia: `comment` vs `direct`  
4) Ramificação **Comentário**:
   - Filter → Switch de tags → Code (gera reply/tag/delay) → HTTP responde comentário → HTTP envia DM (quando aplicável)
5) Ramificação **Direct**:
   - Switch de tags → HTTP envia DM (mensagem por palavra-chave / fallback)

---

## Inventário de nós (o que cada parte faz)

### Bloco: Entrada + Tratamento + Roteamento (roxo)
- **Webhook1**
  - Função: ponto de entrada (POST)
  - Entrada esperada: evento do Instagram (comentário ou DM)
- **Edit Fields1** (manual)
  - Função: padroniza/renomeia campos e garante que o fluxo trabalhe com um “contrato” fixo
  - Saída sugerida (contrato mínimo):
    - `type`: `"comment"` ou `"direct"`
    - `comment`: texto (se for comment)
    - `dm_text`: texto (se for direct)
    - `comment_id` / `message_id`
    - `user_id` (remetente)
    - `ig_account_id` / `page_id` (se aplicável)
- **Switch** (mode: Rules)
  - Função: roteia para:
    - `comment` → bloco “Comentário”
    - `direct` → bloco “HTTP / Direct”

---

### Bloco: Comentário (verde)
- **Filter**
  - Função: filtra eventos inválidos/indesejados (ex.: payload sem texto, eventos duplicados, etc.)
  - Recomendação: incluir checagens tipo:
    - comentário vazio
    - evento sem `comment_id`
    - ignorar comentários do próprio perfil (anti-loop)
- **Comentário** (Switch / Rules)
  - Função: classifica o conteúdo do comentário em tags:
    - `template`
    - `ia`
    - `dm`
    - (fallback) “Não tem palavra-chave” / default
- **Code**
  - Função: gera o objeto final `{ reply, tag, delay_ms, __last_variant }`
  - Observação: este nó não envia mensagem; ele só prepara o “pacote” de resposta.
- **responde comentario** (HTTP Request)
  - Função: envia a resposta pública no comentário (Graph API)
- **Envia mensagens direct template** (HTTP Request)
  - Função: envia DM para entrega (ex.: template/link), quando a regra exigir

---

### Bloco: HTTP / Direct (vermelho)
- **Direct01** (Switch / Rules)
  - Função: classifica mensagens recebidas por DM:
    - `template`, `ia`, `dm` e fallback
- **Envia mensagem direct palavra chave** (HTTP Request)
  - Função: responde DM com mensagem específica para keyword
- **Envia mensagens direct** (HTTP Request)
  - Função: fallback/variante extra de resposta em DM (ex.: mensagem padrão, entrega, instrução)

---

## Contrato de dados (padrão recomendado)
### Entrada (Webhook → Edit Fields1)
Campos mínimos para o fluxo operar com previsibilidade:
- `type`: `"comment"` | `"direct"`
- `text`: string do conteúdo (normalizado)
- `user_id`: identificador do remetente
- `comment_id` (se `type=comment`)
- `message_id` (se `type=direct`)

> Dica: padroniza tudo em `text` e guarda os originais (`comment`/`dm_text`). Isso simplifica filtros e switches.

### Saída do nó **Code**
Retorna um array com um item:
- `reply`: string (texto final)
- `tag`: `"template" | "ia" | "dm" | "default"`
- `delay_ms`: number (milissegundos)
- `__last_variant`: `{ tag, index }` (memória anti-repetição)

---

## Onde alterar URLs e mensagens (pontos de configuração)
**Atenção:** toda alteração de endpoint/mensagem deve acontecer nos nós HTTP:
- `responde comentario`
- `Envia mensagens direct template`
- `Envia mensagem direct palavra chave`
- `Envia mensagens direct`

Recomendação de governança:
- Centralizar textos em um nó “Set/Config” (ex.: `CONFIG_MESSAGES`)
- Usar variáveis/credentials do n8n para tokens e IDs (nunca hardcode)

---

## Script do nó Code (documentação resumida)
### O que ele faz
- Normaliza o texto recebido
- Detecta `tag` por palavra-chave (prioridade: template > ia > dm)
- Escolhe uma variação de resposta evitando repetir a última
- Gera `delay_ms` aleatório (12–75s)
- Retorna no formato padrão do n8n

### Limitações (transparência, sem caô)
- Não envia DM sozinho (isso é responsabilidade dos nós HTTP)
- Não “adivinha” contexto; só classifica por texto e aplica regras
- Anti-repetição depende de `__last_variant` ser propagado no fluxo

---

## Testes (checklist)
1) Simular evento `comment` com:
   - texto = "template"
   - texto = "quero template e ia" (deve priorizar `template`)
   - texto sem keyword (cai em `default`)
2) Simular evento `direct` com:
   - texto = "ia"
   - texto = "dm"
3) Validar:
   - resposta não repete variação quando `__last_variant` está presente
   - delay_ms dentro do range esperado
   - endpoints respondendo 200/201

---

## Segurança & LGPD
- Tokens e IDs sempre via **Credentials** / **ENV**
- Nunca logar payload com dados sensíveis em texto puro
- Amostras de payload devem ser mockadas (dados fake)

---

## Evoluções recomendadas (roadmap)
- Adicionar deduplicação por `comment_id/message_id` (anti reprocessamento)
- Adicionar Error Workflow / alertas (Slack/Discord) quando HTTP falhar
- Adicionar rate limit/backoff para evitar bloqueio de API
