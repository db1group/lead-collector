---
name: typescript-best-practices
description: Boas práticas de TypeScript para garantir código limpo, legível e fácil de manter.
---

# TypeScript

O projeto faz uso de TypeScript para garantir a tipagem dos dados e melhorar a qualidade do código.

Neste sentido, todo código útil (leia-se, componentes, telas, utils, etc) deve ser escrito em TypeScript, exceções apenas para scripts de configuração, como `eslint.config.js`, etc.

## Boas práticas

1. Nomeie classes usando `CamelCase` com inicial maiúscula e mantenha o nome do arquivo igual ao da classe.

    **Motivação**: facilita a navegação entre arquivos e classes, mantendo consistência entre importações e o conteúdo real.

    **Exemplo**:

    ```ts
    export class ListProjectsUseCase {
      // ...
    }
    // Arquivo: ListProjectsUseCase.ts
    ```

1. Interfaces devem usar `CamelCase` com o prefixo `I`.

    **Motivação**: o prefixo destaca rapidamente que o símbolo é uma interface e previne colisões com classes concretas.

    **Exemplo**:

    ```ts
    export interface IAgentService {
      execute(): Promise<void>;
    }
    ```

1. Componentes React devem seguir `CamelCase` com inicial maiúscula.

    **Motivação**: o React identifica componentes por nomes em maiúsculo, além de manter o padrão de legibilidade do JSX.

    **Exemplo**:

    ```tsx
    export function ProjectCard(): JSX.Element {
      return <div>Projeto</div>;
    }
    ```

1. Evite comentários óbvios:

    **Motivação**: Comentários devem ser realizados apenas para explicar decisões "não obvias" que foram tomadas e não para ilustrar o que o código faz.

    **Exemplo**:

    ```ts
    // Correto ✅
    /*
     * Implementa geração aleatória manualmente para evitar de instalar dependência externa.
     */
    return crypto.randomBytes(6).toString('base64').replace(/[^a-zA-Z0-9]/g, '').slice(0, 8) || 'Abc12345';

    // Incorreto ❌
    /*
     * Gera string alfanumérica e garante 8 chars
     */
    return crypto.randomBytes(6).toString('base64').replace(/[^a-zA-Z0-9]/g, '').slice(0, 8) || 'Abc12345';
    if (!user) return;
    ```

1. Prefira `type` em vez de `interface` quando possível

    **Motivação**: `type` oferece mais flexibilidade, permitindo a criação de tipos complexos através de interseções e uniões.

    **Exemplo**:

    ```ts
    // Correto ✅
    type User = {
      name: string;
      age: number;
    };

    // Incorreto ❌
    interface User {
      name: string;
      age: number;
    }
    ```

1. Use `const` em vez de `let` sempre que possível.

    **Motivação**: `const` garante que uma variável não será reatribuída após a sua criação.

    **Exemplo**:

    ```ts
    // Correto ✅
    const name = 'John';

    // Incorreto ❌
    let name = 'John';
    ```

1. Use `generics` para tornar componentes e funções mais flexíveis

    **Motivação**: `generics` permitem que componentes e funções possam ser usadas com diferentes tipos de dados.

    **Exemplo**:

    ```tsx
    // Correto ✅
    function List<Item>(props: { data: Item[] }): JSX.Element {
      return (
        <View>
          {props.data.map((item, index) => (
            <Item item={item} />
          ))}
        </View>
      );
    }

    // Incorreto ❌
    function List(props: { data: any[] }): JSX.Element {
      return (
        <View>
          {props.data.map((item, index) => (
            <Item item={item} />
          ))}
        </View>
      );
    }
    ```

1. Utilize tipos literais para restringir valores possíveis.

    **Motivação**: tipos literais permitem restringir os valores possíveis de uma propriedade.

    **Exemplo**:

    ```ts
    // Correto ✅
    type StatusType = 'active' | 'inactive' | 'pending';

    // Incorreto ❌
    type StatusType = string;
    ```

1. Utilize `?` para indicar que uma propriedade é opcional.

    **Motivação**: Propriedades opcionais tornam os tipos mais flexíveis e evitam erros.

    **Exemplo**:

    ```ts
    // Correto ✅
    type User = {
      name: string;
      age?: number;
    };

    // Incorreto ❌
    type User = {
      name: string;
      age: number | undefined;
    };
    ```

1. Evite `!` (non-null assertion operator) a menos que absolutamente necessário.

    **Motivação**: O uso de `!` pode levar a erros em tempo de execução se o valor for realmente `null` ou `undefined`.

    **Exemplo**:

    ```ts
    // Correto ✅
    const user: UserType | null = getUser();
    if (user) {
      console.log(user.name);
    }

    // Correto ✅
    const user: UserType | null = getUser();
    if (!user) {
      throw new Error('User not found');
    }
    console.log(user.name);

    // Incorreto ❌
    const user: UserType | null = getUser();
    console.log(user!.name); // Pode causar erro se user for null
    ```

    > **Nota**: Caso o uso do `!` seja necessário, deve ser documentado com um comentário explicando o motivo.

1. Use `readonly` sempre que possível para propriedades de componentes.

    **Motivação**: `readonly` garante que uma propriedade não será modificada após a sua criação.

    **Exemplo**:

    ```ts
    // Correto ✅
    type ButtonPropsType = {
      readonly title: string;
      readonly onPress: () => void;
    };

    // Incorreto ❌
    type ButtonPropsType = {
      title: string;
      onPress: () => void;
    }
    ```

1. Exporte tipos diretamente na declaração quando o símbolo já nasce público.

    **Motivação**: declarar `export type` no próprio símbolo reduz ruído no rodapé do arquivo, evita listas de reexport desnecessárias e deixa a intenção pública explícita no ponto em que o tipo é definido.

    **Exemplo**:

    ```ts
    // Correto ✅
    export type UserRole = 'admin' | 'viewer';

    export type Session = {
      userRoles: UserRole[];
    };

    // Incorreto ❌
    type UserRole = 'admin' | 'viewer';

    type Session = {
      userRoles: UserRole[];
    };

    export type { Session, UserRole };
    ```
