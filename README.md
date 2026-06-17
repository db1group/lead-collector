# Lead Collector

Aplicação web otimizada para celular, para captação rápida de leads
em eventos presenciais. O lead é cadastrado em poucos segundos, armazenado localmente
no dispositivo e compartilhado via WhatsApp para o contato ou grupo escolhido no momento
do envio — sem servidores, sem login e funcionando offline.

## 1. Requisitos

- Node.js 22+
- Yarn 4

## 2. Instalação das Dependências

```sh
yarn install
```

## 3. Rodar em Desenvolvimento

```sh
yarn dev
```

O Vite sobe o servidor de desenvolvimento com HMR. Para testar diretamente no celular
na mesma rede, exponha o host:

```sh
yarn dev --host
```

## 4. Tecnologias Principais

- `TypeScript` com `ESM` (`type: module`)
- `React 19` para a interface
- `Vite` como dev server e bundler de produção
- `Vitest` (ambiente `jsdom`) para testes automatizados
- `ESLint` (flat config) para padronização de código
- Alias `@` apontando para `src/`

## 5. Build e Execução Local

```sh
yarn build
yarn preview
```

O comando `yarn build` roda o type check (`tsc -b`) e gera a saída de produção em
`dist/`; o `yarn preview` serve localmente o build gerado.

## 6. Fluxo do Produto

O fluxo-alvo prioriza velocidade de preenchimento durante o evento:

1. Cadastro do lead em formulário compacto (nome, empresa, contato, dor e temperatura)
2. Validação e persistência local do lead no dispositivo
3. Abertura automática do WhatsApp com a mensagem padronizada já preenchida
4. Gestão da lista de leads cadastrados (visualização, reenvio e exclusão)

## 7. Lint e Testes

```sh
# lint
yarn lint
# type check
yarn ts

# testes
yarn test
# testes com cobertura
yarn test:coverage
```

> `yarn lint --fix` para correção automática de problemas de lint.

> Ao rodar `yarn test:coverage` o relatório de cobertura estará disponível em [coverage/index.html](coverage/index.html).

## 8. Documentação e Referências

- Requisitos do produto: [requisitos-captacao-leads.md](requisitos-captacao-leads.md) —
  fonte de verdade dos requisitos funcionais, não funcionais e regras de negócio.
- Skills do projeto: [.agents/skills/](.agents/skills/) — padrões técnicos para agentes
  de geração de código (React Hook Form + Zod, shadcn/ui, Vite, testes e boas práticas).
