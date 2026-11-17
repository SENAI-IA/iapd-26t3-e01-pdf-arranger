# Escavação — PDF Arranger 1.14.0

Missão: descobrir o que este código faz. Nenhum arquivo de `pdfarranger/` foi alterado.

Provas executadas neste ambiente: Python 3.10.12, pikepdf 10.13.0.post1, GTK 3. O typelib Poppler não está instalado e `img2pdf` não está instalado. Funções que só renderizam página ou convertem imagem ficaram no passo 4.

As regras abaixo estão em EARS e descrevem o comportamento observado.

## Passo 1 — Inventário

| Símbolo | Tipo | Efeito |
| --- | --- | --- |
| `Sides` | `NamedTuple` `(left, right, top, bottom)` de `float \| int` | Aritmética ponto a ponto. Sem estado global. |
| `Dims` | `NamedTuple` `(width, height)` | Idem. `cropped` pode devolver medida negativa. |
| `Page` / `LayerPage` | objetos mutáveis | `rotate` altera ângulo, crop, hide/offset e `size` no lugar, inclusive nas camadas. |
| `img2pdf_supported_img`, `IMG2PDF_VERSION` | estado global na importação | Sem `img2pdf`: lista `[]`, versão `"0.0.0"`, módulo `None`. Observado nesta máquina. |
| `POPPLER_VERSION` | string global | `Poppler.get_version()` na importação de `core.py`. |
| `Config` | `ConfigParser` em memória | `_config_file` cria o diretório do `config.ini`. |
| `export_doc` / `export_doc_job` | funções | Gravam PDF. `quit_flag` interrompe antes do `save`. `_scale` com fator diferente de 1 zera `/Rotate` da página de entrada. |
| `merge_doc` | função | Copia metadados para um dict novo. Abrir o XMP com o padrão do pikepdf substitui o Producer. |

Funções lidas e não executadas de ponta a ponta: diálogos GTK, `PDFRenderer`, `white_borders` (o render depende de Poppler), `undo.Manager` (exige a janela viva) e o ramo de sucesso de `_img_to_pdf`.

## Passo 2 a 5 — Regras, prova e contraprova

### R1 — Aritmética de `Sides` e `Dims`

WHEN o operando é outro `Sides`/`Dims` ou um número, a operação é ponto a ponto. IF o divisor é zero, THEN `ZeroDivisionError`. IF o operando não é número nem o mesmo tipo, THEN `TypeError`.

- Linha: `pdfarranger/core.py:90-152` e `195-257`. A subtração por escalar está em `120` e `225` (`self[i] -+ other`, que é subtração do valor positivo).
- Prova: previsão `Sides(9,3,12,6) - 3 = Sides(6,0,9,3)`. Saída: `Sides(left=6, right=0, top=9, bottom=3)`. `Sides / 0` levantou `ZeroDivisionError`. `Sides + "a"` levantou `TypeError`.
- Contraprova: tentar tratar `-+` como outra operação. A saída coincide com a subtração do docstring. A regra se mantém.

### R2 — `Dims.int_scaled` trunca depois de somar 0,5

WHEN `int_scaled(factor)` roda, o código devolve `int(medida * factor + 0.5)` em cada eixo.

- Linha: `pdfarranger/core.py:267-269`. O docstring da linha 268 diz “round to nearest int”.
- Prova: `Dims(0.5, 2.5).int_scaled(1)` → `Dims(width=1, height=3)`. `Dims(1.5, 2.5).int_scaled(1)` → `Dims(width=2, height=3)`. `Dims(-1.5, -0.5).int_scaled(1)` → `Dims(width=-1, height=0)`. No mesmo interpretador, `round(-1.5)` é `-2`.
- Contraprova: a frase do docstring não descreve o negativo. `-1.5` vira `-1`. Defeito em relação ao texto do próprio método: o arredondamento para o inteiro mais próximo não é o que a função executa nesse valor.

### R3 — `cropped` aceita soma de margens acima de 1

WHEN `left + right` passa de 1, a largura resultante é negativa. O mesmo vale para a altura. Não há teto nem piso.

