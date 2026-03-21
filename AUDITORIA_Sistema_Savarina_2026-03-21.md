# Auditoria do Sistema Savarina — ChatSales
**Data:** 2026-03-21
**Executada por:** Claude (Sonnet 4.6)
**Status:** COMPLETA

---

## 1. Visão Geral

O Sistema Savarina é o produto principal da ChatSales: um agente de atendimento receptivo via WhatsApp com IA (GPT-4), que gerencia conversas, agenda consultas no Google Calendar, sincroniza com Chatwoot (CRM) e executa cadências de follow-up automatizadas.

---

## 2. Schema Real do Supabase

### 2.1 Tabelas Confirmadas

| Tabela | Registros | Status |
|--------|-----------|--------|
| `contacts` | 4 | OK |
| `conversations` | 4 | OK |
| `messages` | 157 | OK |
| `appointments` | 1 | OK - ver alerta |
| `ai_pauses` | 2 | OK (nome diferente do planejado) |
| `followup_logs` | 0 | VAZIA - nenhum log registrado |

### 2.2 Tabelas NÃO Existentes (referenciadas no plano, mas ausentes)

| Tabela Esperada | Status | Impacto |
|-----------------|--------|---------|
| `ai_locks` | NAO EXISTE | Confusão no plano — a tabela real é `ai_pauses` |
| `response_blocks` | NAO EXISTE | Feature de pausa manual não implementada via esta tabela |
| `paused_conversations` | NAO EXISTE | Idem |

### 2.3 Schema Detalhado por Tabela

#### `contacts`
```
id (uuid), chatwoot_id (int), name (text), identifier (text),
created_at (timestamptz), phone_number (text)
```

#### `conversations`
```
id (uuid), chatwoot_conversation_id (bigint), contact_id (uuid),
status (text), account_id (int), updated_at (timestamptz),
chatwoot_contact_id (int), labels (text[]), last_incoming_at (timestamptz),
last_outgoing_at (timestamptz), last_outgoing_by (text),
appointment_status (text), followup_cadence (text), last_followup_at (timestamptz)
```

#### `messages`
```
id (uuid), chatwoot_message_id (int), conversation_id (uuid),
content (text), content_type (text), sender_type (text),
created_at (timestamptz), from_who (text), chatwoot_conversation_id (bigint),
source_id (text)
```
> **Nota:** `sender_type` usa valores do Chatwoot (`Contact`, `User`).
> `from_who` usa valores do sistema (`lead`, `ai`, `human`).

#### `appointments`
```
id (uuid), conversation_id (uuid), contact_id (uuid),
google_event_id (text), title (text), start_at (timestamptz), end_at (timestamptz),
modality (text), status (text), confirmation_sent_at (timestamptz),
reminder_sent_at (timestamptz), confirmation_response (text),
created_at (timestamptz), updated_at (timestamptz), meet_link (text)
```

#### `ai_pauses`
```
conversation_id (uuid), paused_until (timestamptz), paused_reason (text),
paused_by (text), updated_at (timestamptz)
```
> **Nota:** Esta é a tabela de lock da IA, **não** `ai_locks` como documentado no plano.
> Funciona como: ao receber mensagem humana, insere/atualiza registro com `paused_until = NOW() + 10min`.
> A IA verifica se `paused_until > NOW()` antes de responder.

#### `followup_logs`
> Tabela existe mas está **completamente vazia**. Nenhum follow-up foi logado.

---

## 3. Estado das Conversas

| ID (parcial) | Status | followup_cadence | appointment_status | last_incoming |
|---|---|---|---|---|
| 393795d3 | open | **null** | null | 2026-03-21 |
| 13eab013 | open | **null** | null | 2026-03-17 |
| 33900f49 | pending | pausado | agendado | 2026-03-21 |
| 095b8ed4 | pending | **null** | null | 2026-03-11 |

**Distribuição de followup_cadence:**
- null: 3 conversas (75%) — nunca entraram em nenhuma cadência
- "pausado": 1 conversa (valor não-padrão, não previsto na state machine)
- nivel_1: 0 / nivel_2: 0 / confirmation: 0 / perdido: 0

**Distribuição de mensagens por conversa:**
- 393795d3: 64 mensagens
- 33900f49: 52 mensagens
- 13eab013: 38 mensagens
- 095b8ed4: 3 mensagens

**Distribuição from_who:**
- lead: 123 mensagens
- ai: 26 mensagens
- human: 8 mensagens

---

## 4. Estado dos Agendamentos

