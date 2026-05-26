# Aula Prática: Property-Based Testing (PBT) com Pandas

## Objetivo
Este tutorial se baseia nos conceitos de teste de propriedades com **Hypothesis** (https://hypothesis.works/). O objetivo é aprender a testar propriedades gerais de um pipeline de dados em vez de testar apenas exemplos fixos. Você usará o framework Hypothesis para gerar automaticamente cenários de dados extremos e validar se suas funções em pandas respeitam invariantes de integridade, evoluindo o código passo a passo.

## Instruções
- Siga o roteiro na ordem. Os passos são incrementais.
- Execute os testes (pytest) após cada alteração para ver o Hypothesis em ação.
- Nos passos marcados com `COMMIT & PUSH`, envie suas alterações para o repositório. O histórico de commits será avaliado para garantir que você seguiu a sequência do tutorial.

## Avaliação
A atividade vale 4,0 pontos (8 testes estruturados $\times$ 0,5 ponto cada).

## Configuração Inicial
Crie uma pasta para o projeto e instale as dependências:
```
pip install hypothesis pandas pytest
```
## Roteiro
### Passo 1: O Começo do Pipeline (Faturamento Numérico)
Vamos criar uma função que recebe dados brutos de faturamento e precisa garantir que eles sejam numéricos e sem valores nulos.

(1.1) Crie o arquivo `pipeline.py`:
```python
import pandas as pd

def limpar_faturamento(df: pd.DataFrame) -> pd.DataFrame:
    df_clean = df.copy()
    if 'faturamento' in df_clean.columns:
        # Converte para numérico (o que não for número vira NaN)
        df_clean['faturamento'] = pd.to_numeric(df_clean['faturamento'], errors='coerce')
        # Remove os nulos gerados
        df_clean = df_clean.dropna(subset=['faturamento'])
    return df_clean
```
(1.2) Crie o arquivo `test_pipeline.py` e adicione nele os testes iniciais de propriedades.

**Teste 1 (Invariante de Tamanho):** 

Uma função de limpeza e filtragem pode remover linhas corrompidas, mas ela nunca deve inventar dados novos. Portanto, uma propriedade fundamental é que o tamanho (shape) do DataFrame de saída deve ser sempre menor ou igual ao de entrada, não importa quais valores estranhos entrem.

```python
import pandas as pd
import hypothesis.strategies as st
from hypothesis import given
from pipeline import limpar_faturamento

# TESTE 1: Invariante de Linhas
@given(st.lists(st.floats(allow_nan=True, allow_infinity=True)))
def test_quantidade_de_linhas_nao_aumenta(lista_faturamentos):
    df_original = pd.DataFrame({"faturamento": lista_faturamentos})
    df_resultado = limpar_faturamento(df_original)
    
    assert len(df_resultado) <= len(df_original)
```

**Teste 2 (Invariante de Sanidade/Nulos):** 

O objetivo principal deste passo do pipeline é expurgar dados nulos. A nossa propriedade afirma que, independentemente do tipo de lixo que o Hypothesis gere (NaN, infinitos, etc.), a coluna resultante deve conter exatamente zero valores nulos após o processamento.

```python
# TESTE 2: Invariante de Nulos
@given(st.lists(st.floats(allow_nan=True, allow_infinity=True)))
def test_remocao_de_valores_nulos(lista_faturamentos):
    df_original = pd.DataFrame({"faturamento": lista_faturamentos})
    df_resultado = limpar_faturamento(df_original)
    
    assert df_resultado["faturamento"].isnull().sum() == 0
```
Execute no terminal: `pytest` (Ambos devem passar).

Faça um **COMMIT & PUSH**.

### Passo 2: Descobrindo um Bug com PBT (Valores Negativos)
Regra de negócio: No nosso domínio, faturamento é uma grandeza absoluta (não existe faturamento negativo). Vamos forçar essa propriedade.

**Teste 3 (Invariante de Sinal):** 

Testes baseados em exemplo costumam testar apenas números positivos amigáveis (ex: 100.0, 50.5). O Hypothesis vai varrer o espaço de busca dos números flutuantes. Nossa propriedade dita que nenhum valor final de faturamento pode ser menor que zero.

(2.1) Adicione o TESTE 3 ao arquivo `test_pipeline.py`:
```python
# TESTE 3: Invariante de Sinal
@given(st.lists(st.floats(allow_nan=False, allow_infinity=False)))
def test_faturamento_sempre_positivo(lista_faturamentos):
    df_original = pd.DataFrame({"faturamento": lista_faturamentos})
    df_resultado = limpar_faturamento(df_original)
    
    assert (df_resultado["faturamento"] >= 0).all()
```
Execute no terminal: `pytest`

**VEJA O ERRO:** O teste vai falhar! O Hypothesis encontrou números negativos gerados aleatoriamente e provou que seu código atual deixa passar valores inválidos. Veja no terminal o processo de shrink (encolhimento do erro).

(2.2) Correção do Código: Altere a função `limpar_faturamento` em `pipeline.py` aplicando o método .abs() para corrigir a falha apontada pelo teste:

```python
def limpar_faturamento(df: pd.DataFrame) -> pd.DataFrame:
    df_clean = df.copy()
    if 'faturamento' in df_clean.columns:
        df_clean['faturamento'] = pd.to_numeric(df_clean['faturamento'], errors='coerce')
        df_clean = df_clean.dropna(subset=['faturamento'])
        # CORREÇÃO AQUI: Garante valores absolutos (positivos)
        df_clean['faturamento'] = df_clean['faturamento'].abs()
    return df_clean
```

Rode o `pytest` novamente. Agora o Teste 3 deve passar!

Faça um **COMMIT & PUSH**.

### Passo 3: Adicionando Regras de Strings
Nosso pipeline também precisa limpar os nomes dos clientes, removendo espaços em branco inúteis nas pontas.

Teste 4 (Invariantes de Formatação de Texto): 

Strings vazias, cheias de tabulações ou espaços ocultos quebram agrupamentos no Pandas (fazendo com que `"Google "` e `"Google"` pareçam clientes diferentes). A propriedade garante que qualquer string gerada pelo Hypothesis passará pelo tratamento e sairá sem espaços nas extremidades (ou seja, `nome == nome.strip()`).

(3.1) Adicione o TESTE 4 ao arquivo `test_pipeline.py`:

```python
# TESTE 4: Invariante de String
@given(st.lists(st.text()))
def test_clientes_sem_espacos_laterais(lista_clientes):
    df_original = pd.DataFrame({"cliente": lista_clientes, "faturamento": [100.0] * len(lista_clientes)})
    df_resultado = limpar_faturamento(df_original)
    
    for nome in df_resultado["cliente"]:
        assert nome == nome.strip()
```

Rode `pytest`. Vai falhar porque nosso código ainda não mexe na coluna `cliente`.

(3.2) Atualize o código em `pipeline.py` para processar a coluna cliente:

```python
def limpar_faturamento(df: pd.DataFrame) -> pd.DataFrame:
    df_clean = df.copy()
    
    # NOVA REGRA: Limpa strings da coluna 'cliente'
    if 'cliente' in df_clean.columns:
        df_clean['cliente'] = df_clean['cliente'].astype(str).str.strip()
        df_clean.loc[df_clean['cliente'] == '', 'cliente'] = 'Anonimo'
        
    if 'faturamento' in df_clean.columns:
        df_clean['faturamento'] = pd.to_numeric(df_clean['faturamento'], errors='coerce')
        df_clean = df_clean.dropna(subset=['faturamento'])
        df_clean['faturamento'] = df_clean['faturamento'].abs()
        
    return df_clean
```

Rode `pytest`. Todos os 4 testes devem passar.

Faça um **COMMIT & PUSH**.

### Passo 4: Engenharia de Features (Cálculo de Proporção)

Agora vamos criar uma nova função que calcula a representação percentual (de 0.0 a 1.0) de cada cliente no faturamento total.

(4.1) Nova Função: Adicione ao final do arquivo `pipeline.py`:

```python
def calcular_percentual_faturamento(df: pd.DataFrame) -> pd.DataFrame:
    if df.empty:
        return df
        
    total = df['faturamento'].sum()
    df['percentual'] = df['faturamento'] / total
    return df
```

E também atualize as importações (linha 4) no arquivo `test_pipeline.py` para incluir a nova funcionalidade:

```python
from pipeline import limpar_faturamento, calcular_percentual_faturamento
```
(4.2) Escrevendo as Propriedades Matemáticas:

**Teste 5 (Limites Estatísticos):** 

Uma proporção matemática de uma parte sobre o todo possui uma regra inviolável: ela deve estar contida estritamente no intervalo $[0.0, 1.0]$. Se o Hypothesis achar qualquer cenário onde o percentual seja negativo ou maior que 100% (1.0), nosso cálculo falhou.

```python
# TESTE 5: Invariante de Escopo
@given(st.lists(st.floats(min_value=0.0, max_value=100000.0)))
def test_limites_do_percentual(lista_faturamentos):
    df_original = pd.DataFrame({"faturamento": lista_faturamentos})
    df_resultado = calcular_percentual_faturamento(df_original)
    
    if not df_resultado.empty:
        assert (df_resultado['percentual'] >= 0.0).all()
        assert (df_resultado['percentual'] <= 1.0).all()
```
**Teste 6 (Invariante da Soma / Fechamento):** 

Se somarmos todas as fatias de um gráfico de pizza, o resultado deve ser sempre 100% (ou 1.0). Essa propriedade valida se a distribuição está correta. Nota: Usamos uma tolerância matemática (1e-5) porque computadores têm pequenas imprecisões ao somar números com pontos decimais (ponto flutuante).

```python
# TESTE 6: Invariante da Soma
@given(st.lists(st.floats(min_value=0.0, max_value=100000.0)))
def test_soma_das_proporcoes_eh_um(lista_faturamentos):
    df_original = pd.DataFrame({"faturamento": lista_faturamentos})
    df_resultado = calcular_percentual_faturamento(df_original)
    
    if not df_resultado.empty and df_resultado['faturamento'].sum() > 0:
        soma_percentuais = df_resultado['percentual'].sum()
        assert abs(soma_percentuais - 1.0) < 1e-5
```

Rode `pytest`.

**VEJA O ERRO:** O Hypothesis foi maldoso e gerou uma lista cheia de zeros: [0.0, 0.0]. Como a soma total é zero, seu código tentou dividir por zero (0.0 / 0.0) e o Pandas estourou um erro ou gerou valores inválidos!

(4.3) Tratando o Bug de Divisão por Zero: Corrija a função em `pipeline.py` para tratar esse caso extremo descoberto pelo framework.

```python
def calcular_percentual_faturamento(df: pd.DataFrame) -> pd.DataFrame:
    if df.empty:
        return df
        
    total = df['faturamento'].sum()
    
    # CORREÇÃO: Trata o caso onde a soma de faturamentos é zero
    if total == 0:
        df['percentual'] = 0.0
    else:
        df['percentual'] = df['faturamento'] / total
        
    return df
```

Rode `pytest`. Agora tudo passou!

Faça um **COMMIT & PUSH**.

### Passo 5: Contratos Arquiteturais (Idempotência e Casos Limite)
Para concluir nossos testes, vamos garantir estabilidade contra execuções duplicadas e tabelas vazias.

**Teste 7 (Idempotência):** 

Um pipeline de dados seguro deve ser idempotente. Isso significa que aplicar a função de cálculo uma vez, ou aplicar a mesma função de novo em cima do resultado já calculado, não deve alterar os dados ou corromper as colunas. O estado deve se estabilizar.

```python
# TESTE 7: Invariante de Idempotência
@given(st.lists(st.floats(min_value=0.0, max_value=50000.0)))
def test_idempotencia_do_calculo(lista_faturamentos):
    df_original = pd.DataFrame({"faturamento": lista_faturamentos})
    
    primeira_passada = calcular_percentual_faturamento(df_original)
    segunda_passada = calcular_percentual_faturamento(primeira_passada.copy())
    
    # Verifica se a tabela estabilizou e continua idêntica
    pd.testing.assert_frame_equal(primeira_passada, segunda_passada)
```

**Teste 8 (Tratamento de Bordas / Dataframe Vazio):** 

Um erro clássico em produção é o pipeline quebrar quando o banco de dados não retorna nenhuma linha no dia. A propriedade garante o contrato de estabilidade: entrar uma tabela vazia deve retornar uma tabela vazia, sem lançar exceções de execução.

```python
# TESTE 8: Caso Limite (DataFrame Vazio)
def test_com_dataframe_vazio():
    df_vazio = pd.DataFrame(columns=['faturamento'])
    df_resultado = calcular_percentual_faturamento(df_vazio)
    
    assert df_resultado.empty
```

Execute a bateria completa de testes para confirmar que todos passaram com sucesso:
```
pytest
```

Faça o último **COMMIT & PUSH**!