- Linha: `pdfarranger/core.py:271-273`.
- Prova: `Dims(100, 200).cropped(Sides(0.6, 0.6, 0.1, 0.1))` → `Dims(width=-19.999999999999996, height=160.0)`. Soma exatamente 1: `Dims(width=0.0, height=200)`.
- Contraprova: procurar um clamp para zero. Não existe. A medida negativa é a saída real.

### R4 — `rotate_times` e o efeito de `Page.rotate` nas camadas

WHEN `Page.rotate(angle)` calcula `rotate_times`, o valor é `round((-angle / 90) % 4)`. IF esse valor é 0, THEN a função devolve `False` e não mexe na página. IF não é 0, THEN devolve `True`, gira crop e hide, soma `int(angle)` ao ângulo da página (módulo 360) e chama `LayerPage.rotate(vezes)`. A camada faz `ângulo - 90 * vezes` e só inverte o tamanho quando `vezes` é ímpar.

- Linha: `pdfarranger/core.py:321-323`, `346-356`, `424-429`.
- Prova:

| Chamada | Saída |
| --- | --- |
| `rotate_times` de 0, 45, 90, 134, 180, 270, 360, -90, 3690 | 0, 4, 3, 3, 2, 1, 0, 1, 3 |
| `rotate(0)`, `rotate(360)`, `rotate(3600)` | `False`; ângulo 0; tamanho 100×200; camada intacta |
| `rotate(90)` | `True`; página e camada no ângulo 90; tamanho 200×100; crop `Sides(0.4, 0.3, 0.1, 0.2)` nos dois |
| `rotate(180)` | página e camada no ângulo 180; tamanho original |
| `rotate(45)` | `True`; página ângulo 45 e tamanho 200×100; camada ângulo 0 e tamanho 100×200; crops iguais ao original |
| `rotate(134)` | `True`; página ângulo 134; camada ângulo 90; os dois tamanhos invertidos; crops iguais entre si |

- Contraprova: a hipótese “página e camada terminam com o mesmo ângulo” cai em 45° e em 134°. Em 45° o tamanho da página inverte e o da camada não. Defeito de sincronia fora dos múltiplos de 90 que zeram `rotate_times`. Em 90° e 180° a contraprova não achou divergência.

### R5 — O ângulo guardado na construção decide o tamanho

WHEN `BasePage` é construído, `size` é `size_orig` se `angle` é exatamente 0 ou 180. Nos outros valores, inclusive 360 e -180, `size` já nasce invertido. `unmodified()` exige `angle == 0`, crop e hide zerados, `scale == 1` e nenhuma camada.

- Linha: `pdfarranger/core.py:292` e `358-361`.
- Prova: construir com ângulo 0 ou 180 → `Dims(100, 200)`. Construir com 360, -180 ou 90 → `Dims(200, 100)`. `unmodified()` é `True` com escala `1.0` e `False` com ângulo 360 ou com uma camada.
- Contraprova: `Page.rotate(360)` a partir de 0 devolve `False` e deixa o tamanho em pé (R4). Construir já com `angle=360` inverte. São dois caminhos diferentes para o número 360. A regra da construção se mantém.

### R6 — `split` devolve todas as células menos a superior esquerda

WHEN `Page.split(vcrops, hcrops)` percorre as faixas de baixo para cima e, em cada faixa, da direita para a esquerda, a célula com início `(0.0, 0.0)` substitui o crop da própria página. As outras viram páginas novas.

- Linha: `pdfarranger/core.py:380-407`.
- Prova: página sem crop, colunas `(0, 0.5)` e `(0.5, 1)`, linha `(0, 1)`. Saída: uma página nova com `Sides(0.5, 0, 0, 0)`; a original fica `Sides(0, 0.5, 0, 0)`.
- Contraprova: a lista devolvida não contém a página original. A regra se mantém.

### R7 — Retângulos e MediaBox

WHEN `_normalize_rectangle` vê o primeiro canto à direita ou acima do segundo, troca os componentes. `_intersect_rectangle` aplica max nos mínimos e min nos máximos, também quando não há interseção. `_mediabox` usa `[0, 0, 612, 792]` se `/MediaBox` não existe. Com crop, as frações são `left/right` na largura e `bottom/top` na altura. IF `/Rotate` arredonda para 90, 180 ou 270, THEN o crop é permutado antes.