| Campo | Valor |
|---|---|
| ID | 99f58e70... |
| status | `scheduled` |
| start_at | 2026-03-09T14:00:00+00:00 |
| google_event_id | c0t2gk9otsu4nohs24tth7bb0g |
| confirmation_sent_at | 2026-03-07T22:50 (enviado) |
| reminder_sent_at | **null** (nao enviado) |

**ALERTA:** Este agendamento ocorreu **12 dias atras** (09/03) e ainda esta com `status=scheduled`. Deveria ter sido marcado como `no_show` pelo workflow B.5, mas B.5 esta INATIVO.

---

## 5. Estado dos AI Pauses

| Conversa | paused_until | Status |
|---|---|---|
| 33900f49 | 2026-03-07T23:25 | EXPIRADO ha 13.7 dias |
| 13eab013 | 2026-03-17T09:42 | EXPIRADO ha 4.2 dias |

Dois registros de pausa expirados. Nao ha limpeza automatica — acumulam indefinidamente.

---

## 6. Auditoria de Workflows n8n

### 6.1 Estado Atual

| Workflow | ID | Status | Funcao |
|---|---|---|---|
| [ChatSales] Fluxo Principal Receptivo | mH3da2kUkwQgEgBd | ATIVO | Nucleo do sistema |
| [ChatSales] Confirmacao de Agendamento -24h | klcRRPGxJlXRCCjk | ATIVO | CRON 8-17h (60min) |
| [ChatSales] Follow-up: Nivel 1 | 0Xz2szGXdNUiqhvI | inativo | B.3 |
| [ChatSales] Follow-up: Nivel 2 | AJO4ddV2GGyJKLNv | inativo | B.4 |
| [ChatSales] Follow-up: Confirmacao D-1 | EfFe9jkBLkbspyQ1 | inativo | B.1 |
| [ChatSales] Follow-up: Lembrete H-2 | GUb1x9tyfkvJjrmm | inativo | B.2 |
| [ChatSales] Follow-up: Gestao de No-show | QaAhCf1zBYgSgBQP | inativo | B.5 |
| [ChatSales] Agente Sincronizador Chatwoot | yDaQHb0kKuhSFXuX | inativo | C.2 (deprecar) |
| [ChatSales] Logger Central | ItkbnDLSJtgmRXeo | inativo | C.1 (deprecar) |
| Calendar MCP | 9jHdRyn24md-vSzgAwSrf | inativo | Standalone (fora escopo) |
| Agente Calendar | ToGDVDe0plU54hLR | inativo | Standalone (fora escopo) |

### 6.2 Execucoes com Erro

**Confirmacao -24h (klcRRPGxJlXRCCjk):**
- Taxa de erro: **100%** (todas execucoes desde pelo menos 19/03)
- Frequencia: 2x por hora (duas instancias do trigger — bug de configuracao?)
- **Causa raiz:** Google OAuth2 refresh token expirado/revogado
- Node com erro: "Busca Eventos nas proximas 24h1" (Google Calendar - getAll)
- Mensagem: `The provided authorization grant... or refresh token is invalid, expired, revoked`

**Fluxo Principal (mH3da2kUkwQgEgBd):**
- Taxa de erro: ~85% nos ultimos 2 dias (maioria falhando entre 19-20/03)
- Hoje (21/03 14:20): 4 execucoes com SUCESSO
- **Causa raiz:** Bug no node "Atualiza Status no Supabase Open"
- Mensagem: `invalid input syntax for type bigint: "393795d3-bd62-45ea-a8f9-adad4c53e20f"`
- **Detalhe do bug:** O filtro usa `chatwoot_conversation_id = $('Merge Conversas').item.json.id`
  - `.id` retorna UUID (ex: `393795d3-...`)
  - `chatwoot_conversation_id` espera bigint (ex: `42`)
  - **Fix:** Mudar para `$('Merge Conversas').item.json.chatwoot_conversation_id`

---

## 7. Gaps e Bugs — Lista Priorizada

### CRITICOS (sistema quebrado)

| # | Gap | Causa | Impacto |
|---|-----|-------|---------|
| **G-CRIT-1** | Confirmacao -24h falhando 100% | OAuth2 Google expirado | Nenhum lembrete de agendamento enviado |
| **G-CRIT-2** | Fluxo Principal com bug no Supabase | `.id` ao inves de `.chatwoot_conversation_id` | Erros em ~85% das mensagens recebidas |
| **G-CRIT-3** | followup_cadence nunca inicializado | Fluxo principal nao seta cadencia | 0 leads entram em follow-up automatico |
| **G-CRIT-4** | Appointment ha 12 dias com status=scheduled | B.5 INATIVO | Dado inconsistente, no-show nao registrado |

### ALTOS (funcionalidade quebrada)

