# Extrator de Exames — Especificação de Design

Data: 01/11/2026
Status: aguardando revisão

## 1. Objetivo

Transformar o texto copiado (Ctrl+C) de um laudo laboratorial em PDF numa linha única e compacta, pronta para colar na evolução do prontuário eletrônico (Soul MV):

> 05/01 08:00: Hb **9,1** g/dL, Ht **27,4** %, Leuco 12.300 /µL (N **86,0**% / L **8,0**% / M 5,0% / E 1,0% / B 0,0%), Plaq 210.000 /µL, Ur **72** mg/dL, Cr **1,45** mg/dL, ...

(Valores fictícios. Nenhum laudo real é versionado neste repositório.)

Usuário: médico em UTI, colando resultados do laboratório DASA (layout validado com três laudos reais, mantidos fora do repositório).

## 2. Fluxo de uso

1. O usuário abre `extrator-de-exames.html` no navegador (arquivo local, sem servidor).
2. Cola o texto do laudo na caixa de entrada. O processamento é automático ao colar ou editar.
3. A linha formatada aparece na área de saída, com valores alterados em negrito.
4. A caixa "Mostrar unidades" liga/desliga as unidades.
5. O botão "Copiar" copia a linha para a área de transferência, em HTML (com negrito) e em texto simples.
6. O usuário cola no Soul MV.

## 3. Requisitos de saída

- **Uma linha por data/hora de coleta**, da mais recente para a mais antiga, iniciada por `dd/mm hh:mm:`. A data vem do cabeçalho `Data de Coleta/Recebimento: dd/mm/aaaa Hora Aproximada: hh:mm`.
- **Separador** entre exames: `, `.
- **Ordem fixa por grupo**, independentemente da ordem no laudo:
  hemograma → coagulograma → função renal → eletrólitos → hepático → outros cadastrados → gasometria → lactato → exames não cadastrados.
- **Abreviações** vindas do dicionário (seção 6), editáveis pelo usuário.
- **Unidade** logo após o valor, como aparece no laudo; removível pela caixa de seleção. No diferencial do leucograma o `%` é mantido mesmo sem unidades.
- **Números** exatamente como no laudo (vírgula decimal, ponto de milhar: `9.800`, `210.000`).
- **Negrito** em todo valor fora da faixa de referência impressa no laudo (regras na seção 5.4).
- **Grupos compostos:**
  - Leucograma: `Leuco 9.800 /µL (N 78,0% / L 12,0% / M 6,0% / E 3,0% / B 1,0%)`. O diferencial usa o percentual; o negrito do diferencial segue a faixa percentual.
  - TP: `TP 13,40 s (AP 88,00% / RNI 1,08)`.
  - TTPA: `TTPA 31,20 s (R 0,95)`.
  - Gasometria: prefixo `Gaso V:` (venosa) ou `Gaso A:` (arterial) antes do primeiro parâmetro gasométrico; parâmetros separados por `, `.
- **Exame repetido na mesma data:** vale o resultado fora de seção de gasometria (laboratório central). Se ambos estiverem na mesma categoria, vale o primeiro. Eletrólitos, Hb e Ht da gasometria só aparecem se não houver o mesmo exame do laboratório central naquela data.
- **Exame reconhecido mas não cadastrado no dicionário:** aparece no fim da linha com o nome original do laudo, para nada sumir.

## 4. Privacidade e armazenamento

- Nenhuma requisição de rede. O HTML final declara uma Content-Security-Policy com `connect-src 'none'` e sem fontes externas, de modo que o próprio navegador bloqueia qualquer envio.
- Linhas de identificação (`Cliente:`, `Data de Nascimento:`, `Médico:`, `Prontuário:`, `Atendimento:`, `Local:`, `FAP`, `CIP`) são descartadas na leitura.
- **Aviso de múltiplos pacientes:** os números de `Prontuário:` são comparados apenas em memória; se houver mais de um, aparece um aviso. Esses números não são exibidos nem gravados.
- `localStorage` guarda:
  - configurações (abreviações editadas, exames visíveis, opções de saída);
  - o último texto colado, **já sem as linhas de identificação**, para sobreviver a um recarregamento acidental.
- Botão "Limpar" apaga a entrada, a saída e o texto armazenado. Não há histórico de pacientes.
- Laudos reais usados no desenvolvimento ficam em `exemplos/`, pasta ignorada pelo git.

## 5. Regras do extrator

O texto é lido linha a linha por uma máquina de estados que mantém: data de coleta atual, seção atual (ex.: Tempo de Protrombina, TTPA, Gasometria Venosa), nome pendente e resultado pendente.

### 5.1 Linhas descartadas