- Linha: `pdfarranger/exporter.py:94-145`. O passo de rotação está em `131-139`.
- Prova, página 100×200 e crop `Sides(0.1, 0.2, 0.3, 0.4)`:

| Entrada | Saída |
| --- | --- |
| cantos invertidos `[10, 20, 0, 0]` | `[0.0, 0.0, 10.0, 20.0]` |
| interseção de `[0,0,10,10]` com `[20,20,30,30]` | `[20, 20, 10, 10]` |
| sem crop | `[0.0, 0.0, 100.0, 200.0]` |
| sem `/MediaBox` | `[0.0, 0.0, 612.0, 792.0]` |
| Rotate 0, 45 ou 360 | `[10.0, 80.0, 80.0, 140.0]` |
| Rotate 89 ou 90 | `[30.0, 20.0, 60.0, 160.0]` |
| Rotate 135 ou 180 | `[20.0, 60.0, 90.0, 120.0]` |
| Rotate 270 ou -90 | `[40.0, 40.0, 70.0, 180.0]` |
| Rotate 450 | igual a 90° |

- Contraprova: 45° não vai para o ramo de 90°. `round(0.5)` neste Python é 0, então 45° usa o mesmo crop de 0°. 89° já cai no ramo de 90°. A interseção vazia devolve retângulo invertido; não há teste de área positiva.

### R8 — `_update_angle` soma uma vez e só tira 360 uma vez

WHEN `model_page.angle` é 0, a função não escreve `/Rotate`. WHEN o ângulo é diferente de 0, soma com o `/Rotate` da página fonte e, se o total é maior ou igual a 360, subtrai 360 uma única vez.

- Linha: `pdfarranger/exporter.py:184-191`.
- Prova: ângulo 0 com Rotate 90 deixa 90. Ângulo 0 sem Rotate deixa a chave ausente. 90+0 grava 90. 270+270 grava 180. 90+270 grava 0. 270+450 grava 360. Ângulo do modelo 360 com fonte 0 grava 0. Ângulo -90 grava -90.
- Contraprova: 720 (270+450) não vira 0. A normalização completa para o intervalo `[0, 360)` não é o que o código faz. Defeito de fronteira: um único `-= 360`.

### R9 — `_scale` com fator diferente de 1 altera a página de entrada

WHEN o fator é 1, a função devolve o mesmo objeto e não mexe em `/Rotate`. WHEN o fator é outro, zera `/Rotate` da página recebida e devolve uma página nova com esse ângulo e com o MediaBox multiplicado.

- Linha: `pdfarranger/exporter.py:157-181`.
- Prova: fator 1 devolve o mesmo objeto e o Rotate 90 continua 90. Fator 2: a entrada fica com Rotate 0, a nova fica com Rotate 90 e MediaBox `[0.0, 0.0, 200.0, 400.0]`.
- Contraprova: a página de entrada não permanece intacta no fator 2. Efeito colateral confirmado.

### R10 — `get_max_pdf_version` desempacota uma string sozinha

WHEN sobra uma única versão, `return max(*versions)` separa os caracteres. WHEN sobram duas ou mais, compara as strings inteiras, em ordem lexicográfica. `None` é ignorado. IF a lista filtrada fica vazia, THEN `TypeError`.

- Linha: `pdfarranger/exporter.py:383-391`.
- Prova: `Pdf.new().pdf_version` é `'1.3'`. `get_max_pdf_version` desse único PDF devolve `'3'`. O mesmo com um objeto cuja versão é a string `'1.3'`. Duas versões `'1.4'` e `'1.7'` devolvem `'1.7'`. `'1.10'`, `'1.9'` e `'1.7'` devolvem `'1.9'`. `None` no meio é pulado. Lista vazia e lista só de `None` levantam `TypeError: max expected at least 1 argument, got 0`.
- Contraprova: a leitura “devolve a maior versão do PDF” cai no caso de um único documento: a saída é o caractere `'3'`, não `'1.3'`. Efeito executado em seguida: `save(min_version='3')` concluiu, e o arquivo reaberto reportou `pdf_version` `'1.2'`. Entre várias strings, `'1.9'` ganha de `'1.10'`.

### R11 — `num_pages` só devolve contagem quando `open` não falha