| # | Gap | Causa | Impacto |
|---|-----|-------|---------|
| **G-ALTO-1** | followup_logs vazio | Nenhum B.x ativo; Logger orfao | Zero auditoria de follow-ups |
| **G-ALTO-2** | Todos os B.x inativos | Intencional ou bug? | Nenhum follow-up sendo enviado |
| **G-ALTO-3** | followup_cadence='pausado' nao previsto | Valor nao-standard inserido manualmente? | State machine corrompida |
| **G-ALTO-4** | Confirmacao -24h roda 2x/hora | Trigger duplicado no CRON | Duplicacao de lembretes (quando funcionar) |

### MEDIOS (logica inconsistente)

| # | Gap | Causa | Impacto |
|---|-----|-------|---------|
| **G-MED-1** | ai_pauses expiradas acumulando | Sem limpeza automatica | Tabela cresce indefinidamente |
| **G-MED-2** | reminder_sent_at=null no appointment existente | H-2 INATIVO | Lembrete nao enviado |
| **G-MED-3** | Tabela documentada como `ai_locks` e real e `ai_pauses` | Divergencia docs/codigo | Confusao de manutencao |
| **G-MED-4** | C.2 (Sincronizador) ainda existe como workflow | Logica deveria estar no Fluxo Principal | Confusao arquitetural |

### INVESTIGAR

| # | Item | Pergunta |
|---|------|----------|
| **I-1** | Por que Fluxo Principal funcionou em 21/03 14:20 (4x success)? | Qual condicao mudou? Qual caminho do fluxo foi atingido? |
| **I-2** | Valor 'pausado' no followup_cadence | Quem inseriu? Ha um path no workflow que seta este valor? |
| **I-3** | CRON duplicado na Confirmacao -24h | Dois Schedule Triggers? Ou n8n duplicando trigger? |
| **I-4** | Redis TTL para Chat Memory | Qual tempo configurado? Contexto expirado entre conversas? |

---

## 8. Arquitetura Real do Fluxo Principal

### Ciclo de Vida de uma Mensagem

```
1. Webhook (Chatwoot) → recebe evento message_created
2. Filtra: apenas message_created + nao outgoing
3. Autentica Contato (GET contacts by phone OR CREATE)
4. Autentica Conversa (GET conversations by chatwoot_id OR CREATE)
   [Merge Conversas — junta GET e CREATE em um item]
5. Processa Midia:
   - Audio → Whisper (com fallback Gemini)
   - Imagem → GPT-4 Vision
   - Texto → passthrough
6. INSERT messages (from_who=lead)
7. Verifica AI Pause (ai_pauses WHERE paused_until > NOW())
   → Se pausado: STOP (nao responde)
8. Busca contexto:
   - Ultimas N mensagens da conversa
   - Status da conversa
   - Agendamentos ativos
9. Invoca AI Agent (GPT-4, ReAct, Redis Memory)
   - Tools: Calendar Manager, Supabase queries, etc.
10. Roteia por action:
    - schedule → cria/atualiza Google Calendar + INSERT appointments
    - confirm  → UPDATE appointments.status='confirmed'
    - pause    → INSERT ai_pauses (pausa humana)
    - none     → apenas responde
11. Envia resposta via Evolution API (WhatsApp)
12. INSERT messages (from_who=ai)
13. UPDATE conversations (last_outgoing_at, status)
14. UPDATE Chatwoot (labels, status)
```

### Nodes com Bug Identificado

**"Atualiza Status no Supabase Open":**
```
ERRADO:  chatwoot_conversation_id = $('Merge Conversas').item.json.id
CORRETO: chatwoot_conversation_id = $('Merge Conversas').item.json.chatwoot_conversation_id
```

---

## 9. State Machine: followup_cadence

### Estado Atual vs Esperado

```
Estado esperado:
null → nivel_1 → nivel_2 → confirmation → perdido | confirmado

Estado real (todas as conversas):
null (3 conversas) + "pausado" (1 conversa, nao-padrao)
```

### Transicoes Necessarias (a implementar no Fluxo Principal)

| Evento | De | Para |
|--------|-----|------|
| Nova mensagem de lead (sem agenda) | null | nivel_1 |
| Lead responde (estava em nivel_1) | nivel_1 | nivel_2 |
| Appointment criado | nivel_2 / qualquer | confirmation |
| Lead confirma ("sim") | confirmation | confirmado |
| D+7 sem resposta (B.3) | nivel_1 | perdido |
| D+8 sem agendar (B.4) | nivel_2 | perdido |

---

## 10. Plano de Correcoes Imediatas

### Prioridade 1 — Fazer hoje

