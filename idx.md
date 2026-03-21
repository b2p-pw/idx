# 📘 Guia de Referência IDX (V1.0)

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
| `@{chave}` | **Contrato** | Manifesto | Definições de cabeçalho. |
| `${NOME}` | **Constante Externa** | Core / Manifesto | `${PACKAGE}`, `${TEMP}`, `${BIN}`. |
| `${nome}` | **Constante Interna** | Script | `${binary}`, `${environment}`. |
| `$NOME` | **Variável Externa** | Ambiente (SO) | `$PATH`, `$USER`, `$CONF`, `$LOG`. |
| `$nome` | **Variável Interna** | Lógica / Script | `$version`, `$p` (iteradores), `$status`. |

---

## 3. Estruturas de Controle e Fluxo

### `(bloco):`
Define uma unidade de execução. O bloco `[main]:` é o ponto de entrada obrigatório.
* `jump: (bloco)` -> Salta para um bloco e volta.
* `jump: (bloco)!` -> Salta para um bloco e encerra ali.
* `call: "file.idx" ; (bloco)` -> Executa um bloco (local ou externo) e retorna.

### `if: / then: / else:`
Tomada de decisão baseada em variáveis ou constantes.
``` shell
if: ${ENV} == production
    then: ${environment}: "prod"
    else: ${environment}: "dev"
```

### `for: $var in $lista`
Iteração sobre coleções de dados.
``` shell
for: $p in $plugins
    win.copy: ${TEMP}/$p ; $BIN/plugins/$p
```

### `match: $source` (Switch/Case)
Tratamento granular de códigos de saída ou valores.
* `0:` -> Sucesso.
* `*:` -> Catch-all (qualquer outro valor).

---

## 4. Operadores de Comando

* **Dois Pontos (`:`)**: Atribuição ou início de comando (`print: "Oi"`).
* **Ponto e Vírgula (` ; `)**: Separador de argumentos principais (objetos diferentes).
* **Vírgula (` , `)**: Separador de itens de mesma natureza (listas ou atributos).
* **Pipe (` | `)**: Redireciona a saída (STDOUT) para uma variável (`comando | $var`).
* **Seta (` > ` / ` >> `)**: Escrita ou anexação em arquivos (`"texto" > arquivo.txt`).
* **Operadores de Erro (`&&:` / `||:`)**:
    * `&&:` Executa o bloco indentado se o comando anterior for 0 (Sucesso).
    * `||:` Executa o bloco indentado se o comando anterior falhar (Erro).

---

## 5. Drivers e Namespacing
A IDX não executa ações diretamente; ela orquestra drivers via aliases definidos no topo.
* **Sintaxe**: `driver: ${DRIVER_ID} ; "alias"`
* **Chamada**: `alias.comando: argumento`

---

## 6. Tratamento Global de Exceções
O comando `on_error` define o comportamento padrão para qualquer falha não tratada no script.
``` shell
on_error: (error_block)
# Dentro do código, pode-se forçar a chamada:
error: (error_block)
```

---

### Exemplo de Fluxo Completo:
``` shell
# 1. Valida Contrato
@{idx_version}: 1
@{idx_scope}: [download]
@{kernel}: windows-nt

# 2. Carrega Ferramentas
driver: WINDOWS_NT ; "win"

# 3. Define Constantes
${url}: "https://repo.com"

(main):
    # 4. Executa com Segurança
    win.download: ${url}/app.zip ; ${TEMP}
        ||:
            print: "Falha crítica no download"
            stop: 1
```