WHEN `pikepdf.Pdf.open` abre o arquivo, a função devolve `len(pdf.pages)`. IF `open` levanta, THEN a cláusula `except` avalia `pikepdf._qpdf.PdfError`.

- Linha: `pdfarranger/exporter.py:534-542`.
- Prova: PDF de duas páginas → `2`. Arquivo de texto, caminho inexistente e PDF cifrado com senha → `AttributeError: module 'pikepdf' has no attribute '_qpdf'`. `hasattr(pikepdf, "_qpdf")` é `False`. `PasswordError` não é subclasse de `pikepdf.PdfError`.
- Contraprova: a previsão “falha de leitura devolve `None`” não se sustentou neste pikepdf 10.13. O `except` procura um módulo privado que esta versão não tem, então a falha vira `AttributeError`. Defeito ligado à versão da dependência.

### R12 — Flags de exportação

WHERE `quit_flag` já está marcado, `export_doc` volta antes de criar o arquivo. WHERE `files_out[0]` é `str` e `test_mode` é falso, grava metadados. WHERE `test_mode` é verdadeiro, o `save` usa as opções de teste e não chama `_set_meta`. WHERE o destino é `str` e há senha, cifra com essa senha. WHERE o destino não é `str`, a senha recebida não entra na cifragem. WHERE `config.start_with_empty()` é verdadeiro, `export` chama `export_doc`; no outro caso chama `export_doc_job`.

- Linha: `pdfarranger/exporter.py:394-437` e `523-531`.
- Prova: com o evento marcado, o arquivo de saída não existe. `test_mode=False` grava Producer `'pikepdf 10.13.0.post1'`. `test_mode=True` deixa Producer `None`. Arquivo com senha `secret`: abrir sem senha levanta `PasswordError`; com senha, abre. Destino `BytesIO` e a mesma senha: o buffer de 516 bytes abre sem senha. Espiando as duas funções, `start_with_empty` verdadeiro chama só `export_doc`; falso chama só `export_doc_job`.
- Contraprova: a senha não “sempre cifra”. No destino que não é caminho, o arquivo abre sem senha. A regra das flags se mantém.

### R13 — Vários arquivos no caminho Job usam `password = None`

WHEN `export_doc_job` recebe mais de um arquivo de saída e `output_password` é verdadeiro, monta `pikepdf.Encryption(user=password, owner=password, R=6)`. `password` foi atribuído `None` na linha 492 e não recebe `output_password`. WHEN há um único arquivo, a senha vai no JSON do Job.

- Linha: `pdfarranger/exporter.py:485-520`, em especial `492` e `505-506`.
- Prova: dois PDFs de saída com senha `secret` → `ValueError: Encryption owner may not be None; use empty string?`. Um único arquivo com a mesma senha: abrir sem senha levanta `PasswordError`; com `secret`, abre.
- Contraprova: a exportação partida em vários arquivos não fica cifrada com a senha pedida. Ela aborta. Defeito: a variável usada na cifra não é a senha recebida.

### R14 — Metadados: validade, texto e precedência

WHEN o valor passa em `re.sub('', '', valor)`, `_pikepdf_meta_is_valid` devolve verdadeiro. Lista vazia passa, porque o laço não roda. Bytes, `int` e `None` devolvem falso. `_metatostr` devolve a string, ou o único item de uma lista de Creator, ou `json.dumps` se a lista de Creator tem mais de um item; nos outros tipos devolve `''`. `_strtometa` só faz `json.loads` quando o nome é Creator. Lista vazia vira `None`. JSON que não é lista devolve o texto original. JSON inválido devolve o texto original. `_safeiter` segue depois de `KeyError` e `ValueError`. `TypeError` propaga. `merge_doc` não substitui chave que já está no dict, e o primeiro arquivo ganha do segundo.

