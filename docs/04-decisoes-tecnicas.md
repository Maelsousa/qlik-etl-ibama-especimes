# 04 · Decisões técnicas (ADR)

Cada decisão segue o formato: **contexto → decisão → consequências**.

---

## ADR-001 · Extração sempre FULL
**Contexto.** O IBAMA republica a base inteira todos os dias. A única data disponível (`DAT_ALTERACAO`) tem nulos e até datas em 2045, então não serve como marca d'água confiável para um incremental. O volume é de cerca de 38 mil linhas.
**Decisão.** A extração lê o CSV completo a cada execução.
**Consequências.** É simples e sempre consistente com a fonte, e roda em segundos. Se o volume crescer muito, a decisão deve ser revista.

## ADR-002 · Extração como cópia fiel, com todos os campos em texto
**Contexto.** O CSV usa ponto decimal ("1.0"), e o app usa a configuração pt-BR, em que o ponto é separador de milhar. Deixando o Qlik interpretar sozinho, "1.0" pode virar **10**.
**Decisão.** A extração aplica `Text()` em todos os campos. A conversão é feita na Transformação com `Num#(x, '', '.', ',')` e `Timestamp#(x, 'YYYY-MM-DD hh:mm:ss')`.
**Consequências.** Nenhum número é reinterpretado sem controle, e o QVD da extração serve de referência para auditoria. A Transformação conta as falhas de conversão (hoje são zero).

## ADR-003 · Três apps em camadas
**Contexto.** Misturar leitura, regra e visual num app só dificulta o diagnóstico e a reutilização.
**Decisão.** Extração → Transformação → Dashboard, cada um gravando ou lendo QVDs.
**Consequências.** O dashboard não tem regra de negócio, e os QVDs da camada de carga podem alimentar outros painéis. A contrapartida é que as recargas precisam rodar na ordem certa.

## ADR-004 · Chave do espécime por sistema de origem
**Contexto.** O Sicafi/Sifisc identifica o espécime por `SEQ_ESPECIME` e o Sabiá/Aiemob por `CD_ESPECIME`; nunca os dois na mesma linha.
**Decisão.** `ID Espécime = 'SICAFI-' & SEQ_ESPECIME` ou `'SABIA-' & CD_ESPECIME`, e o campo `Sistema de Origem` registra de qual sistema veio.
**Consequências.** São 38.432 IDs únicos para 38.432 linhas, conferido no log. Também é possível separar análises por sistema, o que é útil porque o valor só existe no Sabiá.

## ADR-005 · Chaves de dimensão com Hash128
**Contexto.** `SEQ_TAXONOMIA` está quase sempre vazia, e o mesmo nome científico aparece com atributos diferentes.
**Decisão.** `%EspecieKey = Hash128(nome científico, nome popular, grupo, tipo, CITES)`, calculada num *preceding load* sobre os campos já tratados.
**Consequências.** Garante uma linha de dimensão por combinação real de atributos e evita chaves sintéticas. A mesma espécie pode ter mais de uma linha na dimensão quando os atributos variam na fonte.

## ADR-006 · "Não informado" em vez de nulo
**Contexto.** Nulos somem de filtros e gráficos sem aviso nenhum.
**Decisão.** Textos vazios viram "Não informado" (função `vTexto`). Nos rankings eles são excluídos por set analysis, e o rodapé informa quantos ficaram de fora.
**Consequências.** O usuário vê o tamanho da lacuna sem que ela distorça os rankings.

## ADR-007 · Medidas mestre como fonte única das fórmulas
**Decisão.** Todo KPI e todo gráfico usa uma medida mestre com descrição. Gráficos que excluem "Não informado" usam set analysis declarado no próprio objeto.
**Consequências.** Uma correção (como a da contagem de autos) se propaga para o painel inteiro.

## ADR-008 · Visual por imagem de fundo
**Contexto.** O usuário do tenant não tem permissão para instalar temas (a API respondeu 403), e o KPI nativo não aceita cor personalizada pela API.
**Decisão.** Um PNG por pasta (fundo, cabeçalho e título), gerado em canvas e enviado à **biblioteca de mídia do app**. Os objetos ficam sobre o fundo, como cartões brancos.
**Consequências.** Dá um visual profissional sem permissão de administrador. A imagem é esticada para o tamanho da tela e, em proporções muito diferentes, o cabeçalho pode distorcer.

## ADR-009 · Títulos em forma de pergunta e conclusões dinâmicas
**Decisão.** Cada gráfico tem como título a pergunta que responde. O subtítulo é uma expressão que escreve a conclusão (ex.: `'Pico em ' & FirstSortedValue(Ano, -Aggr(Count([ID Espécime]), Ano)) & …`).
**Consequências.** O painel pode ser lido sem treinamento, e as frases continuam corretas quando o usuário filtra.

## ADR-010 · Pastas com tamanho fixo (1500 × 860 px)
**Contexto.** Em telas de notebook (≈1.536 × 640 px úteis), o modo responsivo espremia os 10 objetos da Visão Geral, e o Qlik escondia rótulos e eixos dos gráficos menores.
**Decisão.** `layoutOptions.sheetMode = 'CUSTOM'`, com 1500 × 860 px nas três pastas: a largura cabe sem rolagem horizontal e a altura extra ganha rolagem vertical.
**Consequências.** Todos os gráficos mostram rótulos e eixos. Em monitores grandes, a pasta não ocupa a largura toda.
