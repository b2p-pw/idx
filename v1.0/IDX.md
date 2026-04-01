# 📘 Guia de Referência IDX

A IDX (Instruction Definition eXchange) é uma linguagem de orquestração estruturada desenhada para ser um padrão de intercâmbio entre diferentes sistemas e drivers.

---

## 1. O Manifesto (`@`)

Localizado no topo do arquivo, define o **Contrato de Execução**. O Core valida esses dados antes de iniciar o script.

* `@{idx_version}`: Versão da gramática IDX utilizada.
* `@{idx_scope}`: Define o escopo do script e quais drivers podem ser chamados.

---

## 2. Gerenciamento de Dados (Escopo e Siglas)

A IDX diferencia a origem e a mutabilidade dos dados através de símbolos e caixa (Case).

| Sintaxe | Tipo | Origem | Uso Comum |
| :--- | :--- | :--- | :--- |
| `@{CHAVE}` | **Contrato** | Manifesto | Definições de cabeçalho. |
| `${nome}` | **Variável Interna** | Lógica / Script | `$version`, `$p` (iteradores), `$status`. |
| `${NOME}` | **Constante Interna** | Script | `${binary}`, `${environment}`. |
| `${ext:nome}` | **Variável Externa** | Ambiente (SO) | `$PATH`, `$USER`, `$CONF`, `$LOG`. |
| `${ext:NOME}` | **Constante Externa** | Core / Manifesto | `${PACKAGE}`, `${TEMP}`, `${BIN}`. |

---

## 3. Estruturas de Controle e Fluxo

### `.bloco:`

Define uma unidade de execução. O bloco `[main]:` é o ponto de entrada obrigatório.

* `jump: .bloco` -> Salta para um bloco e volta.
* `jump: !bloco` -> Salta para um bloco e encerra ali.
* `call: "file.idx"; .bloco` -> Executa um bloco (local ou externo) e retorna.

### `if: / then: / else:`

Tomada de decisão baseada em variáveis ou constantes.

``` shell
if: ${sys:ENV} == production
    then: ${environment}: prod
    else: ${environment}: dev
```

### `for: $(var) in $(lista)`

Iteração sobre coleções de dados.

``` shell
for: ${p} in ${plugins}
    win.copy: ${win:TEMP}/${p}; ${BIN}/plugins/${p}
```

### `match: ${idx:exit_code}` (Switch/Case)

Tratamento granular de códigos de saída ou valores.

* `0:` -> Sucesso.
* `*:` -> Catch-all (qualquer outro valor).

---

## 4. Operadores de Comando

* **Dois Pontos (`:`)**: Atribuição ou início de comando (`print: Oi`).
* **Ponto e Vírgula (`;`)**: Separador de argumentos principais (objetos diferentes).
* **Vírgula (`,`)**: Separador de itens de mesma natureza (listas ou atributos).
* **Pipe (`|`)**: Redireciona a saída (STDOUT) para uma variável (`comando | ${var}`).
* **Seta (`>` / `>>`)**: Escrita ou anexação em arquivos (`texto > arquivo.txt`).
* **Operadores de Erro (`&&:` / `||:`)**:
    * `&&:` Executa o bloco indentado se o comando anterior for 0 (Sucesso).
    * `||:` Executa o bloco indentado se o comando anterior falhar (Erro).

---

## 5. Drivers, Múdulos e Namespacing

A IDX não executa ações diretamente; ela orquestra drivers via aliases definidos no topo.

* **Sintaxe**: `driver: DRIVER_ID; alias`
* **Chamada**: `alias.comando: argumento`

Diferente do `call`, o `import` carrega um arquivo externo como um **Driver Virtual**. Isso permite organizar bibliotecas de funções reutilizáveis.

* **Sintaxe**: `import: ./scipt.idx; alias`
* **Chamada**: `alias.bloco: ${arg1}: valor, ${arg2}: valor`

> **Nota**: O `import` é processado no carregamento (estático), enquanto o `call` é processado na execução (dinâmico).

---

## 6. Tratamento Global de Exceções

O comando `on_error` define o comportamento padrão para qualquer falha não tratada no script.

``` shell
on_error: !error_block
# Dentro do código, pode-se forçar a chamada:
error: !error_block
```

---

## 7. Sistema de Escopos e Segurança

A IDX utiliza um sistema de **Capacidades Declarativas**. O `@{IDX_SCOPE}` do script principal dita o que é permitido em todo o fluxo.

### Hierarquia de Escopo

1. **Escopo Pai (Global)**: Define o teto máximo de permissões (ex: `[package]`).
2. **Escopo de Módulo (Local)**: O `import` ou `call` pode ter um escopo específico (ex: `[database]`).

### Regras de Validação:

* **Princípio do Privilégio Mínimo**: Se o script inicial tem escopo `[package]`, ele **não pode** chamar um módulo que exija `[kernel_admin]` a menos que o Core autorize explicitamente via manifesto.
* **Conflito de Escopo**: Se um script `package` tentar importar um módulo `database`:
    * O Core detecta a elevação de privilégio.
    * **Ação**: O Runner bloqueia a execução e emite um erro de "Scope Mismatch", a menos que o manifesto `@` do script pai inclua o escopo secundário: `@{IDX_SCOPE}: [package, database]`.

---

## 8. Caracteres de Escape e Strings

Para utilizar caracteres reservados da sintaxe IDX (como `:`, `;`, `,`, `$`) como texto literal, utiliza-se a barra invertida (`\`).

### Literais Protegidos

Sempre que um caractere especial for necessário dentro de uma string ou argumento, ele deve ser escapado para evitar que o Parser o interprete como um comando.

* `\;` -> Insere um ponto e vírgula literal.
* `\,` -> Insere uma vírgula literal.
* `\$` -> Insere o símbolo de cifrão sem disparar a busca por variáveis.
* `\:` -> Insere dois pontos sem indicar atribuição.
* `\\` -> Insere uma barra invertida literal.

``` shell
# Sem escape (Parser entende como dois argumentos separados por ;)
term.print: Senha: 123;456 

# Com escape literal
term.print: Senha: 123\;456
```

---

## Exemplo de Fluxo Completo:
``` shell
# 1. Valida Contrato
@{IDX_VERSION}: 1
@{IDX_SCOPE}: [download, database]

# 2. Carrega Ferramentas
driver: REQUEST; req
driver: WINDOWS_NT; win

import: ./scripts/db_manager.idx; db

${binary}: "https://repo.com/app.zip"

.main:
    # 3. Execução Procedural via Call
    call: ./check_env.idx; .validate

    # 4. Orquestração de Módulo Importado
    # O Core valida se 'db' tem permissão para o escopo [database]
    db.setup: ${name}: prod_db, $(port): 5432
        ||:
            print: Erro ao configurar banco de dados.
            error: .cleanup

    # 5. Comando de Driver Nativo
    req.download: ${binary}; ${win:TEMP}/app.zip
```