- Linha: `pdfarranger/metadata.py:54-66`, `81-92`, `95-118`, `127-149`.
- Prova: `"abc"` verdadeiro; `b"abc"`, `1` e `None` falsos; `[]` verdadeiro; `["a", 1]` falso. Creator `["A"]` → `'A'`; `["A", "B"]` → `'["A", "B"]'`; a mesma lista com outro nome → `''`; `123` → `''`. `'["A", "B"]'` volta `['A', 'B']`; `'[]'` volta `None`; `'{"a": 1}'` volta o texto `'{"a": 1}'`; `'"quoted"'` volta `'"quoted"'`; `'null'` volta `'null'`; `'["A"]'` com outro nome continua a string. O iterador que solta `KeyError` e `ValueError` no meio ainda entrega `k1` e `k4`. `TypeError` sai da função. Dois PDFs com description `primeiro` e `segundo`: o dict fica `primeiro`. Com a chave já preenchida como `inicial`, continua `inicial`.
- Contraprova: o comentário da linha 57 diz que byte-like é aceito. `b"abc"` devolve `False`, porque o padrão de `re.sub` é `str`. Defeito em relação a esse comentário. A precedência “o que já está no dict ganha, e o primeiro arquivo ganha do segundo” se mantém para `description`.

### R15 — `_crops` redistribui pelo tamanho do primeiro trecho

WHEN a soma das porcentagens é exatamente 100, devolve os intervalos encostados, sem os de comprimento zero. WHEN a soma é outra e há um único trecho, THEN divide por `num_splits - 1` e levanta `ZeroDivisionError`. WHEN há vários trechos, o reajuste usa o tamanho do primeiro, também se a soma fica abaixo de 100.

- Linha: `pdfarranger/splitter.py:178-211`. O comentário da linha 192 marca o overlap como não negativo; a linha 203 diz que o overlap só está definido quando os trechos têm o mesmo tamanho.
- Prova:

| Entrada (porcentagens) | Saída |
| --- | --- |
| 35, 35, 35 | `[(0, 0.35), (0.325, 0.675), (0.65, 1.0)]` |
| 60, 60 | `[(0, 0.6), (0.4, 1.0)]` |
| 50, 50 | `[(0, 0.5), (0.5, 1.0)]` |
| 40, 40 | `[(0, 0.4), (0.6, 1.0)]` |
| 50, 30, 40 | `[(0, 0.5), (0.4, 0.9), (0.5, 1.0)]` |
| 150 | `ZeroDivisionError` |
| 40 | `ZeroDivisionError` |
| 100 | `[(0, 1.0)]` |
| 0, 50 | `[(1.0, 1.0)]` |

- Contraprova: 40+40 não é rejeitado nem completado até cobrir a página sem buraco; fica um vão de 0.4 a 0.6. 50, 30 e 40 não conservam 30% e 40%: os três intervalos saem com o tamanho do primeiro. O trecho 0 seguido de 50 vira um intervalo vazio em 1. A divisão por zero com um único trecho diferente de 100 confirma o limite.

### R16 — `scale` prende o fator para a página ficar entre 72 e 14400 pt

WHEN o fator é um número, ele é usado direto. WHEN é um par `(largura, altura)`, o fator é o menor dos dois eixos. Em seguida o fator sobe até cada lado do tamanho já recortado atingir 72 pt, e desce até cada lado caber em 14400 pt. IF um lado recortado é 0, THEN `ZeroDivisionError`. Seleção vazia devolve `False` e não altera nada. A escala das camadas é multiplicada por `fator_novo / escala_antiga`.

- Linha: `pdfarranger/pageutils.py:30-56`.
- Prova, página 100×200 com escala 1: fator 2 → escala 2, pontos 200×400, `changed` verdadeiro. Fator 0.1 → escala 0.72, pontos 72×144. Fator 1000 → escala 72, pontos 7200×14400. Par `(50, 50)` → 0.72. Página que já está em 0.72, fator 0.1 → `changed` falso e escala 0.72. Seleção vazia → `False`. Largura 0 → `ZeroDivisionError`. Camada com escala 2, página indo de 1 para 2 → camada fica 4.
- Contraprova: 0.1 não permanece 0.1. O piso de 72 pt no lado de 100 pt produz 0.72. A regra do intervalo se mantém.

### R17 — Caminho do `config.ini` e a flag `start-with-empty`

WHERE o processo está congelado no Windows e existe `config.ini` ao lado do executável, `_config_file` devolve esse arquivo. WHERE não, segue a ordem: Darwin usa `~/Library/Preferences`; senão `APPDATA`, se existir; senão `XDG_CONFIG_HOME`; senão `~/.config`. O diretório é criado. WHERE pikepdf é anterior a 8, `start_with_empty()` devolve `True` e `set_start_with_empty(False)` grava verdadeiro. WHERE pikepdf é 8 ou mais, vale o booleano do ini, com padrão falso.

