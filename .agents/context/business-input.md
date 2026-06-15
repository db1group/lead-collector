# Entrada de Negócio — lead-collector

## Fonte principal

Arquivo: `/Users/douglas.junior/Downloads/requisitos-captacao-leads.md`

---

## Visão geral do sistema

Aplicação de uso pessoal, otimizada para celular, para preenchimento rápido de informações de leads capturados durante eventos presenciais. Após o cadastro, o lead é armazenado localmente no dispositivo e compartilhado via WhatsApp para o contato/grupo escolhido pelo usuário no momento do envio.

## Objetivos de negócio

- Reduzir ao mínimo o tempo de preenchimento de um lead durante um evento.
- Garantir que nenhum lead capturado seja perdido (persistência local).
- Facilitar o repasse imediato do lead para a equipe comercial via WhatsApp.
- Padronizar o formato da informação enviada para facilitar follow-up.

---

## Requisitos funcionais (resumo por área)

### Cadastro de Lead (RF01–RF07)

- Campos: Nome\*, Empresa\*, Cargo (opt.), Telefone\*, E-mail\*, Dor identificada\*, Temperatura\*, Próximo passo (opt.).  
  (\* = obrigatório com regra de validação — veja RN01)
- Temperatura: seleção visual por botões com cor e emoji — Frio 🧊, Morno 🌤️, Quente 🔥.
- Validação ao salvar: mensagens junto ao campo inválido; foco no primeiro campo com erro.
- Otimizações mobile: foco automático no primeiro campo, avanço via Enter/próximo, avanço automático ao selecionar temperatura, teclados nativos (numérico, e-mail), autocomplete.

### Persistência (RF08–RF11)

- Armazenamento local no dispositivo (sem servidor externo).
- Registro de data/hora de criação por lead.
- Lista de leads abaixo do formulário, ordenada do mais recente para o mais antigo.
- Exclusão individual de leads.

### Compartilhamento via WhatsApp (RF12–RF14)

- Ao salvar, abrir automaticamente o WhatsApp com mensagem pré-formatada; usuário escolhe o destinatário.
- Botão "Reenviar no WhatsApp" em cada lead da lista.
- Formato padronizado da mensagem (template com campos condicionais para Cargo, Telefone, E-mail e Próximo passo).

### Feedback ao Usuário (RF15–RF17)

- Toast "Lead salvo e enviado" após salvar; rolar para o topo.
- Cabeçalho exibe contagem total de leads.
- Cada item da lista exibe: nome, empresa, cargo (se houver), data/hora, temperatura (cor+emoji), dor resumida.

---

## Regras de negócio

| Código | Regra |
|--------|-------|
| RN01 | Lead válido exige: nome, empresa, dor identificada, temperatura e ao menos um meio de contato (telefone **ou** e-mail). |
| RN02 | Temperatura classifica o nível de interesse/maturidade do lead; é obrigatória para follow-up comercial. |
| RN03 | Destinatário do WhatsApp é definido a cada envio pelo usuário — o sistema **não** armazena contatos. |
| RN04 | Cada lead salvo recebe um identificador único e imutável. |

---

## Fora de escopo (explícito no documento)

- Sincronização entre dispositivos.
- Edição de leads (apenas criação, listagem e exclusão).
- Integração com CRM ou planilhas.
- Relatórios, filtros ou busca.
- Autenticação ou múltiplos usuários.