- Cabeçalho de página: `Exame Resultado Valor de Referência`.
- Rodapé de página: identificação do paciente (seção 4), `Data da Geração`, `Sob a responsabilidade`, `Laboratório registrado`, `-`, linhas só com dígitos/pontos, `PG n de m`.
- `Assinado eletronicamente…`, hashes hexadecimais de 64 caracteres, `Responsável:`, `Locais de execução…` e as linhas `- DASA…` que o seguem.
- Linhas entre parênteses: `(Material: …)`, `(Método: …)`, subtítulos como `(Aspartato amino transferase)`; parênteses abertos continuam sendo descartados até fechar na linha seguinte.
- Bloco de histórico: de `Histórico …` até o fim das linhas `Data …`, datas, horas e `Resultado …`.
- `% /µL` (cabeçalho do leucograma), `Gráfico de Histórico`, `Alteração do Intervalo de Referência…`.
- Blocos de texto (`Nota:`, `Observações:`, `Referência…`, `Fontes:`): continuam sendo lidos com as regras estritas (um resultado real pode aparecer logo depois, como Hb/Ht da gasometria), mas linhas não reconhecidas dentro deles não vão para o painel de não reconhecidos.

### 5.2 Formatos de resultado aceitos

Uma linha só vira resultado se casar integralmente com um destes formatos:

| Formato | Exemplo (layout DASA, valores inventados) |
|---|---|
| `NOME VALOR [UNIDADE] REFERÊNCIA` | `Hemoglobina 10,2 g/dL 12,0 a 15,0 g/dL` |
| Leucograma: `NOME % ABSOLUTO REF% REFABS` | `Neutrófilos 78,0 7.644 40,0 a 80,0 % 1.800 a 7.800 /µL` |
| Nome pendente + `VALOR [UNIDADE] REFERÊNCIA` | `Transaminase pirúvica - TGP` / `(Alanina…)` / `58 U/L Inferior a 33 U/L` |
| `NOME VALOR [UNIDADE]` + linha seguinte só com `REFERÊNCIA` | `Tempo em segundos 31,20 s` / `27,80 a 38,60 s` |

- **VALOR** é o primeiro token puramente numérico (aceita sinal e `<`/`>`); **NOME** é o que vem antes; **UNIDADE** é o que fica entre o valor e a referência (pode conter dígitos: `mL/min/1,73m²`, `10^6/µL`).
- **Nome pendente:** a última linha sem número isolado, curta (até 8 palavras) e que não termina em ponto final. Se não casar com o dicionário, tenta-se a junção com a linha anterior (caso `Razão Normatizada` / `Internacional (RNI):`). Expira após duas linhas não descartadas.
- Prefixos `*` e sufixos `:` do nome são ignorados.

### 5.3 Formatos de referência

- `X a Y`, `X até Y`, `de X até Y` (com ou sem unidade, inclusive colada: `100,00%`), inclusive valores negativos/positivos (`-3,0 a +3,0`).
- `Inferior a X` / `Menor que X`; `Superior a X` / `Maior que X`.
- `Não se aplica` (sem avaliação de negrito).
- `100 %` na linha do total de leucócitos (ignorado; vale a faixa absoluta).

### 5.4 Fora da faixa (negrito)

- `X a Y`: alterado se valor < X ou valor > Y. Valores iguais ao limite são normais.
- `Inferior a X`: alterado se valor ≥ X (decisão do usuário: valor igual a X é alterado).
- `Superior a X`: alterado se valor ≤ X (mesma leitura literal).
- Valor com `<`/`>` ou referência não numérica: sem negrito.
- O negrito é mecânico em relação ao laudo; não há interpretação clínica.

## 6. Dicionário padrão

Cada entrada tem: nomes aceitos (comparação sem acentos, sem maiúsculas, também por partes separadas por ` - ` ou entre parênteses), abreviação, grupo, ordem, visível por padrão e, quando necessário, contexto de seção ou unidade.

| Nome no laudo | Abrev. | Grupo | Visível |
|---|---|---|---|
| Eritrócitos | Hm | hemograma | não |
| Hemoglobina | Hb | hemograma | sim |
| Hematócrito | Ht | hemograma | sim |
| VCM, HCM, CHCM, RDW | (iguais) | hemograma | não |
| Leucócitos | Leuco | hemograma | sim |
| Neutrófilos / Bastonetes / Segmentados | N / Bast / Seg | hemograma (diferencial) | sim |
| Linfócitos / Monócitos / Eosinófilos / Basófilos | L / M / E / B | hemograma (diferencial) | sim |
| Contagem de Plaquetas / Plaquetas | Plaq | hemograma | sim |
| VPM | VPM | hemograma | não |
| Tempo em segundos (seção TP) | TP | coagulograma | sim |
| Atividade de Protrombina | AP | coagulograma | sim |
| RNI | RNI | coagulograma | sim |
| Tempo em segundos (seção TTPA) | TTPA | coagulograma | sim |
| Relação (seção TTPA) | R | coagulograma | sim |
| Ureia | Ur | renal | sim |
| Creatinina | Cr | renal | sim |
| eGFR | TFGe | renal | sim |
| Sódio / Potássio / Cloro / Magnésio / Cálcio Iônico / Cálcio / Fósforo | Na / K / Cl / Mg / Cai / Ca / P | eletrólitos | sim |
| TGO / TGP / Gama-Glutamil Transferase / Fosfatase Alcalina | TGO / TGP / GGT / FA | hepático | sim |
| Bilirrubina Total / Direta / Indireta, Albumina | BT / BD / BI / Alb | hepático | sim |
| PCR, Glicose | PCR / Glic | outros | sim |
| pH, pCO2, HCO3, Excesso de Bases, Saturação de O2 | pH / pCO2 / HCO3 / BE / SatO2 | gasometria | sim |
| pO2 | pO2 | gasometria | só arterial |
| Conteúdo de CO2 | ctCO2 | gasometria | não |
| Ácido Láctico (mmol/L) | Lac | lactato | sim |
| Ácido Láctico (mg/dL) | Lac | lactato | não |

