# ChatSales

Plataforma de automação de vendas e atendimento integrada com n8n, Supabase e Chatwoot.

## Stack

- **n8n** — Automações e fluxos de atendimento
- **Supabase** — Banco de dados, autenticação e funções edge
- **Chatwoot** — CRM e central de atendimento multicanal

## Estrutura

```
chatsales/
├── n8n/
│   └── workflows/       # Workflows exportados do n8n (.json)
├── supabase/
│   ├── migrations/      # Migrations SQL
│   └── functions/       # Edge Functions
├── chatwoot/            # Configurações e webhooks do Chatwoot
└── docs/                # Documentação
```

## Setup

### Pré-requisitos

- n8n (self-hosted ou cloud)
- Supabase projeto configurado
- Chatwoot instância ativa

### Variáveis de ambiente

Copie `.env.example` para `.env` e preencha:

```bash
cp .env.example .env
```
