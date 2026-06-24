# Documento de Requisitos — Sistema de Captação de Leads em Eventos

## 1. Visão Geral

Aplicação otimizada para celular, destinada ao preenchimento rápido de informações de leads capturados durante eventos presenciais. Após o cadastro, o lead é armazenado localmente no dispositivo e compartilhado via WhatsApp para o contato/grupo escolhido pelo usuário no momento do envio.

## 2. Objetivos de Negócio

- Reduzir ao mínimo o tempo de preenchimento de um lead durante um evento.
- Garantir que nenhum lead capturado seja perdido (persistência local).
- Facilitar o repasse imediato do lead para a equipe comercial via WhatsApp.
- Padronizar o formato da informação enviada para facilitar follow-up.

## 3. Requisitos Funcionais

### 3.1 Cadastro de Lead

- **RF01** — O sistema deve permitir cadastrar um lead com os seguintes campos:
  - Nome (obrigatório)
  - Empresa (obrigatório)
  - Cargo (opcional)
  - Telefone (obrigatório se e-mail não for informado)
  - E-mail (obrigatório se telefone não for informado)
  - Dor identificada (obrigatório)
  - Temperatura do lead (obrigatório): **Frio**, **Morno** ou **Quente**
  - Próximo passo combinado (opcional)

- **RF02** — A seleção da temperatura deve ser feita por botões visuais com cores e emojis distintos:
  - Frio 🧊
  - Morno 🌤️
  - Quente 🔥

- **RF03** — Ao salvar, o sistema deve validar todos os campos obrigatórios e exibir mensagens de erro junto a cada campo inválido, posicionando o foco no primeiro campo com erro.

### 3.2 Otimização de Preenchimento

- **RF04** — O primeiro campo do formulário deve receber foco automaticamente ao abrir a tela.
- **RF05** — Pressionar "Enter" (ou a tecla de avanço do teclado mobile) deve mover o foco para o próximo campo do formulário.
- **RF06** — Ao selecionar a temperatura, o foco deve avançar automaticamente para o próximo campo.
- **RF07** — Os campos devem aproveitar recursos nativos do teclado mobile:
  - Capitalização automática em campos de nome, empresa, cargo e textos livres.
  - Teclado numérico/telefone no campo de telefone.
  - Teclado de e-mail no campo de e-mail.
  - Sugestões de autocompletar (nome, organização, cargo, telefone, e-mail).

### 3.3 Persistência

- **RF08** — Cada lead salvo deve ser armazenado localmente no dispositivo do usuário e permanecer disponível após fechar e reabrir a aplicação.
- **RF09** — Cada lead deve registrar a data/hora de criação.
- **RF10** — A lista de leads cadastrados deve ser exibida abaixo do formulário, ordenada do mais recente para o mais antigo.
- **RF11** — Deve ser possível excluir um lead da lista.

### 3.4 Compartilhamento via WhatsApp

- **RF12** — Após salvar um lead, o sistema deve abrir automaticamente o WhatsApp com a mensagem do lead já preenchida, permitindo que o usuário selecione um contato ou grupo de destino.
- **RF13** — Deve haver um botão "Reenviar no WhatsApp" em cada lead da lista, repetindo o mesmo fluxo de seleção de destino.
- **RF14** — A mensagem enviada deve seguir o seguinte formato padronizado:

  ```
  *Novo Lead — {emoji} {Temperatura}*

  *Nome:* {nome}
  *Empresa:* {empresa}
  *Cargo:* {cargo}            (somente se preenchido)
  *Telefone:* {telefone}       (somente se preenchido)
  *E-mail:* {email}            (somente se preenchido)

  *Dor identificada:*
  {dor}

  *Próximo passo:*             (somente se preenchido)
  {próximo passo}
  ```

### 3.5 Feedback ao Usuário

- **RF15** — Após salvar com sucesso, exibir uma notificação temporária ("Lead salvo e enviado") e rolar a tela para o topo.
- **RF16** — O cabeçalho deve mostrar a quantidade total de leads cadastrados.
- **RF17** — Cada lead na lista deve exibir: nome, empresa, cargo (se houver), data/hora, temperatura com cor/emoji correspondente e a dor identificada (resumida).

## 4. Requisitos Não Funcionais

### 4.1 Usabilidade

- **RNF01** — A aplicação deve ser otimizada para uso em celular (largura máxima de conteúdo equivalente a um único toque visível, sem necessidade de zoom).
- **RNF02** — O preenchimento completo de um lead típico deve ser possível em menos de 30 segundos.
- **RNF03** — Os campos devem ser compactos para minimizar a rolagem necessária para concluir o cadastro.
- **RNF04** — Todos os textos e botões devem ter contraste adequado para leitura em ambiente com iluminação variada (eventos).

### 4.2 Visual

- **RNF05** — O tema da aplicação deve ser escuro (dark mode) por padrão, para reduzir o consumo de bateria em telas OLED.
- **RNF06** — As cores de temperatura devem ser semanticamente reconhecíveis (frio = azul, morno = amarelo/laranja, quente = vermelho).
- **RNF07** — O botão principal de ação deve usar a identidade visual do WhatsApp (verde) para reforçar a ação esperada.

### 4.3 Disponibilidade e Resiliência

- **RNF08** — A aplicação deve funcionar sem conexão com a internet para o cadastro e armazenamento local de leads.
- **RNF09** — Apenas o envio via WhatsApp requer conexão; os dados já cadastrados não devem ser perdidos caso o envio falhe.
- **RNF10** — Os dados armazenados localmente não devem ser apagados sem ação explícita do usuário.

### 4.4 Privacidade

- **RNF11** — Os dados dos leads não devem ser enviados a nenhum servidor externo da aplicação; permanecem exclusivamente no dispositivo do usuário até serem compartilhados manualmente via WhatsApp.
- **RNF12** — A aplicação não deve exigir cadastro, login ou qualquer identificação do usuário operador.

### 4.5 Performance

- **RNF13** — A tela inicial deve estar pronta para preenchimento em menos de 2 segundos em conexão móvel comum.
- **RNF14** — A inserção, exclusão e listagem de leads devem ser instantâneas (sem indicadores de carregamento perceptíveis).

## 5. Regras de Negócio

- **RN01** — Um lead só é considerado válido se possuir, no mínimo: nome, empresa, dor identificada, temperatura e um meio de contato (telefone **ou** e-mail).
- **RN02** — A temperatura classifica o nível de interesse/maturidade do lead e é informação obrigatória para o follow-up comercial.
- **RN03** — O destinatário do WhatsApp é definido a cada envio pelo próprio usuário — o sistema não armazena contatos.
- **RN04** — Cada lead salvo gera um identificador único e imutável.

## 6. Fora de Escopo

- Sincronização entre dispositivos.
- Edição de leads já cadastrados (apenas criação, listagem e exclusão).
- Integração com CRM ou planilhas.
- Relatórios, filtros ou busca na lista de leads.
- Autenticação ou múltiplos usuários.