Entradas não vistas nos laudos de exemplo (Bastonetes, Segmentados, Cloro, Cálcio, Fósforo, bilirrubinas, Albumina, PCR, Glicose, gasometria arterial) são melhor esforço e devem ser validadas quando surgir um laudo real.

## 7. Interface

Página única, em português:

- **Entrada:** área de texto grande. Colar dispara o processamento.
- **Saída:** prévia formatada (negrito real), uma linha por data.
- **Controles:** "Mostrar unidades" (caixa de seleção), "Copiar", "Limpar" e "Marcar alterados no texto simples" (nenhum / `*` / `↑↓`; padrão: nenhum). A marcação no texto simples serve para o caso de o Soul MV descartar a formatação ao colar.
- **Avisos:** múltiplos pacientes; nenhuma data de coleta encontrada.
- **Painel "Não reconhecido"** (recolhível, com contador): linhas com número isolado que não foram descartadas nem reconhecidas, fora de blocos de texto.
- **Configurações** (recolhível): tabela do dicionário com abreviação editável e caixa "visível" por exame; botão "Restaurar padrão".

Cópia: um evento `copy` grava `text/html` (com `<b>`) e `text/plain` simultaneamente. Esse método funciona em `file://`, sem pedir permissão ao navegador.

## 8. Arquitetura

Sem dependências externas e sem etapa obrigatória de build para desenvolver.

```
index.html              página (desenvolvimento), carrega os scripts abaixo
src/estilos.css
src/dicionario.js       dicionário padrão + casamento de nomes
src/extrator.js         texto → lista de resultados {data, secao, nome, idExame, valor, unidade, ref, alterado, direcao}
src/formatador.js       resultados + opções → linhas [{texto, alterado}] → HTML e texto simples
src/armazenamento.js    leitura/gravação de configurações e último texto (localStorage)
src/app.js              ligação com a interface
build.js                gera dist/extrator-de-exames.html com tudo embutido
tests/*.test.js         testes (node --test)
tests/fixtures/*.txt    laudos sintéticos no layout DASA, com valores inventados
exemplos/               laudos reais locais (ignorados pelo git)
```

- `extrator.js`, `dicionario.js` e `formatador.js` são funções puras, usáveis no navegador (scripts clássicos) e no Node (`module.exports`), o que permite testá-las sem navegador.
- O entregável para o usuário é o arquivo único `dist/extrator-de-exames.html`, que pode ser copiado para qualquer computador e aberto com duplo clique.

## 9. Tratamento de erros

- Texto vazio: saída vazia, sem aviso.
- Texto sem nenhum resultado reconhecido: mensagem "Nenhum resultado reconhecido" e o painel de não reconhecidos aberto.
- Resultado sem data de coleta (texto colado parcialmente): linha sem prefixo de data e aviso.
- `localStorage` indisponível: o app funciona normalmente, sem persistência, com aviso discreto.
- Falha ao copiar: mensagem pedindo para selecionar a prévia e copiar manualmente.

## 10. Testes

- `node --test`, sem dependências.
- Testes unitários de referência (cada formato da seção 5.3), de negrito (limites inclusivos, "Inferior a" literal), de casamento de nomes (incluindo armadilhas como "Hemoglobina glicada" e "Creatinina urinária", que não podem virar Hb/Cr).
- Testes de ponta a ponta com fixtures sintéticas que reproduzem as armadilhas observadas: rodapés repetidos, blocos de histórico, notas com números, leucograma em duas colunas, nome e valor em linhas separadas, valor e referência em linhas separadas, "Tempo em segundos" em TP e TTPA, gasometria com eletrólitos repetidos, lactato em duas unidades, dois laudos com datas diferentes, dois pacientes.
- Teste local opcional que roda os laudos de `exemplos/` (se existirem) e compara com saídas esperadas guardadas na mesma pasta ignorada.
- Verificação manual da interface no navegador (colar, alternar unidades, copiar).

## 11. Fora do escopo (v1)

- Outros laboratórios além do layout DASA (o painel de não reconhecidos mostra o que faltar).
- Resultados qualitativos (urina tipo 1, culturas, sorologias).
- Reordenação manual dos exames pela interface (a ordem fica no dicionário).
- Histórico de resultados ou de pacientes.
- Hospedagem (ex.: GitHub Pages); pode ser avaliada depois.

## 12. Questão em aberto

- Se o campo de evolução do Soul MV preserva negrito ao colar. Teste sugerido: copiar uma palavra em negrito do Word e colar na evolução. Se o negrito sumir, usar a opção de marcação `*` ou `↑↓`.
