# Guia de implementação: CRM com WhatsApp Business Platform (Meta)

## Objetivo de negócio
Construir um fluxo que:
1. receba mensagens de leads pelo WhatsApp Business;
2. identifique automaticamente **origem (landing page/campanha)** e **nicho** a partir da frase inicial;
3. salve os dados em um CRM/warehouse;
4. disponibilize um dashboard para medir desempenho por nicho e campanha.

---

## Arquitetura recomendada (MVP)

```text
WhatsApp (Meta Cloud API)
   -> Webhook (API própria)
   -> Processador de mensagem inicial (regras + NLP)
   -> Banco transacional (leads, mensagens, sessões)
   -> Camada analítica (fatos e dimensões)
   -> Dashboard (Looker Studio, Metabase, Power BI)
```

### Componentes principais
- **Meta App + WhatsApp Cloud API**: recebe eventos de mensagens.
- **Webhook backend**: endpoint HTTPS para validação e recepção dos eventos.
- **Serviço de classificação**:
  - fase 1: regras por palavras-chave;
  - fase 2: modelo NLP/LLM para mensagens ambíguas.
- **Banco operacional** (PostgreSQL): rastreia mensagens e leads.
- **Banco analítico** (pode ser o mesmo no começo): tabelas para métricas.
- **Dashboard**: KPIs por nicho, campanha e origem.

---

## Modelagem mínima de dados

### Tabela `lead`
- `id`
- `wa_id` (telefone em formato WhatsApp)
- `first_message_at`
- `first_message_text`
- `origin_lp` (landing page identificada)
- `niche`
- `campaign`
- `confidence` (0-1)
- `classification_method` (`rule`, `nlp`, `manual`)
- `created_at`, `updated_at`

### Tabela `message`
- `id`
- `lead_id`
- `meta_message_id`
- `direction` (`inbound`/`outbound`)
- `text`
- `timestamp`
- `raw_payload` (JSON)

### Tabela `campaign_dim` (dimensão)
- `campaign_id`
- `utm_source`
- `utm_medium`
- `utm_campaign`
- `landing_page`
- `niche`

### Tabela `lead_fact` (fato analítico)
- `date`
- `campaign_id`
- `niche`
- `leads_total`
- `qualified_total`
- `meetings_total`
- `sales_total`
- `revenue_total`

---

## Como identificar origem e nicho da frase inicial

## 1) Estratégia prioritária (mais confiável)
Antes da conversa começar, inclua no link do WhatsApp parâmetros no texto pré-preenchido:

- Exemplo de link com contexto:
  - `https://wa.me/<numero>?text=LP:lp_financas|NICHO:contabilidade|CMP:meta_fev24`

No backend, extraia esse padrão da **primeira mensagem** e classifique com confiança alta.

Vantagem: evita inferência frágil por texto livre.

## 2) Estratégia por regras (keyword mapping)
Se o lead escreve texto livre:
- Normalize (lowercase, sem acento, sem pontuação).
- Aplique dicionário de termos por nicho/origem.
- Exemplo:
  - "abrir cnpj", "mei", "contador" -> nicho `contabilidade`
  - "tratamento capilar", "implante" -> nicho `estetica`

## 3) Fallback com NLP/LLM
Quando regra não atingir score mínimo:
- envie texto para classificador;
- retorne `{niche, origin_lp_guess, confidence}`;
- registre para auditoria e melhoria contínua.

## 4) Fila de revisão manual
Leads com `confidence < 0.65` devem ir para fila de revisão para corrigir classificação.

---

## Passos na API da Meta (Cloud API)

1. Criar app no **Meta for Developers**.
2. Adicionar produto **WhatsApp** no app.
3. Obter:
   - `WHATSAPP_PHONE_NUMBER_ID`
   - `WHATSAPP_BUSINESS_ACCOUNT_ID`
   - token de acesso (idealmente token de sistema).
4. Configurar webhook:
   - URL pública HTTPS;
   - `verify_token` para handshake;
   - assinar campo de mensagens.
5. Validar assinatura `X-Hub-Signature-256` no backend.
6. Inscrever eventos de mensagem recebida e status.

---

## Fluxo do webhook (boas práticas)

1. Receber payload.
2. Validar assinatura.
3. Responder `200` rapidamente (< 2s).
4. Publicar evento em fila (SQS/Rabbit/Kafka) para processamento assíncrono.
5. No worker:
   - idempotência por `meta_message_id`;
   - detectar primeira mensagem do lead;
   - classificar origem/nicho;
   - persistir no banco.

---

## KPIs do dashboard por nicho/campanha

- Leads recebidos por dia/semana/mês.
- Taxa de qualificação (% leads qualificados).
- Tempo médio até primeiro atendimento.
- Taxa de resposta em 24h.
- Conversão por etapa (lead -> qualificado -> reunião -> venda).
- Receita por nicho e campanha.
- Custo por lead (se integrar custos da mídia).
- ROAS por nicho/campanha.

### Visualizações sugeridas
- Funil por nicho.
- Tabela dinâmica por campanha x nicho.
- Série temporal de leads e conversão.
- Heatmap de horário/dia para entrada de leads.

---

## Governança e conformidade

- Definir base legal (LGPD) para tratamento de dados.
- Armazenar somente dados necessários.
- Criptografar dados sensíveis em repouso e trânsito.
- Implementar política de retenção de mensagens.
- Controlar acesso por perfil (comercial, marketing, admin).

---

## Roadmap de implementação (30 dias)

### Semana 1
- Subir app da Meta + webhook em ambiente de teste.
- Persistir payload bruto e mensagens.

### Semana 2
- Implementar parser da frase inicial com padrão de campanha (`LP|NICHO|CMP`).
- Implementar classificador por regras.

### Semana 3
- Montar modelo analítico (fato/dimensões).
- Publicar dashboard MVP com KPIs principais.

### Semana 4
- Adicionar fallback NLP e fila de revisão manual.
- Ajustar score de confiança com feedback da operação.

---

## Checklist técnico de produção

- [ ] Webhook com autenticação e assinatura validadas.
- [ ] Reprocessamento idempotente.
- [ ] Retry com DLQ para falhas.
- [ ] Logs estruturados por `lead_id` e `meta_message_id`.
- [ ] Alertas de indisponibilidade webhook.
- [ ] Métricas de latência e throughput.
- [ ] Pipeline de dados diário para dashboard.

---

## Próximo passo recomendado
Começar pelo **MVP com identificação via texto pré-preenchido no link de WhatsApp** (LP/NICHO/CMP), pois entrega rastreabilidade forte desde o primeiro dia. Em paralelo, ativar classificador por regras para cobrir variações de texto livre.
