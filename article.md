> link para o artigo original [Aqui](http://blog.bjrn.se/2008/10/lets-build-mp3-decoder.html)

# Introdução

Embora o MP3 seja provavelmente o formato de arquivo e codec mais conhecido do mundo, ele não é muito bem compreendido pela maioria dos programadores — para muitos, codificadores/decodificadores estão na categoria de softwares que "outras pessoas" escrevem, como bibliotecas padrão ou kernels de sistemas operacionais. Este artigo tenta desmistificar o decodificador, com introduções curtas e diretas sobre processamento de sinais e teoria da informação, quando necessário. Além disso, será escrito um pequeno decodificador (em Haskell), não completo, mas adequado para experimentação.

O foco deste artigo está nos conceitos e nas decisões de design que a equipe do MPEG tomou ao criar o codec — não em detalhes de implementação sem graça ou teoria pesada. Algumas partes de um decodificador são bastante obscuras e são melhor compreendidas lendo a especificação, um bom livro sobre processamento de sinais ou os diversos artigos sobre MP3 (veja as referências no final).

Uma observação sobre o código: o decodificador que acompanha este artigo foi escrito pensando na legibilidade, não na velocidade. Além disso, alguns recursos incomuns foram deixados de fora. O resultado final é um decodificador ineficiente e que não segue o padrão, mas que, com sorte, tem um código fácil de entender. Você pode baixar o código-fonte aqui: `mp3decoder-0.0.1.tar.gz`. Role até o final do artigo ou veja o README para instruções de compilação.

Um aviso justo: o autor é um programador hobbista, não uma autoridade em processamento de sinais. Se você encontrar algum erro, por favor, me mande um e-mail: be@bjrn.se

Com isso fora do caminho, começamos nossa jornada pelo ouvido.

# Audição Humana e Psicoacústica

A ideia principal da codificação MP3 — e da compressão de áudio com perda em geral — é remover informações acusticamente irrelevantes de um sinal de áudio para reduzir seu tamanho. O trabalho do codificador é eliminar parte ou toda a informação de um componente do sinal, sem ao mesmo tempo alterar o sinal de forma que cause artefatos audíveis.

Várias características (ou “limitações”) da audição humana são exploradas pelos codecs de áudio com perda. Uma dessas características básicas é que não conseguimos ouvir sons acima de 20 kHz ou abaixo de 20 Hz, aproximadamente. Além disso, existe um limiar de audição — quando um sinal está abaixo desse limiar, ele não é audível por ser fraco demais. Esse limiar varia com a frequência: um tom de 20 Hz só pode ser ouvido se for mais forte que cerca de 60 decibéis, enquanto frequências na faixa de 1 a 5 kHz podem ser percebidas facilmente mesmo em volumes baixos.

Uma propriedade muito importante que afeta o sistema auditivo é conhecida como mascaramento. Um sinal alto pode “mascarar” outros sinais próximos em frequência ou no tempo — ou seja, o sinal mais alto modifica o limiar de audição para vizinhos espectrais e temporais. Essa propriedade é extremamente útil: não apenas os sinais mascarados podem ser removidos, como também o próprio sinal audível pode ser comprimido ainda mais, pois o ruído introduzido pela compressão intensa também será mascarado.

Esse fenômeno do mascaramento ocorre dentro de regiões de frequência conhecidas como bandas críticas — um sinal forte dentro de uma banda crítica mascarará outras frequências dentro da mesma banda. Podemos imaginar a orelha como um conjunto de filtros passa-faixa, onde diferentes partes da orelha captam diferentes regiões de frequência. Um audiologista ou professor de acústica teria muito a dizer sobre bandas críticas e os detalhes dos efeitos de mascaramento, mas neste artigo adotamos uma abordagem mais simples e voltada à engenharia: para nosso propósito, basta pensar nessas bandas críticas como regiões fixas de frequência onde ocorrem os efeitos de mascaramento.

Usando essas propriedades do sistema auditivo humano, codecs e codificadores com perda removem sinais inaudíveis para reduzir a quantidade de informação, comprimindo assim o sinal. O padrão MP3 não define como um codificador deve ser implementado (embora assuma a existência de bandas críticas), e os desenvolvedores têm bastante liberdade para remover conteúdos que considerem imperceptíveis. Um codificador pode decidir que determinada frequência é inaudível e deve ser removida, enquanto outro pode manter o mesmo sinal. Codificadores diferentes usam modelos psicoacústicos diferentes — modelos que descrevem como os humanos percebem sons e, assim, que informações podem ser descartadas.

## Sobre o MP3

Antes de começarmos a decodificar MP3, é necessário entender exatamente o que é MP3. MP3 é um codec formalmente conhecido como MPEG-1 Audio Layer 3, e está definido no padrão MPEG-1. Esse padrão define três codecs de áudio diferentes, onde o layer 1 é o mais simples e possui a pior taxa de compressão, e o layer 3 é o mais complexo, mas tem a maior taxa de compressão e a melhor qualidade de áudio por taxa de bits. O layer 3 é baseado no layer 2, que por sua vez é baseado no layer 1. Todos os três codecs compartilham semelhanças e têm muitas partes de codificação/decodificação em comum.

A razão por trás dessa escolha de design fazia sentido na época em que o padrão MPEG-1 foi escrito, já que as semelhanças entre os três codecs facilitariam o trabalho dos desenvolvedores. Olhando em retrospecto, construir o layer 3 sobre os outros dois talvez não tenha sido a melhor ideia. Muitos dos recursos avançados do MP3 foram "encaixados à força" e são mais complexos do que seriam se o codec tivesse sido projetado do zero. Na verdade, muitos dos recursos do AAC foram projetados para serem mais "simples" do que os equivalentes no MP3.

De forma bem geral, um codificador MP3 funciona assim: uma fonte de entrada, como um arquivo WAV, é fornecida ao codificador. Ali, o sinal é dividido em partes (no domínio do tempo) para serem processadas individualmente. O codificador então pega um desses sinais curtos e o transforma para o domínio da frequência. O modelo psicoacústico remove o máximo de informação possível, com base no conteúdo e em fenômenos como o mascaramento. As amostras de frequência, agora com menos informação, são comprimidas em uma etapa genérica de compressão sem perdas. As amostras, assim como os parâmetros sobre como foram comprimidas, são então escritas em disco em um formato binário.

O decodificador funciona de forma inversa. Ele lê o formato de arquivo binário, descomprime as amostras de frequência, reconstrói as amostras com base nas informações de como o conteúdo foi removido pelo modelo, e depois as transforma de volta para o domínio do tempo. Vamos começar pelo formato de arquivo binário.

Where the ByteString is the unparsed bit stream, and the [Word8] is an internal buffer used to reconstruct logical frames from physical frames. Not familiar with Haskell? Don’t worry; all the code in this article is only complementary.

As the bit stream may contain data we consider garbage, such as ID3 tags, we are using a simple helper function, mp3Seek, which takes the MP3Bitstream and discards bytes until it finds a valid header. The new MP3Bitstream can then be passed to a function that does the actual physical to logical unpacking.

# Decodificando, passo 1: Entendendo os dados

Muitos usuários de computador sabem que um arquivo MP3 é composto por vários “quadros” – blocos consecutivos de dados. Embora importantes para interpretar o fluxo de bits, esses quadros não são fundamentais e não podem ser decodificados individualmente. Neste artigo, o que normalmente é chamado de quadro será chamado de **quadro físico**, enquanto o bloco de dados que pode realmente ser decodificado será chamado de **quadro lógico**, ou simplesmente **quadro**.

Um quadro lógico possui várias partes: um cabeçalho de 4 bytes facilmente distinguível de outros dados no fluxo de bits, 17 ou 32 bytes chamados de **informações auxiliares**, e algumas centenas de bytes com os **dados principais**.

Um quadro físico possui um cabeçalho, um **checksum** (verificação) opcional de 2 bytes, informações auxiliares, e apenas parte dos dados principais — exceto em casos muito raros. A imagem abaixo (que não está disponível) mostra um quadro físico como uma borda preta grossa, o cabeçalho como 4 bytes vermelhos, e as informações auxiliares como bytes azuis (este MP3 não possui o checksum opcional). Os bytes em cinza representam os dados principais que correspondem ao cabeçalho e informações auxiliares destacados. O cabeçalho do quadro físico seguinte também está destacado, para mostrar que o cabeçalho sempre começa no deslocamento 0.

> imagem ausente :(

A primeira coisa que fazemos ao decodificar um MP3 é **desempacotar os quadros físicos em quadros lógicos** – isso é uma forma de abstração. Uma vez que temos um quadro lógico, podemos esquecer todo o restante do fluxo de bits. Fazemos isso lendo um valor de deslocamento nas informações auxiliares que aponta para o início dos dados principais.

### Por que os dados principais de um quadro lógico não ficam totalmente dentro do quadro físico?

No início isso pode parecer desnecessariamente complicado, mas tem suas vantagens. O tamanho de um quadro físico é constante (com variação de no máximo 1 byte) e é baseado apenas na taxa de bits e outros valores armazenados no cabeçalho, que é fácil de localizar. Isso permite que players de mídia acessem quadros arbitrários com eficiência. Além disso, como os quadros não são limitados a um tamanho fixo em bits, partes do áudio com sons mais complexos podem usar bytes de quadros anteriores — essencialmente oferecendo taxa de bits variável para todos os MP3s.

No entanto, há algumas limitações: um quadro pode salvar seus dados principais em vários quadros anteriores, mas **não em quadros seguintes** — isso dificultaria o streaming. Também, os dados principais de um quadro não podem ser grandes demais, sendo limitados a cerca de **500 bytes**. Esse limite é relativamente curto e é alvo de muitas críticas.

O leitor mais atento pode perceber que os bytes cinza dos dados principais na imagem (não exibida) começam com um padrão interessante (`3E 50 00 00...`), que se parece com os primeiros bytes dos dados principais do próximo quadro lógico (`38 40 00 00...`). Existe alguma estrutura nesses dados, mas geralmente isso não é visível em um editor hexadecimal.

### Trabalhando com o fluxo de bits

Para lidar com o fluxo de bits, vamos usar um tipo de dado bem simples:

```
data MP3Bitstream = MP3Bitstream {
    bitstreamStream :: B.ByteString,
    bitstreamBuffer :: [Word8]
}
```

Nessa estrutura, `ByteString` representa o fluxo de bits ainda não processado, e `[Word8]` é um buffer interno usado para reconstruir quadros lógicos a partir de quadros físicos. Não conhece Haskell? Sem problema — todo o código neste artigo é apenas complementar.

Como o fluxo de bits pode conter dados que consideramos lixo, como tags ID3, vamos usar uma função auxiliar simples, `mp3Seek`, que recebe um `MP3Bitstream` e descarta bytes até encontrar um cabeçalho válido. O novo `MP3Bitstream` pode então ser passado para uma função que faz o desempacotamento físico-lógico de fato.

```
mp3Seek :: MP3Bitstream -> Maybe MP3Bitstream
mp3UnpackFrame :: MP3Bitstream -> (MP3Bitstream, Maybe MP3LogicalFrame)
```

# A anatomia de um quadro lógico

Quando terminamos a decodificação propriamente dita, um quadro lógico terá nos fornecido exatamente 1152 amostras no domínio do tempo por canal. Em um arquivo PCM WAV típico, armazenar essas amostras exigiria 2304 bytes por canal – mais de 4½ KB no total para uma faixa de áudio comum. Embora grande parte da compressão de 4½ KB para 0,4 KB venha da remoção de conteúdo de frequência, uma parte não insignificante se deve a uma representação binária extremamente eficiente.

Antes disso, precisamos entender o quadro lógico, especialmente as informações laterais e os dados principais. Quando terminamos de analisar o quadro lógico, teremos o áudio comprimido e um conjunto de parâmetros que descrevem como descompactá-lo.

Desempacotar o quadro lógico exige conhecer suas diferentes partes. O cabeçalho de 4 bytes armazena algumas propriedades do sinal de áudio – o mais importante é a taxa de amostragem e o modo de canal (mono, estéreo etc). As informações no cabeçalho são úteis tanto para o software do reprodutor de mídia quanto para a decodificação do áudio. Vale lembrar que o cabeçalho não armazena muitos dos parâmetros usados pelo decodificador – como, por exemplo, como as amostras de áudio devem ser reconstruídas. Esses parâmetros estão armazenados em outras partes.

As informações laterais ocupam 17 bytes no modo mono, e 32 bytes nos demais casos. Há muitas informações nas side infos. A maior parte dos bits descreve como os dados principais devem ser analisados, mas também há alguns parâmetros ali que são usados por outras partes do decodificador.

Os dados principais contêm dois “blocos” por canal, que são porções de áudio comprimido (e seus parâmetros correspondentes), decodificados individualmente. Um quadro mono tem dois blocos, enquanto um quadro estéreo tem quatro. Essa divisão é um legado das camadas 1 e 2. A maioria dos codecs modernos criados do zero não se preocupa mais com essa separação.

Os primeiros bits de cada bloco são os chamados fatores de escala – basicamente 21 números, usados para decodificar o bloco mais tarde. O motivo pelo qual os fatores de escala estão nos dados principais, e não nas informações laterais como outros parâmetros, é que eles ocupam bastante espaço. Como esses fatores devem ser interpretados – por exemplo, quantos bits cada um tem – é descrito nas informações laterais.

Após os fatores de escala vêm os dados de áudio comprimido propriamente ditos. São algumas centenas de números que ocupam a maior parte do espaço em cada bloco. Esses dados são comprimidos de uma forma que muitos programadores já conhecem: codificação de Huffman, usada por zip, zlib e outros métodos comuns de compressão de dados sem perda.

A codificação de Huffman é, na verdade, um dos maiores motivos para um arquivo MP3 ser tão pequeno em comparação ao áudio bruto – e vale a pena investigá-la com mais profundidade. Por enquanto, vamos fingir que já decodificamos completamente os dados principais, incluindo os dados codificados em Huffman. Quando fizermos isso para todos os quatro blocos (ou dois no caso de mono), teremos desempacotado o quadro com sucesso. A função que realiza essa tarefa é:

```
mp3ParseMainData :: MP3LogicalFrame -> Maybe MP3Data
```

Onde `MP3Data` armazena algumas informações, e os dois/quatro blocos já analisados.

# Codificação de Huffman

A ideia básica da codificação de Huffman é simples. Pegamos alguns dados que queremos comprimir, como uma lista de caracteres de 8 bits. Em seguida, criamos uma tabela de valores onde ordenamos os caracteres por frequência. Se não soubermos de antemão como será a nossa lista de caracteres, podemos ordená-los por probabilidade de ocorrência na string. Depois, atribuiremos códigos a essa tabela, dando os códigos mais curtos aos valores mais prováveis. Um código nada mais é do que um número inteiro de n bits projetado de forma a não haver ambiguidades ou conflitos com códigos menores.

Por exemplo, digamos que temos uma string muito longa composta pelas letras A, C, G e T. Como bons programadores, percebemos que é desperdício armazenar essa string como caracteres de 8 bits, então usamos 2 bits para cada letra. A codificação de Huffman pode comprimir ainda mais, se algumas letras forem mais frequentes que outras. No nosso exemplo, sabemos de antemão que a letra 'A' aparece na string com aproximadamente 40% de probabilidade. Criamos então uma tabela de frequências:

```
A	40%
C	35%
G	20%
T	5%
```

Em seguida, atribuímos códigos a essa tabela. Isso é feito de uma maneira específica – se atribuirmos códigos aleatórios, já não estamos mais usando Huffman, mas um código genérico de comprimento variável.

```
A	0
C	10
G	110
T	111
```

Digamos que temos uma string com mil caracteres. Se armazenarmos essa string em ASCII, ela ocupará 8000 bits. Se usarmos a representação de 2 bits por letra, ocupará 2000 bits. Mas com Huffman, podemos armazenar em apenas 1850 bits.

A decodificação é o processo inverso da codificação. Se tivermos uma sequência de bits, como `00011111010`, lemos os bits até encontrar uma correspondência na tabela. Essa sequência, no exemplo, decodifica para `AAATGC`. Note que a tabela de códigos é construída para que não haja conflitos. Se a tabela fosse assim:

```
A	0
C	01
```

... e encontrássemos o bit `0`, nunca poderíamos obter um `C`, já que `A` corresponderia sempre.

O método padrão para decodificar uma string codificada com Huffman é percorrer uma árvore binária, criada a partir da tabela de códigos. Quando encontramos um bit `0`, vamos — digamos — à esquerda na árvore, e à direita com um bit `1`. Esse é o método mais simples usado no decodificador.

Existe um método mais eficiente para decodificar a string, um clássico trade-off entre tempo e espaço, usado quando a mesma tabela de códigos é usada para codificar e decodificar várias sequências de bits diferentes — como no caso do MP3. Em vez de percorrer uma árvore, usamos uma tabela de consulta de forma inteligente. Isso é melhor ilustrado com um exemplo:

```
lookup[0xx] = (A, 1)
lookup[10x] = (C, 2)
lookup[110] = (G, 3)
lookup[111] = (T, 3)
```

Na tabela acima, `xx` significa todas as permutações de 2 bits — todos os padrões de bits de `00` a `11`. Nossa tabela então contém todos os índices de `000` a `111`. Para decodificar uma string com essa tabela, espiamos 3 bits da sequência codificada. Nossa string exemplo é `00011111010`, então o índice é `000`. Isso corresponde ao par `(A, 1)`, o que significa que encontramos o valor `A` e devemos descartar 1 bit da entrada. Espiamos mais 3 bits da string e repetimos o processo.

Para tabelas de Huffman muito grandes, onde o maior código tem dezenas de bits, não é viável criar uma tabela de consulta com esse método de preenchimento, pois seria necessário uma tabela com aproximadamente `2^n` elementos, onde `n` é o tamanho do maior código. No entanto, ao observar cuidadosamente a tabela de códigos, muitas vezes é possível criar uma tabela de consulta muito eficiente manualmente, usando um método com "ponteiros" para diferentes tabelas, que tratam os códigos mais longos.

# Como a codificação de Huffman é usada no MP3

Para entender como a codificação de Huffman é usada no MP3, é necessário compreender exatamente o que está sendo codificado ou decodificado. Os dados comprimidos que vamos descomprimir são amostras no domínio da frequência. Cada quadro lógico tem até quatro blocos – dois por canal – cada um contendo até 576 amostras de frequência. Para um sinal de áudio de 44100 Hz, a primeira amostra de frequência (índice 0) representa frequências em torno de 0 Hz, enquanto a última amostra (índice 575) representa uma frequência em torno de 22050 Hz.

Essas amostras são divididas em cinco regiões de comprimento variável. As três primeiras regiões são conhecidas como "regiões de valores grandes", a quarta é chamada de "região count1" (ou região quad), e a quinta é a "região zero". As amostras na região zero são todas zero, então não são realmente codificadas com Huffman. Se as regiões de valores grandes e a região quad decodificarem para 400 amostras, as 176 restantes são preenchidas com zeros.

As três regiões de valores grandes representam as frequências mais baixas e importantes do áudio. O nome "valores grandes" refere-se ao conteúdo informativo: após a decodificação, essas regiões contêm inteiros no intervalo de –8206 a 8206.

Essas três regiões são codificadas com três diferentes tabelas de Huffman, definidas no padrão MP3. O padrão define 15 tabelas grandes para essas regiões, onde cada tabela gera duas amostras de frequência para um determinado código. As tabelas são projetadas para comprimir ao máximo o conteúdo "típico" dessas regiões de frequência.

Para aumentar ainda mais a compressão, as 15 tabelas são combinadas com outro parâmetro, totalizando 29 maneiras diferentes de comprimir cada uma das três regiões. As informações auxiliares (side information) contêm qual das 29 possibilidades deve ser usada. Confusamente, o padrão chama essas possibilidades de "tabelas". Vamos chamá-las de "pares de tabela".

### Exemplo

Aqui está a tabela de Huffman 1 (table1), conforme definida no padrão:

```
Código     Valor
1          (0, 0)
001        (0, 1)
01         (1, 0)
000        (1, 1)
```

E aqui está o par de tabela 1: (table1, 0).

Para decodificar uma região de valores grandes usando o par de tabela 1, fazemos o seguinte: Suponha que o bloco contenha os bits: `000101010...`. Primeiro decodificamos os bits como normalmente se faz com Huffman: os três bits `000` correspondem às duas amostras de saída 1 e 1, que chamamos de x e y.

### O detalhe importante

A maior tabela de códigos definida no padrão tem amostras com valor máximo de 15. Isso é suficiente para representar a maioria dos sinais de forma satisfatória, mas às vezes é necessário um valor maior. O segundo valor do par de tabela é conhecido como "linbits", e sempre que encontramos uma amostra com valor 15, lemos `linbits` bits adicionais e somamos à amostra. Para o par de tabela 1, o linbits é 0, e o valor máximo nunca é 15, então ignoramos nesse caso. Para outras amostras, linbits pode ser até 13, então o valor máximo pode ser 15 + 8191.

Depois de ler os linbits da amostra x, obtemos o sinal. Se x não for 0, lemos um bit. Se for 1, então x é negativo.

No total, as duas amostras são decodificadas assim:

1. Decodificar os primeiros bits usando a tabela de Huffman. Chamar os valores de x e y.
2. Se x = 15 e linbits ≠ 0, ler `linbits` bits e somar a x. Agora x pode ser até 8206.
3. Se x ≠ 0, ler 1 bit. Se for 1, então x = –x.
4. Fazer os passos 2 e 3 para y.

### Região count1

A região count1 codifica frequências tão altas que foram compactadas fortemente. Ao serem decodificadas, as amostras variam de –1 a 1. Existem apenas duas tabelas possíveis para essa região; são chamadas de "tabelas quad" pois cada código gera 4 amostras. Não há linbits aqui, então basta usar a tabela apropriada e pegar os bits de sinal.

1. Decodificar os primeiros bits usando a tabela de Huffman. Chamar as amostras de v, w, x e y.
2. Se v ≠ 0, ler 1 bit. Se for 1, então v = –v.
3. Fazer o passo 2 para w, x e y.

# Passo 1 - Sumário

Descompactar um fluxo de bits MP3 é um processo bem trabalhoso, sendo sem dúvida a etapa que exige mais linhas de código na decodificação. As tabelas de Huffman sozinhas já ocupam cerca de 70 kilobytes, e toda a lógica de parsing e descompactação requer algumas centenas de linhas de código.

A codificação Huffman é sem dúvida uma das características mais importantes do MP3. Para um quadro lógico de 500 bytes com dois canais, a saída são 4x576 amostras (1152 por canal) com um alcance de quase 15 bits — isso antes de qualquer transformação nas amostras. Sem a codificação Huffman, um quadro lógico exigiria até 4 a 4,5 kilobytes de armazenamento — cerca de oito vezes mais.

Toda a descompactação é feita por `Unpack.hs`, que exporta duas funções: `mp3Seek` e `mp3Unpack`. Esta última é uma função auxiliar simples que combina `mp3UnpackFrame` e `mp3ParseMainData`. Ela é mais ou menos assim:

```
mp3Unpack :: MP3Bitstream -> (MP3Bitstream, Maybe MP3Data)
```

# Decodificação, passo 2: Re-quantização

Tendo descompactado com sucesso um frame, agora temos uma estrutura de dados contendo o áudio que será processado mais adiante, junto com os parâmetros de como isso deve ser feito. Aqui estão nossos tipos, ou seja, o que recebemos da função `mp3Unpack`:

```
data MP3Data = MP3Data1Channels SampleRate ChannelMode (Bool, Bool) 
                                MP3DataChunk MP3DataChunk
             | MP3Data2Channels SampleRate ChannelMode (Bool, Bool) 
                                MP3DataChunk MP3DataChunk 
                                MP3DataChunk MP3DataChunk

data MP3DataChunk = MP3DataChunk {
    chunkBlockType    :: Int,
    chunkBlockFlag    :: BlockFlag,
    chunkScaleGain    :: Double,
    chunkScaleSubGain :: (Double, Double, Double),
    chunkScaleLong    :: [Double],
    chunkScaleShort   :: [[Double]],
    chunkISParam      :: ([Int], [[Int]]),
    chunkData         :: [Int]
}  
```

`MP3Data` é simplesmente um frame lógico descompactado e interpretado. Ele contém algumas informações úteis: primeiro, a taxa de amostragem (sample rate); segundo, o modo de canal (channel mode); e terceiro, os modos de estéreo (falaremos mais sobre isso depois). 

Em seguida, temos de dois a quatro blocos de dados, que são decodificados separadamente. O que os valores armazenados em um `MP3DataChunk` representam será explicado em breve. Por enquanto, basta saber que `chunkData` armazena no máximo 576 amostras no domínio da frequência. 

Um `MP3DataChunk` também é conhecido como um *granule*, mas para evitar confusão, não vamos usar esse termo até mais adiante no artigo.

# Re-quantização

Já realizamos uma das etapas principais da decodificação de um MP3: a decodificação dos dados de Huffman. Agora, vamos realizar a segunda etapa chave – a re-quantização.

Como mencionado no capítulo sobre a audição humana, o coração da compressão MP3 é a quantização. A quantização é simplesmente a aproximação de um grande intervalo de valores com um conjunto menor de valores, ou seja, usando menos bits. Por exemplo, se você pegar um sinal de áudio analógico e amostrado em intervalos discretos de tempo, você obterá um sinal discreto – uma lista de amostras. Como o sinal analógico é contínuo, essas amostras serão valores reais. Se quantizarmos as amostras, por exemplo, aproximando cada amostra de valor real com um número inteiro entre –32767 e +32767, terminamos com um sinal digital – discreto nas duas dimensões.

A quantização pode ser usada como uma forma de compressão com perdas. Para PCM de 16 bits, cada amostra no sinal pode assumir um dos 216 valores possíveis. Se, em vez disso, aproximarmos cada amostra no intervalo de –16383 a +16383, perdemos informações, mas economizamos 1 bit por amostra. A diferença entre o valor original e o valor quantizado é conhecida como erro de quantização, e isso resulta em ruído. A diferença entre uma amostra de valor real e uma amostra de 16 bits é tão pequena que é inaudível para a maioria das pessoas, mas, se removermos informações demais da amostra, a diferença entre o original logo se tornará audível.

Vamos parar por um momento e pensar sobre de onde vem esse ruído. Isso exige uma visão matemática, devido a Fourier: todos os sinais contínuos podem ser criados pela soma de senos – até mesmo a onda quadrada! Isso significa que, se pegarmos uma onda senoidal pura, digamos a 440 Hz, e a quantizarmos, o erro de quantização se manifestará como novos componentes de frequência no sinal. Isso faz sentido – a senóide quantizada não é realmente uma senóide pura, então deve haver algo mais no sinal. Essas novas frequências estarão por todo o espectro, e isso é ruído. Se o erro de quantização for pequeno, a magnitude do ruído será pequena.

E é aqui que podemos agradecer à evolução: nosso ouvido não é perfeito. Se houver um sinal forte dentro de uma faixa crítica, o ruído devido aos erros de quantização será mascarado até o limiar. O codificador pode, assim, jogar fora o máximo possível de informações das amostras dentro da faixa crítica, até o ponto em que descartar mais informações resultaria em ruído ultrapassando o limiar audível. Essa é a chave da codificação de áudio com perdas.

Os métodos de quantização podem ser expressos como expressões matemáticas. Digamos que temos uma amostra de valor real no intervalo de –1 a 1. Para quantizar esse valor para uma forma adequada para um arquivo WAV de 16 bits, multiplicamos a amostra por 32727 e descartamos a parte fracionária: `q = floor(s * 32767)` ou, de forma equivalente, em uma forma com a qual muitos programadores estão familiarizados: `(short)(s * 32767.0)`. A re-quantização, neste caso simples, é uma divisão, onde a diferença entre a amostra re-quantizada e a original é o erro de quantização.

# Requantização no MP3

Após desembrulharmos o fluxo de bits do MP3 e decodificarmos as amostras de frequência em um bloco usando Huffman, acabamos com amostras de frequência quantizadas entre -8206 e 8206. Agora é hora de requantizar essas amostras para valores reais (flutuantes), como quando pegamos uma amostra PCM de 16 bits e a transformamos em um valor flutuante. Quando terminamos, temos uma amostra na faixa de -1 a 1, muito menor que 8206. No entanto, nossa nova amostra tem uma resolução muito maior, graças às informações que o codificador deixou no quadro sobre como a amostra deve ser reconstruída.

O codificador MP3 usa um quantizador não linear, o que significa que a diferença entre valores consecutivos requantizados não é constante. Isso ocorre porque sinais de baixa amplitude são mais sensíveis ao ruído e, portanto, exigem mais bits do que sinais mais fortes – pense nisso como usar mais bits para valores pequenos e menos bits para valores grandes. Para alcançar essa não linearidade, as diferentes quantidades de escalonamento são não lineares.

O codificador primeiro aumenta todas as amostras por 3/4, ou seja, `novasample = antigasample * 3/4`. O objetivo é, segundo a literatura, tornar a relação sinal-ruído mais consistente. Vamos simplificar e apenas aumentar todas as amostras por 4/3 para restaurar as amostras ao seu valor original.

Todas as 576 amostras são então escalonadas por uma quantidade conhecida como ganho, ou ganho global, porque todas as amostras são afetadas. Isso é `chunkScaleGain`, e também é um valor não linear.

Até aqui, não fizemos nada realmente incomum. Pegamos um valor, no máximo 8206, e o escalonamos com uma quantidade variável. Isso não é tão diferente de um arquivo WAV PCM de 16 bits, onde pegamos um valor, no máximo 32767, e o escalonamos com a quantidade fixa 1/32767. Agora as coisas ficam mais interessantes.

Algumas regiões de frequência, divididas em várias faixas de fatores de escala, são ainda mais escalonadas individualmente. Para isso servem os fatores de escala: as frequências na primeira faixa de fatores de escala são todas multiplicadas pelo primeiro fator de escala, e assim por diante. As faixas são projetadas para aproximar as bandas críticas. Aqui está uma ilustração das larguras de banda dos fatores de escala para um MP3 de 44100 Hz. O leitor atento pode perceber que existem 22 faixas, mas apenas 21 fatores de escala. Isso é uma limitação de design que afeta as frequências muito altas.

> sem a imagem :(

A razão para essas faixas serem escalonadas individualmente é para controlar melhor o ruído de quantização. Se houver um sinal forte em uma faixa, ele mascarará o ruído dessa faixa, mas não das outras. Os valores dentro de uma faixa de fator de escala são, portanto, quantizados independentemente das outras faixas pelo codificador, dependendo dos efeitos de mascaramento.

Por razões que serão esclarecidas em breve, um bloco pode ser escalonado de três maneiras diferentes.

Para um tipo de bloco – chamado "longo" – escalonamos as 576 frequências pelo ganho global e pelos 21 fatores de escala (`chunkScaleLong`), e deixamos assim.

Para outro tipo de bloco – chamado "curto" – as 576 amostras são na verdade três conjuntos entrelaçados de 192 amostras de frequência. Não se preocupe se isso não fizer sentido agora, falaremos sobre isso em breve. Neste caso, as faixas de fatores de escala têm uma aparência ligeiramente diferente da ilustração acima, para acomodar as larguras de banda reduzidas das faixas de fatores de escala. Além disso, os fatores de escala não são 21 números, mas conjuntos de três números (`chunkScaleShort`). Um parâmetro adicional, `chunkScaleSubGain`, escala ainda mais os três conjuntos individuais de amostras.

O terceiro tipo de bloco é uma mistura dos dois tipos acima.

Quando multiplicamos cada amostra pelo fator de escala correspondente e outros ganhos, ficamos com uma representação de ponto flutuante de alta precisão do domínio de frequência, onde cada amostra está na faixa de -1 a 1.

Aqui está um código que usa quase todos os valores de um `MP3DataChunk`. Os três métodos de escalonamento diferentes são controlados pela `BlockFlag`. Haverá muito mais informações sobre a flag de bloco mais adiante neste artigo.

```
mp3Requantize :: SampleRate -> MP3DataChunk -> [Frequency]
mp3Requantize samplerate (MP3DataChunk bt bf gain (sg0, sg1, sg2) 
                         longsf shortsf _ compressed)
    | bf == LongBlocks  = long
    | bf == ShortBlocks = short
    | bf == MixedBlocks = take 36 long ++ drop 36 short
    where 
        long  = zipWith procLong  compressed longbands
        short = zipWith procShort compressed shortbands

        procLong sample sfb = 
            let localgain   = longsf !! sfb
                dsample     = fromIntegral sample
            in gain * localgain * dsample **^ (4/3)

        procShort sample (sfb, win) =
            let localgain = (shortsf !! sfb) !! win
                blockgain = case win of 0 -> sg0
                                        1 -> sg1
                                        2 -> sg2
                dsample   = fromIntegral sample
            in gain * localgain * blockgain * dsample **^ (4/3)
                                
        -- Frequency index (0-575) to scale factor band index (0-21).
        longbands = tableScaleBandIndexLong samplerate
        -- Frequency index to scale factor band index and window index (0-2).
        shortbands = tableScaleBandIndexShort samplerate
```

Um aviso justo: Esta apresentação da etapa de requantização do MP3 difere um pouco da especificação oficial. A especificação apresenta a quantização como uma longa fórmula baseada em quantidades inteiras. Este decodificador, por outro lado, trata essas quantidades inteiras como representações de ponto flutuante de quantidades não lineares, para que a requantização possa ser expressa como uma série intuitiva de multiplicações. O resultado final é o mesmo, mas a intenção é, esperamos, mais clara.

Passo menor: Reordenamento
Antes de quantizar as amostras de frequência, o codificador, em alguns casos, vai reordenar as amostras de uma maneira pré-definida. Já encontramos isso acima: após o reordenamento pelo codificador, os "blocos" curtos com três pequenos blocos de 192 amostras cada são combinados em 576 amostras ordenadas por frequência (de certa forma). Isso é feito para melhorar a eficiência da codificação Huffman, já que o método com valores grandes e tabelas diferentes assume que as frequências mais baixas estão primeiro na lista.

Quando terminarmos de re-quantizar no nosso decodificador, vamos reordenar as amostras "curtas" de volta para sua posição original. Após esse reordenamento, as amostras nesses blocos não estão mais ordenadas por frequência. Isso é um pouco confuso, então, a menos que você tenha um grande interesse em MP3, pode ignorar isso e se concentrar nos blocos "longos", que têm poucas surpresas.

# Decodificação, passo 3: Stereo Combinado (Joint Stereo)

O MP3 suporta quatro modos diferentes de canal. Mono significa que o áudio possui um único canal. Stereo significa que o áudio tem dois canais. Canal duplo é idêntico ao modo estéreo para fins de decodificação – é destinado a informar ao reprodutor de mídia caso os dois canais contenham áudios diferentes, como um audiolivro em dois idiomas.

Em seguida, temos o stereo combinado. Isso é como o modo estéreo regular, mas com algumas etapas extras de compressão, levando em consideração as semelhanças entre os dois canais. Isso faz sentido, especialmente para músicas estéreo, onde geralmente há uma correlação muito alta entre os dois canais. Ao remover alguma redundância, a qualidade do áudio pode ser muito maior para uma taxa de bits dada.

O MP3 suporta dois modos de stereo combinado conhecidos como *middle/side stereo* (MS) e *intensity stereo* (IS). Se esses modos estão em uso, é indicado pela tupla (Bool, Bool) no tipo MP3Data. Além disso, o parâmetro *chunkISParam* armazena o parâmetro usado pelo modo IS.

O stereo MS é muito simples: em vez de codificar dois canais semelhantes literalmente, o codificador calcula a soma e a diferença dos dois canais antes da codificação. O conteúdo de informação no canal “lateral” (diferença) será menor do que no canal “central” (soma), e o codificador pode usar mais bits para o canal central, resultando em um resultado melhor. O stereo MS é sem perdas e é um modo muito comum, frequentemente utilizado em MP3s com stereo combinado. A decodificação do stereo MS é bem interessante:

```
mp3StereoMS :: [Frequency] -> [Frequency] -> ([Frequency], [Frequency])
mp3StereoMS middle side =
    let sqrtinv = 1 / (sqrt 2)
        left  = zipWith0 (\x y -> (x+y)*sqrtinv) 0.0 middle side
        right = zipWith0 (\x y -> (x-y)*sqrtinv) 0.0 middle side
    in (left, right)
```

A única peculiaridade aqui é a divisão pela raiz quadrada de 2, em vez de simplesmente 2. Isso é feito para reduzir a escala dos canais para uma quantização mais eficiente pelo codificador.

Um modo de estéreo mais incomum é conhecido como estéreo de intensidade, ou IS para abreviar. Vamos ignorar o estéreo IS neste artigo.

Após a decodificação do estéreo, a única coisa que resta é levar as amostras de frequência de volta para o domínio do tempo. Esta é a parte mais teórica.

Decodificação, passo 4: Frequência para tempo

Neste ponto, os únicos valores restantes de MP3DataChunk que vamos usar são chunkBlockFlag e chunkBlockType. Estes são os dois únicos parâmetros que determinam como vamos transformar nossas amostras do domínio da frequência para o domínio do tempo. Para entender o bloco de flag e o tipo de bloco, precisamos nos familiarizar com algumas transformações, assim como com uma parte do codificador.

# O codificador: bancos de filtros e transformações

A entrada para um codificador é provavelmente um arquivo WAV PCM no domínio do tempo, como o que normalmente se obtém ao ripar um CD de áudio. O codificador pega 576 amostras de tempo, daqui em diante chamadas de "granulação", e codifica duas dessas granulações em um quadro. Para uma fonte de entrada com dois canais, duas granulações por canal são armazenadas no quadro. O codificador também salva informações sobre como o áudio foi comprimido no quadro. Este é o tipo MP3Data em nosso decodificador.

As amostras no domínio do tempo são transformadas para o domínio da frequência em várias etapas, uma granulação por vez.

## Banco de filtros de análise

Primeiro, as 576 amostras são alimentadas em um conjunto de 32 filtros passa-faixa, onde cada filtro passa-faixa gera 18 amostras do domínio do tempo representando 1/32 do espectro de frequência do sinal de entrada. Se a taxa de amostragem for 44100 Hz, cada banda terá aproximadamente 689 Hz de largura (22050/32 Hz). Note que há um processo de redução de amostras aqui: filtros passa-faixa comuns geram 576 amostras de saída para 576 amostras de entrada, mas os filtros MP3 também reduzem o número de amostras em 32, então a saída combinada de todos os 32 filtros tem o mesmo número de entradas.

Esta parte do codificador é conhecida como o banco de filtros de análise (adicionar o termo "polyphase" para maior precisão), e é uma parte do codificador comum a todas as camadas MPEG-1. Nosso decodificador fará o processo reverso no final do processo de decodificação, combinando as sub-bandas ao sinal original. O reverso é conhecido como o banco de filtros de síntese. Esses dois bancos de filtros são simples conceitualmente, mas verdadeiros "mamutes" matematicamente – pelo menos o banco de filtros de síntese. Vamos tratá-los como caixas pretas.

## MDCT

A saída de cada filtro passa-faixa é transformada ainda mais pela MDCT, a Transformada Discreta de Cosseno Modificada. Esta transformação é apenas um método de transformar as amostras do domínio do tempo para o domínio da frequência. A camada 1 e 2 não utilizam essa MDCT, mas ela foi adicionada ao topo do banco de filtros para a camada 3, pois uma resolução de frequência mais fina que 689 Hz (dada uma taxa de amostragem de 44,1 kHz) provou ser mais eficiente na compressão. Isso faz sentido: simplesmente dividir todo o espectro de frequência em blocos de tamanho fixo significa que o decodificador precisa considerar várias bandas críticas ao quantizar o sinal, o que resulta em uma pior taxa de compressão.

A MDCT pega um sinal e o representa como uma soma de ondas cosseno, transformando-o para o domínio da frequência. Comparada com a DFT/FFT e outras transformadas bem conhecidas, a MDCT possui algumas propriedades que a tornam muito adequada para compressão de áudio.

Primeiramente, a MDCT tem a propriedade de compactação de energia comum a várias outras transformadas de cosseno discretas. Isso significa que a maior parte da informação no sinal está concentrada em poucas amostras de saída com alta energia. Se você pegar uma sequência de entrada, fizer uma transformação (M)DCT nela, definir os valores de saída "pequenos" como 0 e depois fizer a transformação inversa – o resultado será uma mudança bem pequena na entrada original. Essa propriedade é claro, muito útil para compressão, e por isso diferentes transformadas de cosseno são usadas não apenas no MP3 e na compressão de áudio em geral, mas também no JPEG e nas técnicas de codificação de vídeo.

Em segundo lugar, a MDCT foi projetada para ser executada em blocos consecutivos de dados, de forma que possui discrepâncias menores nas fronteiras dos blocos em comparação com outras transformadas. Isso também a torna muito adequada para áudio, pois quase sempre estamos lidando com sinais realmente longos.

Tecnicamente, a MDCT é uma transformada chamada "sobreposta", o que significa que usamos amostras de entrada dos dados anteriores ao trabalharmos com os dados de entrada atuais. A entrada são 2N amostras de tempo e a saída são N amostras de frequência. Em vez de transformar blocos de comprimento 2N separadamente, os blocos consecutivos são sobrepostos. Esse sobreposição ajuda a reduzir os artefatos nas fronteiras dos blocos. Primeiro, realizamos a MDCT, por exemplo, nas amostras 0-35 (inclusive), depois em 18-53, depois em 36-71... Para suavizar as fronteiras entre os blocos consecutivos, a MDCT geralmente é combinada com uma função de janela que é aplicada antes da transformação. Uma função de janela é simplesmente uma sequência de valores que são zero fora de uma região, e frequentemente entre 0 e 1 dentro da região, e que deve ser multiplicada por outra sequência. Para a MDCT suave, funções de janela em arco são geralmente usadas, o que faz com que as fronteiras do bloco de entrada se aproximem suavemente de zero nas bordas.

No caso do MP3, a MDCT é aplicada às sub-bandas do banco de filtros de análise. Para obter todas as boas propriedades da MDCT, a transformação não é feita diretamente nas 18 amostras, mas em um sinal com janela formado pela concatenação das 18 amostras anteriores e atuais. Isso é ilustrado na imagem abaixo, mostrando duas granulações consecutivas (MP3DataChunk) em um canal de áudio. Lembre-se: estamos vendo o codificador aqui, o decodificador trabalha ao contrário. Esta ilustração mostra a MDCT da faixa de 0-679 Hz.

> sem a imagem :(

A MDCT pode ser aplicada às 36 amostras como descrito acima, ou três MDCTs podem ser feitas em 12 amostras cada – em qualquer dos casos, a saída será de 18 amostras de frequência. A primeira escolha, conhecida como método longo, nos dá maior resolução de frequência. A segunda escolha, conhecida como método curto, nos dá maior resolução temporal. O codificador seleciona a MDCT longa para obter melhor qualidade de áudio quando o sinal muda muito pouco, e seleciona o método curto quando há muita atividade, ou seja, para transientes.

Para a granulação inteira de 576 amostras, o codificador pode fazer a MDCT longa em todas as 32 sub-bandas – este é o modo de bloco longo, ou pode fazer a MDCT curta em todas as sub-bandas – este é o modo de bloco curto. Existe uma terceira opção, conhecida como o modo de bloco misto. Nesse caso, o codificador usa a MDCT longa nas duas primeiras sub-bandas, e a MDCT curta nas restantes. O modo de bloco misto é um compromisso: é usado quando a resolução temporal é necessária, mas o uso do modo de bloco curto resultaria em artefatos. As frequências mais baixas são assim tratadas como blocos longos, onde o ouvido é mais sensível a imprecisões de frequência. Note que as fronteiras do modo de bloco misto são fixas: as duas primeiras, e apenas as duas primeiras, sub-bandas utilizam a MDCT longa. Isso é considerado uma limitação de design do MP3: às vezes seria útil ter alta resolução de frequência em mais de duas sub-bandas. Na prática, muitos codificadores não suportam blocos mistos.

Discutimos os modos de bloco brevemente no capítulo sobre requantização e reordenação, e esperamos que essa parte faça um pouco mais de sentido agora que entendemos o que está acontecendo dentro do codificador. As 576 amostras em uma granulação curta são na verdade 3x 192 pequenas granulações, mas armazenadas de tal forma que as facilidades para comprimir uma granulação longa podem ser usadas.

A combinação do banco de filtros de análise e a MDCT é conhecida como o banco de filtros híbrido, e é uma parte muito confusa do decodificador. O banco de filtros de análise é utilizado por todas as camadas MPEG-1, mas como as bandas de frequência não refletem as bandas críticas, a camada 3 adicionou a MDCT no topo do banco de filtros de análise. Uma das características do AAC é um método mais simples para transformar as amostras do domínio do tempo para o domínio da frequência, que utiliza apenas a MDCT, sem se preocupar com os filtros passa-faixa.

# O Decodificador

Digestar essas informações sobre o codificador leva a uma realização surpreendente: na verdade, não podemos decodificar granulações, ou quadros, de forma independente! Devido à natureza sobreposta da MDCT (Transformada Discreta de Cosseno Modificada), precisamos da saída da inversa da MDCT da granulação anterior para decodificar a granulação atual.

É aqui que os parâmetros `chunkBlockType` e `chunkBlockFlag` são usados. Se o `chunkBlockFlag` for definido com o valor `LongBlocks`, o codificador usou uma única MDCT de 36 pontos para todas as 32 sub-bandas (do banco de filtros), com sobreposição da granulação anterior. Se o valor for `ShortBlocks`, em vez disso, três MDCTs menores de 12 pontos foram usadas. O `chunkBlockFlag` também pode ser `MixedBlocks`. Nesse caso, as duas sub-bandas de menor frequência do banco de filtros são tratadas como `LongBlocks` e o restante como `ShortBlocks`.

O valor de `chunkBlockType` é um número inteiro, podendo ser 0, 1, 2 ou 3. Isso decide qual janela será usada. Essas funções de janela são bastante simples e semelhantes: uma é para os blocos longos, uma é para os três blocos curtos e as duas outras são usadas exatamente antes e depois de uma transição entre um bloco longo e curto.

Antes de realizarmos a inversa da MDCT, precisamos levar em conta algumas deficiências do banco de filtros de análise do codificador. O subamostragem no banco de filtros introduz alguns aliasing (quando sinais se tornam indistinguíveis de outros sinais), mas de uma forma em que o banco de filtros de síntese cancela o aliasing. Após a MDCT, o codificador removerá parte desse aliasing. Isso, claro, significa que temos que desfazer essa redução de alias em nosso decodificador, antes da IMDCT. Caso contrário, a propriedade de cancelamento de alias do banco de filtros de síntese não funcionará.

Quando lidarmos com o aliasing, podemos aplicar a IMDCT e depois a janela, lembrando de fazer a sobreposição com a saída da granulação anterior. Para os blocos curtos, as três pequenas entradas individuais de IMDCT são sobrepostas diretamente e, então, esse resultado é tratado como um bloco longo.

A palavra "sobreposição" requer alguns esclarecimentos no contexto da transformada inversa. Quando falamos da MDCT, uma função que vai de 2N entradas para N saídas, isso significa que usamos metade das amostras anteriores como entradas para a função. Se acabamos de realizar uma MDCT com 36 amostras de entrada a partir do deslocamento 0 em uma sequência longa, então realizamos a MDCT de 36 novas amostras a partir do deslocamento 18.

Quando falamos da IMDCT, uma função que vai de N entradas para 2N saídas, há um passo adicional necessário para reconstruir a sequência original. Fazemos a IMDCT nas primeiras 18 amostras da sequência de saída acima. Isso nos dá 36 amostras. A saída 18..35 é somada, elemento por elemento, à saída 0..17 da IMDCT das próximas 18 amostras. Aqui está uma ilustração.

> sem a imagem aqui :(

Com isso resolvido, aqui está um código:

```
mp3IMDCT :: BlockFlag -> Int -> [Frequency] -> [Sample] -> ([Sample], [Sample])
mp3IMDCT blockflag blocktype freq overlap =
    let (samples, overlap') = 
            case blockflag of
                 LongBlocks  -> transf (doImdctLong blocktype) freq
                 ShortBlocks -> transf (doImdctShort) freq
                 MixedBlocks -> transf (doImdctLong 0)  (take 36 freq) <++>
                                transf (doImdctShort) (drop 36 freq)
        samples' = zipWith (+) samples overlap
    in (samples', overlap')
    where
        transf imdctfunc input = unzipConcat $ mapBlock 18 toSO input
            where
                -- toSO takes 18 input samples b and computes 36 time samples
                -- by the IMDCT. These are further divided into two equal
                -- parts (S, O) where S are time samples for this frame
                -- and O are values to be overlapped in the next frame.
                toSO b = splitAt 18 (imdctfunc b)
                unzipConcat xs = let (a, b) = unzip xs
                                 in (concat a, concat b)

doImdctLong :: Int -> [Frequency] -> [Sample]
doImdctLong blocktype f = imdct 18 f `windowWith` tableImdctWindow blocktype

doImdctShort :: [Frequency] -> [Sample]
doImdctShort f = overlap3 shorta shortb shortc
  where
    (f1, f2, f3) = splitAt2 6 f
    shorta       = imdct 6 f1 `windowWith` tableImdctWindow 2
    shortb       = imdct 6 f2 `windowWith` tableImdctWindow 2
    shortc       = imdct 6 f3 `windowWith` tableImdctWindow 2
    overlap3 a b c = 
      p1 ++ (zipWith3 add3 (a ++ p2) (p1 ++ b ++ p1) (p2 ++ c)) ++ p1
      where
        add3 x y z = x+y+z
        p1         = [0,0,0, 0,0,0]
        p2         = [0,0,0, 0,0,0, 0,0,0, 0,0,0]
```

Antes de passarmos o sinal no domínio do tempo para o banco de filtros de síntese, há um último passo. Algumas sub-bandas do banco de filtros de análise possuem espectros de frequência invertidos, que o codificador corrige. Temos que desfazer isso, assim como a redução de aliasing.

Aqui estão os passos necessários para levar nossas amostras de frequência de volta ao tempo:

- [Frequência] Desfazer a redução de aliasing, levando em conta o sinalizador do bloco.
- [Frequência] Realizar a IMDCT, levando em conta o sinalizador do bloco.
- [Tempo] Inverter os espectros de frequência para algumas bandas.
- [Tempo] Banco de filtros de síntese.

Um decodificador típico de MP3 passará a maior parte de seu tempo no banco de filtros de síntese – é, de longe, a parte mais pesada computacionalmente do decodificador. Em nosso decodificador, usaremos a implementação (mais lenta) da especificação. Decodificadores típicos do mundo real, como o do seu reprodutor de mídia favorito, utilizam uma versão altamente otimizada do banco de filtros, utilizando uma transformação de forma inteligente. Não vamos aprofundar nesta técnica de otimização.

# Passo 4, resumo

É fácil não ver a floresta por causa das árvores, mas devemos lembrar que esse passo de decodificação é conceitualmente simples; ele é apenas confuso no MP3 porque os designers reutilizaram partes da camada 1, o que torna as fronteiras entre o domínio do tempo, o domínio da frequência e os grânulos menos claras.
