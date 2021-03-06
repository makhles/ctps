# Carteira de Trabalho e Previdência Social

Implementação de um contrato inteligente escrito na linguagem Solidity que representa uma Carteira de Trabalho. Testes de execução foram feitos na IDE Remix.

---
## Contextualização

Identificam-se três entidades que interagem entre si neste projeto:
* O dono da CTPS, referenciado no código por _empregado_.
* O INSS, órgão regulamentador e que controla a Previdência Social.
* O _empregador_, representando uma pessoa física ou jurídica.

Considera-se cada uma destas entidades terá acesso às informações que lhes cabem através de interfaces (por exemplo um app de celular ou uma interface web) com a plataforma Ethereum.


---
## Configuração da IDE Remix
Selecionou-se o compilador `0.4.24+commit.e67f0147` na aba _Settings_ do Remix para a realização dos testes.


---
## Contratos

A carteira de trabalho criada possui dois contratos do Ethereum:
* `CTPS` - contrato que representa a Carteira de Trabalho de uma pessoa.
* `Contrato` - contrato que representa um Contrato de Trabalho firmado entre duas partes, o empregado e o empregador.


---
## Requisitos Atendidos

As seções a seguir explicam a implementação e o funcionamento dos requisitos estabelecidos no documento de requisitos.

---
### `RF01` - Criação da Carteira de Trabalho

A criação da Carteira de Trabalho pode feita em alguma das instalações da Previdência Social através da interface do INSS. Supõe-se a existência de uma aplicação que possua um formulário para inserção dos dados da pessoa. Esta aplicação pode chamar o construtor do contrato CTPS e passar o endereço da conta da pessoa no Ethereum e o _hash_ dos seus dados pessoais (vide subseção abaixo):
```java
    CTPS.constructor(address _empregado, uint8 _dummy, address _dadosPessoais) public {
        previdenciaSocial = msg.sender;
        empregado = _empregado;
        dadosPessoais = _dadosPessoais;
    }
```

Obs.: o parâmetro `dummy` é necessário devido a um _bug_ que encontrei que não permite a instanciação de um contrato passando dois parâmetros do tipo `address` em sequência. Não se sabe se o _bug_ está presente somente no Remix ou se é alguma restrição da linguagem.


#### Hash dos Dados Pessoais

