# 01 · Arquitetura

## Visão geral

O projeto é dividido em **três apps Qlik**, um por camada. Cada camada grava QVDs que a seguinte lê. Assim, um problema pode ser isolado na etapa em que aconteceu.

```mermaid
flowchart TB
    subgraph Fonte
        CSV[especime.csv<br/>Portal de Dados Abertos IBAMA<br/>38.432 linhas · 20 campos]
    end
    subgraph "01 · Extração"
        E1[Lê os 20 campos como TEXTO]
        E2[Acrescenta DT_EXTRACAO]
        E3[(extracao/Extracao_Especime.qvd)]
        E1 --> E2 --> E3
    end
    subgraph "02 · Transformação"
        T1[Funções: vTexto, vNumero, vDataHora]
        T2[Mapas: condição, Sim/Não, unidade]
        T3[Temp_Especime<br/>tipos + padronização + chaves]
        T4[Checagem de qualidade no log]
        T5[Dim_Especie · Dim_Auto · Fato_Especime · Dim_Calendario · Info_Fonte]
        T6[(carga/especime/*.qvd)]
        T1 --> T3
        T2 --> T3
        T3 --> T4 --> T5 --> T6
    end
    subgraph "03 · Dashboard"
        D1[LOAD * dos 5 QVDs]
        D2[8 medidas mestre · 8 dimensões mestre]
        D3[3 pastas]
        D1 --> D2 --> D3
    end
    CSV --> E1
    E3 --> T3
    T6 --> D1
```

## Responsabilidade de cada camada

### 01 · Extração
- **Faz:** lê o CSV (`utf8`, separador `;`, cabeçalho na 1ª linha), converte os 20 campos com `Text()` e grava `Extracao_Especime.qvd`.
- **Não faz:** nenhuma limpeza, renomeação ou regra de negócio.
- **Por quê:** o QVD da extração é uma **cópia fiel da fonte** e serve de referência para auditoria. Se um número divergir no painel, dá para saber se o problema veio da fonte ou da transformação.
- **Modo de carga:** sempre FULL (ver [ADR-001](04-decisoes-tecnicas.md#adr-001--extração-sempre-full)).

### 02 · Transformação
Ordem de execução:

| Aba | O que faz |
|---|---|
| Parâmetros | Caminho do QVD de extração e da pasta de carga |
| Funções | `vTexto` (vazio → "Não informado"), `vNumero` (ponto decimal → número), `vDataHora` (texto ISO → data/hora) |
| Mapas | `Map_Condicao` (vivo/viva/morto/morta), `Map_SimNao` (S/N), `Map_Unidade` (m³/M³) |
| Temp_Especime | Conversão de tipos e padronização (1º LOAD); chaves e campos derivados (preceding load) |
| Qualidade | Conta linhas × IDs distintos, falhas de conversão e datas futuras, e escreve no log |
| Dim_Especie / Dim_Auto / Fato_Especime / Dim_Calendario | Modelo estrela |
| Info_Fonte | Tabela de 1 linha com a data da fonte e da extração, mostrada no painel |
| STORE | Grava os 5 QVDs em `carga/especime/` |

Saída do log na última execução:
```
Linhas: 38432 · IDs distintos: 38432 (devem ser iguais)
Falhas de conversão -> quantidade: 0 · valor: 0 · data: 0
Datas futuras descartadas: 2
Fato_Especime: 38432 · Dim_Especie: 5254 · Dim_Auto: 15153 · Dim_Calendario: 1847
```

### 03 · Dashboard
- O script só faz `LOAD *` dos QVDs, **sem nenhuma regra**.
- Toda a lógica de indicador fica nas **medidas mestre**: uma única definição, usada em todos os objetos.
- Pastas, gráficos e estilo estão descritos em [05-painel-e-medidas.md](05-painel-e-medidas.md).

## Ordem de recarga

```
Extração  →  Transformação  →  Dashboard
```
Cada app depende do QVD gerado pelo anterior. O próximo passo recomendado é encadear as três recargas numa automação do Qlik Cloud.

## Convenções
- Campos de medida com prefixo `#` (`#Quantidade`, `#Valor Atribuído`).
- Chaves técnicas com prefixo `%` (`%EspecieKey`, `%AutoKey`), para ficarem separadas dos campos de negócio.
- Nomes de campos em português, prontos para o usuário final.
- Textos ausentes padronizados como **"Não informado"**.