- Linha: `pdfarranger/config.py:135-153` e `245-252`. A comparação de versão está na linha 182.
- Prova: com `XDG_CONFIG_HOME=/tmp/escavacao-xdg` a saída é `/tmp/escavacao-xdg/escavacao-domain/config.ini`. Com `APPDATA` e `XDG_CONFIG_HOME` juntos, o caminho fica em `APPDATA`. Com `platform.system` Darwin e `APPDATA` definido, a saída é `/home/x/Library/Preferences/dom/config.ini`. pikepdf 10 e ini `false` → `False`. Ini `true` → `True`. Versão forçada para 6.2.0 e ini `false` → a leitura continua `True`. `set_start_with_empty(False)` nesse ramo grava a string `True`.
- Contraprova: `APPDATA` não perde para `XDG_CONFIG_HOME` neste código: o `elif` de `APPDATA` vem antes. Darwin não olha `APPDATA`. No teste do executável congelado, `os.name` foi simulado como `nt` neste Linux: com o arquivo existente a função voltou cedo e o `os.path` daqui produziu `config.ini` (barra invertida não é separador neste SO); sem o arquivo, caiu em `C:\Users\a\AppData\Roaming/dom/config.ini`. O texto exato do caminho Windows nativo não foi obtido. A ordem dos ramos, sim.

### R18 — Atalhos: `enable_custom`, `zoom-fit` e o menu de contexto

WHERE `enable_custom` é falso, cada atalho conhecido é reescrito com `_DEFAULT_ACCELS`. WHERE é verdadeiro, valor já presente permanece. `zoom-fit` é apagado se existir. `is_popup_key_event` exige o keyval e que os modificadores do atalho estejam ligados; bits extras no evento continuam valendo.

- Linha: `pdfarranger/config.py:171-177`, `184-188`, `190-195`. O padrão de `context-menu` está na linha 65: `<Shift>F10 Menu`.
- Prova: `enable_custom=false` com `save = F1` termina com `save` igual a `<Primary>s`. `enable_custom=true` conserva `F1`. Nos dois casos `zoom-fit` some. O primeiro acelerador do menu é keyval 65479 (F10) e mods 1 (Shift). Evento com Shift mais outro bit → `True`. Sem o Shift → `False`. Outra tecla com Shift → `False`.
- Contraprova: um modificador a mais não bloqueia o atalho. A comparação é `state & mods == mods`. A regra se mantém.

### R19 — Seleção contígua e tamanho “igual”

WHEN `is_selection_contiguous` recebe números, devolve verdadeiro se `len(lista) == ultimo - primeiro + 1`. A função usa o primeiro e o último item na ordem recebida. Lista vazia levanta `IndexError`. `is_same_page_size` compara `size_in_points` com a primeira página e devolve falso quando a diferença absoluta de largura ou de altura é maior que `1e-2`. Uma página só devolve verdadeiro. Lista vazia levanta `IndexError`.

- Linha: `pdfarranger/pdfarranger.py:3418-3424` e `3432-3433`.
- Prova: `[1,2,3]` verdadeiro; `[1,2,4]` falso; `[1,3,2]` falso; `[5]` verdadeiro; `[]` `IndexError`; `[1,2,2]` falso. Larguras 100 e 100.009: diferença `0.009000000000000341`, que não é maior que `0.01`, resultado `True`. Larguras 100 e 100.01: diferença `0.010000000000005116`, resultado `False`. Uma página: `True`. Lista vazia: `IndexError`.
- Contraprova: `{1,2,3}` fora de ordem, na lista `[1,3,2]`, não é tratado como bloco contíguo. A função não ordena. O literal `100.01` também não passa na tolerância de um centésimo de ponto, porque o float já fica acima de `1e-2`.

### R20 — URI, serialização e `is_ctrl_pressed`

WHEN `get_file_path_from_uri` recebe um texto, descarta os cinco primeiros caracteres, passa por `url2pathname`, remove de cada ponta CR, LF, NUL e `/` (`\x2F`), e no POSIX coloca uma barra na frente. `Page.serialize` junta os campos com `///`. `deserialize` faz `split('///')` e converte posição fixa para `int`. `is_ctrl_pressed` está no nível do módulo, com `@staticmethod`, e chama `ctypes.windll`.