O arquivo [dados_pessoais](./dados_pessoais) contém alguns dados fictícios utilizados como exemplo na geração da carteira de trabalho. A geração dos dados foi feita através do site [4devs](https://www.4devs.com.br/).

Para se calcular o hash desses dados, deve-se utilizar uma função de dispersão criptográfica, tal como o [SHA-1](https://en.wikipedia.org/wiki/SHA-1 "Wikipedia: SHA-1"). Pode-se utilizar o programa `sha1sum`, disponível na maioria dos sistemas Unix. Fornecendo-se o arquivo com os dados pessoais, este programa produz um resumo de mensagem de 160 bits (20 bytes), na forma de um número hexadecimal de 40 dígitos:
```sh
% sha1sum dados_pessoais
0c6c57e6e93725646e60bb23308a054e8870aa9c  dados_pessoais
```
Após adicionar o prefixo `0x`, este número pode ser usado no construtor da CTPS e como parâmetro da função `alterarDadosPessoais`.


---
### `RF02` e `RF03` - Alteração dos dados pessoais

Percebe-se que o endereço do INSS é armazenado na carteira de trabalho em seu construtor. Isto é necessário para permitir a alteração dos dados pessoais do dono da CTPS e para que o órgão tenha acesso ao tempo de serviço do dono da carteira. 
Assim como é feito em uma CTPS física, a alteração dos dados pessoais deverá ser feita pela Previdência Social. Para tal, foi definido o método `alterarDadosPessoais`, que recebe o novo _hash_:
```java
    function CTPS.alterarDadosPessoais(address _dadosPessoais) public onlyBy(previdenciaSocial) {
        dadosPessoais = _dadosPessoais;
    }
``` 

---
### `RF04` - Solicitação de Firma de Contrato

Um empregador pode fazer uma solicitação de firma de contrato com uma pessoa ao chamar o método `solicitarContrato`. O método recebe o parâmetro `_info`, que representa os termos do contrato:
```java
    function CTPS.solicitarContrato(string _info) public {
        Contrato c = new Contrato(this, _info, msg.sender, 0, inss);
        uint indice = solicitacoes.push(c) - 1;
        emit SolicitacaoContrato(indice);
    }

    Contrato.constructor(address _ctps, string _info, address _empregador, uint8 _dummy, address _inss) public {
        ctps = _ctps;
        empregador = _empregador;
        info = _info;
        inss = _inss;
    }

    event SolicitacaoContrato(uint _indice);
```
Cria-se um novo `Contrato` com os endereços das três entidades especificadas anteriormente. Entretanto, este contrato não ainda está firmado pelo empregado e é armazenado no arranjo dinâmico `CTPS.solicitacoes`. Um evento contendo o índice do contrato no arranjo é emitido, permitindo, assim, que o dono da carteira possa aceitar ou rejeitar a solicitação quando o evento surgir em sua interface.


---
### `RF05` - Aceite & Rejeição de Firma de Contrato

Ao receber o evento de solicitação de firma de contrato em sua interface, o dono da CTPS poderá aceitar firmar o contrato através do método `firmarContrato`. O índice da solicitação deverá ser passado como parâmetro:
```java
    function CTPS.firmarContrato(uint _indice) public acesso(empregado) {
        require (_indice < solicitacoes.length, "Índice inválido.");
        Contrato c = Contrato(solicitacoes[_indice]);
        c.firmar();
        contratos.push(c);
        removerElemento(solicitacoes, _indice);
        emit ContratoFirmado(contratos[contratos.length-1]);
    }

    function Contrato.firmar() public {
        require(msg.sender == ctps, "Acesso negado.");
        require(dataAdmissao == 0, "Este contrato já foi firmado.");
        dataAdmissao = block.timestamp;
    }
```
 O contrato é adicionado no arranjo de contratos do trabalhador, `contratos`, e removido do arranjo de solicitações. O contrato estará oficialmente firmado no momento da mineração do bloco, `block.timestamp`. Finalmente, emite-se o evento `ContratoFirmado` para que o empregador e o INSS obtenham uma referência ao contrato criado e possam acessar suas funcionalidades.

Semelhantemente, o dono da carteira de trabalho pode decidir-se por não aceitar a proposta de trabalho, e a solicitação será removida do arranjo de solicitações:
```java
    function CTPS.rejeitarSolicitacao(uint _indice) public acesso(empregado) {
        removerElemento(solicitacoes, _indice);
    }
```

---
### `RF06` - Rescisão de um Contrato

Esta funcionalidade permite que tanto o empregado quanto o empregador rescindam o contrato firmado entre si. O empregado pode rescindir um contrato passando o índice do contrato para o método `rescindirContrato`. O contrato continua na lista de contratos para fins de cálculo de tempo de serviço, porém sua variável de estado `dataRescisao` é atualizada no momento da transação:
```java
    function CTPS.rescindirContrato(uint _indice) public acesso(empregado) {
        require (_indice < contratos.length, "Índice inválido.");
        Contrato c = Contrato(contratos[_indice]);
        c.rescindir();
    }

    function Contrato.rescindir() public {
        require(msg.sender == ctps || msg.sender == empregador, "Acesso negado.");
        require(dataAdmissao != 0, "Este contrato ainda não foi firmado.");
        dataRescisao = block.timestamp;
    }
```
O empregador pode rescindir o contrato chamando o método `Contrato.rescindir` a partir de sua interface.

---
### `RF07` - Adição de Licenças

A adição de licenças é feita pelo empregador diretamente no contrato de trabalho. Conforme visto na seção __`RF05` - Aceite & Rejeição de Firma de Contrato__, quando um contrato é firmado pelo empregado, emite-se o evento `ContratoFirmado` com o endereço do contrato de trabalho. O empregador, que certamente estará observando esse evento, poderá armazenar esse endereço em sua interface. Posteriormente, esse endereço poderá ser usado para a adição de licenças através do método `adicionarLicenca`:
```java
    function Contrato.adicionarLicenca(uint _tipo, uint _inicio, uint _termino) public {
        require(msg.sender == empregador, "Acesso negado.");
        require(_tipo <= uint(TipoLicenca.NAO_REMUNERADA), "Tipo de licença inexistente.");
        require(_termino > _inicio, "A data de término deve ser posterior à data de início.");
        Licenca memory licenca = Licenca(TipoLicenca(_tipo), _inicio, _termino);
        licencas.push(licenca);
    }

    struct Contrato.Licenca {
        TipoLicenca tipo;
        uint inicio;
        uint termino;
    }

```
Definiu-se a estrutura `Licenca` para armazenar os dados, na qual `TipoLicenca` é o enumerador `TipoLicenca { MATERNIDADE, PATERNIDADE, CASAMENTO, OBITO, MILITAR, NAO_REMUNERADA }`. Destes, apenas licençás do tipo `NAO_REMUNERADA` são excluídas do tempo de serviço calculado para a aposentadoria.


---
### `RF08` - Adição de Afastamentos

A adição de afastamentos na CTPS só poderá ser feita pelo órgão regulador, que neste caso é o INSS. Assim como no caso anterior, a inserção é feita diretamente no contrato de trabalho em que houve o afastamento:
```java
    function Contrato.adicionarAfastamento(string _motivo, uint _inicio, uint _termino, bool _integra) public {
        require(msg.sender == inss, "Acesso negado.");
        require(_termino > _inicio, "A data de término deve ser posterior à data de início.");
        Afastamento memory periodo = Afastamento(_motivo, _inicio, _termino, _integra);
        afastamentos.push(periodo);
    }
```
Além das datas de início e término do afastamento, considerou-se colocar também o motivo do afastamento que, assim como as informações do contrato (armazenadas em `info`), pode ser um texto cifrado cuja decifragem somente poderá ser feita nas interfaces do empregado, do empregador e do INSS, e também uma flag que indica se o período de afastamento deve contar para a integralização da aposentadoria:
```java
    struct Contrato.Afastamento {
        string motivo;
        uint inicio;
        uint termino;
        bool integra;
    }
```


---
### `RF09` - Adição de Férias


Da mesma forma que no requisito `RF07`, cabe ao empregador adicionar os períodos de férias que o empregado gozou:
```java
    function Contrato.adicionarFerias(uint _inicio, uint _termino) public {
        require(msg.sender == empregador, "Acesso negado.");
        require(_termino > _inicio, "A data de término deve ser posterior à data de início.");
        Ferias memory periodoFerias = Ferias(_inicio, _termino);
        ferias.push(periodoFerias);
    }

    struct Contrato.Ferias {
        uint inicio;
        uint termino;
    }
```

---
### `RF10` - Cálculo do Tempo Total

O empregado pode calcular o tempo total de trabalho em todos os seus contratos de trabalho chamando o método `tempoAposentadoria`:

```java
    function CTPS.tempoAposentadoria() public view returns (uint) {
        require(msg.sender == empregado || msg.sender == inss, "Acesso negado.");
        
        uint total = 0;

        for (uint i = 0; i < contratos.length; i++) {
            Contrato c = Contrato(contratos[i]);
            total = total + c.tempoAposentadoria();
        }

        return total;
    }
```

Em `Contrato.tempoAposentadoria()`, calcula-se o tempo total máximo daquele contrato pela diferença entre a data de rescisão do contrato e a data de admissão. Caso o contrato ainda esteja vigente, então utiliza-se o tempo marcado no bloco minerado. A seguir, deduz-se desse valor o tempo em que o empregado esteve de licença não-remunerada (ou seja, houve uma suspensão no contrato de trabalho, ao invés de uma interrupção deste). Finalmente, deduz-se também os períodos de afastamento que não integram para a aposentadoria:
```java
    function Contrato.tempoAposentadoria() public view returns (uint) {
        require(msg.sender == empregado || msg.sender == empregador || msg.sender == inss, "Acesso negado.");
        require(dataAdmissao != 0, "Este contrato ainda não foi firmado.");

        uint i;
        uint total = 0;

        // Tempo total do contrato
        if (dataRescisao != 0) {
            total = dataRescisao - dataAdmissao;
        } else {
            total = block.timestamp - dataAdmissao;
        }

        // Remoção dos períodos gozados por licenças que não são consideradas para
        // a integração da aposentadoria pelo INSS
        for (i = 0; i < licencas.length; i++) {
            if (licencas[i].tipo == TipoLicenca.NAO_REMUNERADA) {
                total = total - (licencas[i].termino - licencas[i].inicio);
            }
        }

        // Remoção dos períodos de afastamento que não integram na aposentadoria
        for (i = 0; i < afastamentos.length; i++) {
            if (!afastamentos[i].integra) {
                total = total - (afastamentos[i].termino - afastamentos[i].inicio);
            }
        }
        return total;
    }
```