1. **Renovar OAuth2 Google Calendar** no n8n (credencial da Confirmacao -24h)
   - Ir em Settings > Credentials > Google Calendar OAuth2
   - Reconectar (re-autenticar)

2. **Corrigir bug no Fluxo Principal** — node "Atualiza Status no Supabase Open"
   - Mudar `$('Merge Conversas').item.json.id` para `$('Merge Conversas').item.json.chatwoot_conversation_id`

3. **Corrigir trigger duplicado** na Confirmacao -24h
   - Verificar se ha dois Schedule Triggers ou configuracao incorreta

4. **Marcar appointment como no_show manualmente** (ID: 99f58e70, start: 09/03)
   - UPDATE appointments SET status='no_show' WHERE id='99f58e70...'

### Prioridade 2 — Proxima semana

5. **Adicionar logica de followup_cadence no Fluxo Principal:**
   - Ao receber primeira mensagem: UPDATE followup_cadence='nivel_1' (se null)
   - Ao criar appointment: UPDATE followup_cadence='confirmation'
   - Ao receber confirmacao: UPDATE appointment.status='confirmed'

6. **Ativar B.5 (No-show)** com filtros corretos do Supabase

7. **Criar rotina de limpeza de ai_pauses expirados** (DELETE WHERE paused_until < NOW())

8. **Padronizar valor 'pausado'** — verificar o path que seta esse valor e corrigir para um valor da state machine

### Prioridade 3 — Medio prazo

9. **Refatorar B.3 (Nivel 1)** com logica correta de CRON + Supabase como fonte
10. **Refatorar B.4 (Nivel 2)** idem
11. **Ativar B.1 (Confirmacao D-1)** e unificar com -24h (escolher um como fonte da verdade)
12. **Deprecar C.2 (Sincronizador)** — absorver no Fluxo Principal
13. **Deprecar C.1 (Logger)** — escrever direto em followup_logs nos B.x

---

## 11. Runbook Operacional

### Como renovar credencial Google Calendar no n8n
1. Acessar `https://chatsales-n8n.yvssrw.easypanel.host`
2. Menu lateral: Settings > Credentials
3. Buscar "Google Calendar" (ou "Google OAuth2")
4. Clicar na credencial > Sign in with Google
5. Completar fluxo OAuth2
6. Verificar execucao manual do workflow Confirmacao -24h

### Como pausar uma conversa manualmente
```sql
-- Via Supabase REST ou SQL:
INSERT INTO ai_pauses (conversation_id, paused_until, paused_reason, paused_by)
VALUES ('<uuid>', NOW() + INTERVAL '24 hours', 'manual', 'admin')
ON CONFLICT (conversation_id) DO UPDATE SET paused_until=EXCLUDED.paused_until;
```

### Como resetar um AI Pause travado
```sql
DELETE FROM ai_pauses WHERE conversation_id = '<uuid>';
-- OU: atualizar para data passada
UPDATE ai_pauses SET paused_until = NOW() - INTERVAL '1 second' WHERE conversation_id = '<uuid>';
```

### Como marcar appointment como no-show manualmente
```sql
-- Via Supabase REST API (POST /rest/v1/appointments com Prefer: resolution=merge-duplicates):
PATCH /rest/v1/appointments?id=eq.<uuid>
Body: {"status": "no_show"}

-- E atualizar a conversa:
PATCH /rest/v1/conversations?id=eq.<conversation_uuid>
Body: {"appointment_status": "no_show"}
```

### Como verificar saude do sistema rapidamente
1. **n8n:** Checar execucoes com erro em `/executions?status=error&limit=10`
2. **Supabase conversations:** Checar followup_cadence != null para ver progresso
3. **Supabase appointments:** Checar `status=scheduled AND start_at < NOW()` (no-shows perdidos)
4. **ai_pauses:** Checar registros com `paused_until > NOW()` (pausas ativas)

### O que fazer quando Evolution API falha
1. Checar se instancia "Mana Garrincha" esta conectada em `https://evo.chatsales.com.br` (ou similar)
2. Reconectar instancia WhatsApp se necessario (QR Code)
3. Verificar execucoes com erro no Fluxo Principal — erro sera no node de envio

---

## 12. Verificacao (Definition of Done)

- [x] Schema completo das tabelas Supabase documentado
- [x] Fluxo principal mapeado com nodes e decisoes
- [x] Lista de issues/bugs priorizados
- [x] Runbook operacional basico escrito
- [ ] Cadencias de follow-up documentadas com timeline (pendente ativacao dos B.x)
- [ ] Diagrama de arquitetura gerado (pendente — recomendado fazer no draw.io)

---

*Auditoria gerada automaticamente por Claude Sonnet 4.6 em 2026-03-21*