- Linha: `pdfarranger/pdfarranger.py:228-235`, `238-242`, `core.py:363-368`, `pdfarranger.py:2023-2050`.
- Prova: `file:///tmp/a%20b.pdf` → `/tmp/a b.pdf`. `file:///tmp/foo/` → `/tmp/foo`. O texto `/home/cristiano/arquivo.pdf` → `/cristiano/arquivo.pdf`. Com `os.name` simulado `nt`, `file:///C:/Users/a.pdf` → `C:/Users/a.pdf`. Serializar descrição `d` e desserializar devolve `('copy', 1, 'd', 0, 1.0, ...)`. Descrição `a///b` levanta `ValueError: invalid literal for int() with base 10: 'b'`. `is_ctrl_pressed()` neste Linux levanta `AttributeError: module 'ctypes' has no attribute 'windll'`.
- Contraprova: a descrição com o separador `///` não sobrevive ao cola. O caminho que não começa com `file:` perde cinco caracteres: `home` some de `/home/...`. A barra final de diretório também some, porque `\x2F` no `strip` da linha 232 é `/`.

### R21 — `duplicate(incl_thumbnail=False)` zera a miniatura da cópia

WHEN `incl_thumbnail` é falso, a cópia perde a miniatura e a prévia. O objeto original conserva a miniatura.

- Linha: `pdfarranger/core.py:370-378`.
- Prova: miniatura original era um objeto sentinela e continuou sendo esse objeto. Na cópia, `thumbnail` é `None` e `preview` é `None`.
- Contraprova: a cópia não compartilha a miniatura com o original nesse modo. A regra se mantém.

## Passo 4 — O que o arquivo não permite concluir

- O pixel branco do Cairo é o zero que `white_borders` compara? A sentinela executada é `b'\x00' * 16` (`pdfarranger/pageutils.py:409`). Sem o typelib Poppler não houve render, então não dá para dizer se uma página em branco cai nessa comparação.
- O que `_img_to_pdf` faz com imagem RGBA e com EXIF inválido? O `import img2pdf` falhou aqui. O ramo executado é o `except`: lista vazia e versão `"0.0.0"` (`core.py:50-61`). O ramo do `convert` não rodou.
- `PDFRenderer`, diálogos e `undo.Manager` dependem da janela GTK e do fio de render. Não foram exercitados.
- O caminho final de um PDF Arranger congelado no Windows. O ramo “arquivo ao lado do executável” foi tomado sob `os.name == 'nt'` simulado, mas `os.path` deste Linux não trata `\`. A string nativa `C:\app\config.ini` não saiu desta prova.
- Se o Producer gravado no PDF de origem sobrevive a `merge_doc`. Na prova, abrir o XMP com o padrão `set_pikepdf_as_editor=True` fez o pikepdf avisar que ia sobrescrever Producer, e o valor lido foi `'pikepdf 10.13.0.post1'`. A precedência entre arquivos ficou demonstrada em `description`, não nesse campo.

## Defeitos que a contraprova confirmou

1. `get_max_pdf_version` com uma única versão devolve um caractere (`'1.3'` → `'3'`), e `save(min_version='3')` reabre como `'1.2'`. Com várias versões, a ordem é a lexicográfica (`'1.9'` acima de `'1.10'`).
2. `export_doc_job` com mais de um arquivo e senha chama `Encryption` com `None` e levanta `ValueError`.
3. `num_pages`, neste pikepdf 10.13, transforma erro de abertura em `AttributeError` porque `pikepdf._qpdf` não existe.
4. `Page.rotate(45)` e `Page.rotate(134)` deixam o ângulo da camada diferente do ângulo da página. Em 45° os tamanhos também divergem.
5. `_update_angle` guarda 360 quando a soma é 720.
6. `_crops` com um único trecho diferente de 100% divide por zero; com trechos desiguais usa só o primeiro; com soma abaixo de 100 abre um vão.
7. `Dims.int_scaled(-1.5)` devolve `-1`. `cropped` devolve largura negativa. `deserialize` quebra se a descrição contém `///`. `get_file_path_from_uri` sempre corta cinco caracteres e remove `/` das pontas.
