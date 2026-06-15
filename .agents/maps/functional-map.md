# Mapa Funcional — lead-collector

## Visão geral

Sistema SPA mobile-first para captação de leads em eventos presenciais. Dois domínios de negócio independentes: o **Lead** (ciclo de vida completo da entidade) e o **Compartilhamento WhatsApp** (formatação e envio).

---

## Domínios

### 1. Lead

**Objetivo de negócio:** Gerenciar o ciclo de vida completo de um lead capturado em evento — cadastro, validação, persistência local e listagem/exclusão.

**Evidências no material fornecido:** `requisitos-captacao-leads.md` — seções 3.1 (RF01–RF03), 3.2 (RF04–RF07), 3.3 (RF08–RF11), 3.5 (RF15–RF17), seção 5 (RN01, RN02, RN04).

**Responsabilidades:**

- Formulário de captura com campos: Nome, Empresa, Cargo (opt.), Telefone, E-mail, Dor identificada, Temperatura, Próximo passo (opt.).
- Validação de negócio: lead válido exige nome, empresa, dor, temperatura e ao menos um meio de contato (RF01, RN01).
- Seleção de temperatura por botões visuais com cor e emoji (RF02).
- Otimizações de UX mobile: foco automático, avanço via Enter/próximo, avanço ao selecionar temperatura, teclados nativos, autocomplete (RF04–RF07).
- Persistência local no dispositivo via **IndexedDB** com a biblioteca `idb`, com data/hora de criação e ID único imutável (RF08, RF09, RN04).
- Listagem de leads do mais recente ao mais antigo com exibição de nome, empresa, cargo, data/hora, temperatura e dor resumida (RF10, RF17).
- Exclusão individual de lead (RF11).
- Feedback pós-salvamento: toast "Lead salvo e enviado" e scroll ao topo (RF15).
- Contador de leads no cabeçalho (RF16).

**Regras de negócio chave:**

- Telefone OU e-mail: pelo menos um dos dois é obrigatório (RN01).
- Temperatura é sempre obrigatória (RN02).
- ID imutável gerado no momento da criação (RN04).

**Dependências entre domínios:** nenhuma (domínio fundacional).

**Dependências externas:** `idb` (wrapper para IndexedDB — decisão humana confirmada); `shadcn/ui` (componentes de formulário, toast, lista); `react-hook-form` ou similar (gestão de formulário — a confirmar nas próximas etapas); `zod` (validação de schema — a confirmar).

**Grau de confiança:** alto — evidência direta e extensa no material fornecido.

---

### 2. Compartilhamento WhatsApp

**Objetivo de negócio:** Formatar os dados de um lead conforme template padronizado e abrir o WhatsApp para o usuário escolher o destinatário, sem armazenar contatos no sistema.

**Evidências no material fornecido:** `requisitos-captacao-leads.md` — seção 3.4 (RF12–RF14), seção 5 (RN03).

**Responsabilidades:**

- Composição da mensagem no formato padronizado com campos condicionais (RF14):
  - Linha de título com emoji e temperatura.
  - Nome, Empresa obrigatórios; Cargo, Telefone, E-mail e Próximo passo incluídos apenas se preenchidos.
- Geração do deeplink `https://wa.me/?text=<mensagem_codificada>` para abertura do WhatsApp sem destinatário fixo.
- Acionamento automático ao salvar um lead (RF12).
- Acionamento manual via botão "Reenviar no WhatsApp" em cada item da lista (RF13).

**Regras de negócio chave:**

- O destinatário é sempre definido pelo usuário no ato do envio; o sistema não armazena contatos (RN03).
- O mesmo formato de mensagem é usado tanto no envio imediato quanto no reenvio (RF14).

**Dependências entre domínios:** Lead (requer os dados do lead para formatar a mensagem).

**Dependências externas:** `wa.me` deeplink (API pública do WhatsApp, sem SDK).

**Grau de confiança:** alto — evidência direta no material fornecido.

---

## Dependências entre domínios

```
Lead ←── Compartilhamento WhatsApp
```

WhatsApp Sharing consome dados do domínio Lead; Lead não depende de Sharing.

---

## Ordem de implementação sugerida

| Ordem | Domínio | Justificativa |
|-------|---------|---------------|
| 0 | Lead | Domínio fundacional; contém a entidade central, persistência e toda a lógica de negócio principal. |
| 1 | Compartilhamento WhatsApp | Depende dos dados do Lead; pode ser implementado e testado de forma isolada após o domínio Lead estar estável. |
