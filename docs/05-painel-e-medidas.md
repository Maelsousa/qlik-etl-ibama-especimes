# 05 · Painel e medidas

## Medidas mestre

| Medida | Fórmula | Valor atual |
|---|---|---|
| Espécimes apreendidos | `Count([ID Espécime])` | 38.432 |
| Autos de infração | `Count(DISTINCT [Nº do Auto] & '\|' & [Série do Auto])` | 15.129 |
| Espécies diferentes | `Count(DISTINCT {<[Nome Científico]-={'Não informado','Não identificado'}>} [Nome Científico])` | 1.760 |
| Animais apreendidos (unid.) | `Sum({<[Unidade de Medida]={'Unidade'}>} [#Quantidade])` | 728.993 |
| % marcados como CITES | `Count({<[Ameaçada (CITES)]={'Sim'}>} [ID Espécime]) / Count([ID Espécime])` | 9,8% |
| % encontrados vivos | `Count({<[Condição]={'Vivo'}>} [ID Espécime]) / Count({<[Condição]={'Vivo','Morto'}>} [ID Espécime])` | 75,0% |
| Valor Atribuído | `Sum([#Valor Atribuído])` (só Sabiá/Aiemob) | R$ 57,4 mi |
| Quantidade | `Sum([#Quantidade])`, sempre usada junto com a dimensão Unidade de Medida | — |

Dimensões mestre: Tipo de Infração, Grupo, Nome Popular, Nome Científico, Ano, Condição, Ameaçada (CITES), Unidade de Medida.

## Pastas

### 1. Visão Geral
| Objeto | Pergunta (título) | Conclusão dinâmica (subtítulo) |
|---|---|---|
| 6 KPIs | — | — |
| Área por ano | Como as apreensões evoluíram ao longo dos anos? | "Pico em 2015, com 6.779 espécimes registrados" |
| Barras horizontais | O que mais se apreende? | "Ave lidera com 70% dos espécimes com grupo informado" |
| Rosca | Em que estado os animais foram encontrados? | "75% estavam vivos (entre os casos com condição informada)" |
| Barras horizontais | Fauna, flora ou pesca? | "Fauna responde por 76% dos espécimes" |

### 2. Espécies
| Objeto | Pergunta | Conclusão |
|---|---|---|
| Filtros | Tipo, Grupo, CITES, Condição | — |
| Top 10 | Quais espécies são mais apreendidas? | "Trinca-ferro-verdadeiro é a espécie mais apreendida, com 1.832 espécimes" |
| Top 10 CITES | Quais espécies marcadas como CITES aparecem mais? | "3.757 espécimes marcados como CITES pela fiscalização" |
| Tabela | Todas as espécies | "Clique em uma linha para filtrar o painel inteiro" |

### 3. Detalhe
| Objeto | Pergunta |
|---|---|
| Filtros | Ano, Sistema de Origem, Padrão do Auto, Unidade de Medida, Tipo |
| Tabela | Quanto foi apreendido, por unidade de medida? (sem linha de total, de propósito) |
| Barras | Como os autos são numerados? (Sicafi × AIe, mudança em 08/10/2019) |
| Tabela | Lista de espécimes por auto de infração |

## Exemplo de conclusão dinâmica

```qlik
='Pico em ' & FirstSortedValue(Ano, -Aggr(Count([ID Espécime]), Ano))
 & ', com ' & Num(Max(Aggr(Count([ID Espécime]), Ano)), '#.##0')
 & ' espécimes registrados'
```

Rodapé que informa o que foi excluído do gráfico:
```qlik
='Não exibidos: ' & Num(Count({<Grupo={'Não informado'}>} [ID Espécime]), '#.##0')
 & ' espécimes sem grupo informado na fonte.'
```

## Identidade visual
- **Fundo:** imagem PNG por pasta (degradê azul-marinho `#0E2338` → `#1B4468`, ícone de folha, título e subtítulo), na biblioteca de mídia do app e esticada para a pasta.
- **Paleta:** azul `#4477AA` (a mesma dos KPIs), azul-escuro `#2C4F73` e terracota `#C0623B` para o tema CITES.
- **Tipografia:** títulos de 15px em negrito `#1F2A36`; eixos e rodapés em `#6B7785`.
- **Regras:** nada de "Não informado" nos rankings; quantidades nunca somadas entre unidades; fonte e data dos dados no rodapé.

## Lições do desenvolvimento visual
- Objetos criados pela API do Qlik **precisam das propriedades visuais padrão** (cor, legenda, eixos), que a interface preenche sozinha. Sem elas, o gráfico fica em branco.
- **Um tema com ID inválido quebra o desenho de todos os gráficos.**
- O fundo da pasta fica em `components → general` (`bgImage.mode = 'media'`, `sizing = 'stretch'`).
- O Qlik não desenha gráficos (canvas) em abas ocultas do navegador. Para validar o visual, a janela precisa estar visível.
