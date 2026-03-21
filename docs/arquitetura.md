# Arquitetura ChatSales

## Fluxo principal

```
WhatsApp / Instagram / Web Chat
        ↓
    Chatwoot (recebe mensagem)
        ↓
    Webhook → n8n (processa)
        ↓
    Supabase (armazena / consulta)
        ↓
    n8n (resposta via API Chatwoot)
        ↓
    Cliente
```

## Integrações planejadas

- [ ] Webhook Chatwoot → n8n
- [ ] Roteamento de conversas por segmento
- [ ] Qualificação de leads com IA
- [ ] Registro automático no Supabase (CRM)
- [ ] Notificações e alertas de vendas
