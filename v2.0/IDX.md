# 📘 Guia de Referência IDX

A IDX (Instruction Definition eXchange) é uma linguagem de orquestração estruturada desenhada para ser um padrão de intercâmbio entre diferentes sistemas e módulos.

## 1. Gerenciamento de Dados (Escopo e Siglas)

A IDX diferencia a origem e a mutabilidade dos dados através de símbolos e caixa (Case).

| Sintaxe | Tipo | Origem | Uso Comum |
| :--- | :--- | :--- | :--- |
| `{nome}` | **Variável Interna** | Lógica / Script | `{version}`, `{i}`, `{status}`. |
| `{NOME}` | **Constante Interna** | Script | `{BINARY}`, `{DB_NAME}`. |
| `{ext.nome}` | **Variável Externa** | Ambiente (SO) | `{sys.conf}`, `{sys.env}`. |
| `{ext.NOME}` | **Constante Externa** | Core / Manifesto | `{sys.ROOT}`. |

---

## 2. Estruturas de Controle e Fluxo

### `fn funcao:`

Define uma unidade de execução.

* `funcao`: Roda função e volta.
* `script.funcao`: Executa uma função de um scrip ou modulo importado e retorna.

### `var = "value" if...`

Pode ser definir "if in line".

``` coffescript
env_type = "prod" if {sys.env} == "production" else "dev"
```

### `for {var} in {lista}`

Iteração sobre coleções de dados.

``` coffescript
for plugin in "auth.dll", "logs.dll":
    sys.copy "{sys.TEMP}/{plugin}" to "{b2p.bin}/plugins/"
```

### `match funcao` (Switch/Case)

Tratamento granular de códigos de saída ou valores.

* `{res}`: Sucesso com saida simples.
* `{res, 0}`: Sucesso com saida em array.
* `{err}`: Saida com erro.
* `*`: Saida coringa.

---

## 3. Operadores de Comando

* **Dois Pontos (`:`):** Início de bloco, podendo ou não ser identado (`try: term.print "{sys.time}"`).
* **Igual (`=`):** Atribuição de valor (`var = 123`).
* **Vírgula (`,`):** Separador de itens de mesma natureza como listas, atributos, ou arrays (`array = 1, 2, 3`).

---

## 4. Módulos e Namespacing

A IDX não executa ações diretamente; ela orquestra módulos importados em `allow`.

* **Sintaxe:** `allow: sys`
* **Chamada:** `sys.comando argumento`

Diferente do `run`, o `use` carrega um arquivo. Isso permite organizar bibliotecas de funções reutilizáveis.

* **Sintaxe:** `use "./scipt.idx" as alias`
* **Chamada:** `alias.bloco`

---

## 5. Tratamento Global de Exceções

O comando `on_error` define o comportamento padrão para qualquer falha não tratada no script.

``` coffescript
on_error: error_block
```

Ou

``` coffescript
on_error: "script.idx"
```

---

## 6. Sistema de Escopos e Segurança

A IDX utiliza um sistema de **Capacidades Declarativas**. O `allow` do script principal dita o que é permitido em todo o fluxo.

### Hierarquia de Escopo

1. **Escopo Pai (Global):** Define o teto máximo de permissões (ex: `package`).
2. **Escopo de Módulo (Local):** O `use` ou `run` pode ter um escopo específico (ex: `database`).

### Regras de Validação:

* **Princípio do Privilégio Mínimo:** Se o script inicial tem escopo `package`, ele **não pode** chamar um módulo que exija `kernel_admin` a menos que o Core autorize explicitamente via manifesto.
* **Conflito de Escopo:** Se um script `package` tentar importar um módulo `database`:
    * O Core detecta a elevação de privilégio.
    * **Ação:** O Runner bloqueia a execução e emite um erro de "Scope Mismatch", a menos que o script pai inclua o escopo secundário: `allow: package, database`.

---

## 7. Caracteres de Escape e Strings

Para utilizar caracteres reservados da sintaxe IDX (como `{}`) como texto literal, utiliza-se a barra invertida (`\`).

``` coffescript
term.print "\{Literal\}"    # Saida do terminal: {Literal}
```

## 8. Definição de tipos

A IDX permite que variáveis transitem entre um estado Dinâmico (flexível) e um estado Restrito (contrato rígido) através do operador `as`.

* Sem a declaração `as`, a variável é um ponteiro genérico. Ela aceita qualquer tipo de dado e pode ser sobrescrita por tipos diferentes a qualquer momento:

``` coffescript
var = "abacate", "pera"    # Nasce como Array/String
var = 1                    # Muda para Number
var = "boliche"            # Volta para String
```

* O operador `as` à direita de uma expressão realiza a conversão volátil do dado. Se o valor à esquerda das chaves for compatível, ele é transformado antes da atribuição.

``` coffescript
var = "123"              # Valor é definido como String.
var = {var} as number    # O valor é convertido e reatribuído.
```

* Quando as tipo é utilizado à esquerda do operador de atribuição (`=`), ele define um contrato permanente para aquele slot de memória. A partir desse momento, a variável deixa de ser dinâmica.

> **Regra:** Qualquer tentativa de atribuir um valor de tipo diferente resultará em um erro de TypeMismatch, a menos que o valor passe por uma conversão explícita.

``` coffescript
var = "boliche"          # Estado inicial: Dinâmico.

var as number = "123"    # O slot 'var' é bloqueado como 'number'. O valor "123" é convertido automaticamente.

var = 321                # PASSOU: Tipo compatível.
var = "456" as number    # PASSOU: Conversão explícita antes da entrada.
var = "654"              # ERRO:   Tentativa de injetar String em slot Number.
```