# 03 · Qualidade e validação dos dados

## 1. Perfilamento da fonte (antes do ETL)

| Campo | Achado | Tratamento |
|---|---|---|
| CARACTERISTICA | Deveria ser vivo/morto, mas tem 126 valores: "Vivo", "Viva", "Morto", "Morta", textos livres ("Óleo", "madeiras em toras", "300 reais o m3"…) e 3.557 vazios | Padronizado em **Vivo / Morto / Outro (texto livre) / Não informado**; o texto original foi mantido em outro campo |
| UNIDADE_MEDIDA | "m³" e "M³" contados como unidades diferentes | Padronizado para **m³** |
| CD_SERIE_AUTO_INFRACAO | "a" e "A" | Convertido para maiúsculas |
| QUANTIDADE / NU_VALOR | Ponto decimal ("170010.0"), que o Qlik em pt-BR leria como milhar | Extração em texto; conversão explícita com `Num#(x, '', '.', ',')` |
| QUANTIDADE | 290 vazios | Mantidos nulos |
| NU_VALOR | Preenchido só no Sabiá/Aiemob (3.763 registros) | Indicador de valor com ressalva no painel |
| DAT_ALTERACAO | 5.221 vazios e 2 em **2045** | Datas futuras viram inválidas; campo `Qualidade da Data` |
| GRUPO / TIPO | 8.143 e 1.219 vazios | "Não informado", fora dos rankings e com o total informado no rodapé |
| SEQ_TAXONOMIA | Preenchido só em 1.013 linhas | Não usado como chave |
| SITUACAO | 94% vazio, o resto em texto livre | Só na tabela de detalhe |
| Chave do espécime | Sicafi usa `SEQ_ESPECIME`; Sabiá usa `CD_ESPECIME` (nunca os dois juntos) | `ID Espécime` = sistema + código |

## 2. Checagens automáticas na Transformação

A aba **Qualidade** escreve no log a cada execução:

| Checagem | Esperado | Última execução |
|---|---|---|
| Linhas = IDs distintos | iguais | 38.432 = 38.432 ✔ |
| Quantidades não convertidas | 0 | 0 ✔ |
| Valores não convertidos | 0 | 0 ✔ |
| Datas não convertidas | 0 | 0 ✔ |
| Datas futuras | informativo | 2 |

## 3. Reconciliação independente: painel × CSV bruto

O painel foi comparado com um app separado que carrega o CSV **direto**, sem passar pelo ETL. Os mesmos indicadores foram calculados nos dois:

| Indicador | CSV bruto | Painel | Resultado |
|---|---|---|---|
| Total de registros | 38.432 | 38.432 | ✔ |
| CITES = S | 3.757 | 3.757 | ✔ |
| Tipo = Fauna | 28.290 | 28.290 | ✔ |
| Grupo = Ave | 21.331 | 21.331 | ✔ |
| Vivo + Viva | 25.999 | 25.999 | ✔ |
| Morto + Morta | 8.649 | 8.649 | ✔ |
| Trinca-ferro-verdadeiro | 1.832 | 1.832 | ✔ |
| Soma das quantidades (Unidade) | 728.992,591 | 728.992,591 | ✔ |
| Soma do valor | 57.396.722,93 | 57.396.722,93 | ✔ |
| Registros por ano 2012 a 2019 | 1 · 136 · 6.666 · 6.779 · 6.276 · 4.520 · 5.154 · 3.677 | idênticos | ✔ |
| Sem data | 5.221 + 2 (2045) | 5.223 | ✔ (as datas futuras foram tratadas como inválidas, de propósito) |
| Números de auto distintos | 15.130 | 15.129 | ✔ explicado: a padronização de maiúsculas e espaços juntou duas grafias do mesmo número |

**Conclusão:** o ETL não perdeu, não duplicou e não alterou nenhum dado.

## 4. Problemas de interpretação encontrados na revisão (e corrigidos)

### 4.1 "Não identificado" aparecia como espécie
- 507 registros têm nome popular **e** científico escritos literalmente "Não identificado", e 181 estão marcados como CITES.
- Eles apareciam em 2º lugar no ranking de espécies CITES e contavam como 1 espécie no KPI.
- **Correção:** foram excluídos dos rankings e do KPI de espécies (1.761 → **1.760**). O rodapé informa: "Não exibidos: 2.842 espécimes sem espécie identificada ou informada".

### 4.2 A marcação CITES é inconsistente na fonte
- **447 das 1.760 espécies** têm registros marcados como CITES e outros não.
- Papagaio-verdadeiro (*Amazona aestiva*), espécie da lista CITES: 311 "Sim" e **236 "Não"**.
- Trinca-ferro (*Saltator similis*), que não é CITES: 1.728 "Não" e **32 "Sim"**.
- Sabiá/Aiemob marca 35% dos registros; Sicafi/Sifisc marca 7%.
- **Correção:** o KPI foi renomeado de "% espécies ameaçadas" para **"% marcados como CITES"**, com uma nota no gráfico.
- **Próximo passo:** usar a lista oficial CITES (Species+) por nome científico.

### 4.3 Autos contados em dobro
- O mesmo número de auto aparecia com dois códigos internos (`SEQ_AUTO_INFRACAO`), e o KPI mostrava 15.153.
- **Correção:** a medida passou a contar número + série = **15.129**.

### 4.4 Soma de unidades diferentes
- Uma tabela somava m³, gramas e kg num único total (78 milhões), o que não tem significado.
- **Correção:** o total foi removido e o rodapé explica que as quantidades não se somam entre unidades.
- Também há valores extremos na fonte (ex.: um auto com 19.199.628 m³), mantidos como vieram, mas visíveis na pasta de detalhe.

## 5. Lições
- **Validação técnica não basta.** Contagens e falhas de conversão zeradas mostram que o ETL está correto, mas não mostram que o indicador significa o que o nome diz. Os itens da seção 4 só apareceram na revisão de significado.
- **Reconciliar com uma fonte independente** (um app que lê o CSV bruto) é a forma mais simples de provar que o ETL não mudou nenhum número.
