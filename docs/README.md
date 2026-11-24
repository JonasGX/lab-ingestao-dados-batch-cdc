# Validação de schema do DataFrame de fornecedores

Este documento explica a função `validador_tipagem`, usada no notebook
`notebooks/cases/01_full_load_fornecedores_legado.ipynb`. A função verifica se
o DataFrame possui as colunas esperadas e se os valores são compatíveis com o
schema da tabela `fornecedores_legado`.

## Contrato da função

```python
def validador_tipagem(df: pd.DataFrame, nome_tabela: str) -> int:
```

Parâmetros:

- `df`: DataFrame que será validado;
- `nome_tabela`: nome usado nas mensagens exibidas durante a validação.

Retorno:

- `0`: o DataFrame está de acordo com o schema esperado;
- `1`: pelo menos uma inconsistência foi encontrada.

A função apenas valida e informa os problemas. Ela não altera o DataFrame e
não corrige valores inválidos.

## Schema esperado

| Coluna | Tipo na origem | Regra aplicada no DataFrame |
|---|---|---|
| `id_fornecedor` | `VARCHAR(50)` | valor do tipo `str`, com no máximo 50 caracteres |
| `razao_social` | `VARCHAR(150)` | valor do tipo `str`, com no máximo 150 caracteres |
| `cnpj` | `VARCHAR(14)` | valor do tipo `str`, com no máximo 14 caracteres |
| `status_credito` | `VARCHAR(20)` | valor do tipo `str`, com no máximo 20 caracteres |
| `limite_credito` | `NUMERIC(12,2)` | valor `Decimal` ou `int`, com até 10 dígitos inteiros e 2 casas decimais |

Valores nulos são permitidos. Antes de verificar o conteúdo de cada coluna, a
função usa `dropna()`, portanto `None`, `NaN` e outros valores reconhecidos pelo
pandas como nulos não são considerados erros de tipagem.

## Como a validação funciona

### 1. Define as colunas de texto

```python
schema_texto = {
    "id_fornecedor": 50,
    "razao_social": 150,
    "cnpj": 14,
    "status_credito": 20,
}
```

O dicionário relaciona cada coluna textual ao tamanho máximo permitido. Isso
evita repetir a mesma lógica para cada `VARCHAR`.

### 2. Compara as colunas recebidas com as esperadas

```python
colunas_esperadas = set(schema_texto) | {"limite_credito"}
faltantes = colunas_esperadas - set(df.columns)
extras = set(df.columns) - colunas_esperadas
```

A união dos nomes do dicionário com `limite_credito` forma o conjunto completo
de colunas. A diferença entre conjuntos identifica:

- colunas obrigatórias que não estão no DataFrame;
- colunas inesperadas que vieram no DataFrame.

A comparação não exige que as colunas estejam na mesma ordem do schema.

### 3. Rejeita nomes de colunas duplicados

```python
if df.columns.has_duplicates:
    print(f"Falha na validação de {nome_tabela}: colunas duplicadas.")
    return 1
```

Colunas duplicadas deixam operações como `df[coluna]` ambíguas. Por isso, essa
condição interrompe imediatamente a validação e retorna falha.

### 4. Valida os campos `VARCHAR`

Para cada coluna textual, a função remove os nulos e verifica duas condições:

```python
not isinstance(valor, str) or len(valor) > tamanho
```

Um valor é inválido quando não é uma string ou ultrapassa o limite definido no
schema. Se houver falhas, a mensagem apresenta a quantidade encontrada e até
cinco índices como amostra para facilitar a investigação.

Essa regra valida tipo e tamanho. Ela não verifica o formato ou os dígitos do
CNPJ, nem restringe `status_credito` a uma lista de valores.

### 5. Valida `NUMERIC(12,2)`

A função interna `numeric_valido` trata individualmente cada valor de
`limite_credito`:

1. aceita somente `Decimal` e `int`;
2. rejeita `bool`, embora `bool` seja uma subclasse de `int` em Python;
3. rejeita números infinitos e `NaN` do tipo `Decimal`;
4. exige valor absoluto menor que `10.000.000.000`, permitindo no máximo dez
   dígitos na parte inteira;
5. usa `quantize(Decimal("0.01"))` para garantir no máximo duas casas decimais.

O tipo `float` é rejeitado porque sua representação binária pode introduzir
imprecisão em valores monetários. O `psycopg2` normalmente converte colunas
`NUMERIC` do PostgreSQL para `Decimal`.

O bloco `localcontext()` limita a alteração de precisão ao trecho da validação,
sem modificar o contexto decimal usado pelo restante do programa. Caso a
quantização não seja possível, `InvalidOperation` é capturada e o valor é
considerado inválido.

### 6. Consolida e apresenta os erros

As inconsistências são adicionadas à lista `erros`. Assim, a função consegue
informar várias falhas em uma única execução, em vez de parar no primeiro valor
inválido.

```python
if erros:
    print(f"Falha na validação de {nome_tabela}:")
    for erro in erros:
        print(f"- {erro}")
    return 1

print(f"Schema de {nome_tabela} validado com sucesso.")
return 0
```

## Exemplo de uso

```python
resultado = validador_tipagem(df_fornecedor, "fornecedores_legado")

if resultado != 0:
    raise ValueError("DataFrame inválido. Gravação cancelada.")

print("DataFrame pronto para ser gravado.")
```

Em uma carga para o MinIO, a validação deve ocorrer antes da gravação do
Parquet. Dessa forma, o arquivo e o marcador `_SUCCESS` só são criados quando o
DataFrame está aderente ao schema.

## Exemplos de falha

Se o DataFrame não possuir `cnpj`, trouxer uma coluna chamada `email` e tiver
um valor de crédito com três casas decimais, a saída será semelhante a:

```text
Falha na validação de fornecedores_legado:
- Colunas ausentes: ['cnpj']
- Colunas inesperadas: ['email']
- limite_credito: 1 valores fora de NUMERIC(12,2). Índices de exemplo: [7]
```

O retorno será `1`, permitindo que a função responsável pela carga cancele a
gravação.

## Limites da validação

Esta função valida aderência estrutural e tipagem básica. Ela não verifica:

- obrigatoriedade de valores, porque nulos são permitidos;
- unicidade de `id_fornecedor`;
- tamanho exato e dígitos verificadores do CNPJ;
- valores permitidos para `status_credito`;
- valores mínimos ou máximos de negócio para `limite_credito`;
- ordem das colunas no DataFrame.

Essas regras podem ser implementadas em validadores de qualidade separados e
reunidas pelo `validador_aderencia`.
