# Manual Completo do MCIBr (Motor de Cálculo de Impostos Brasileiro)

Bem-vindo ao **Manual Definitivo do MCIBr**. Este documento foi elaborado para ser a referência técnica completa para desenvolvedores que utilizam o framework MCIBr em seus projetos Delphi e Lazarus. Ele abrange desde a arquitetura fundamental até os detalhes de implementação de cada tributo suportado, incluindo o Sistema Tributário atual, o Simples Nacional e a Reforma Tributária (IVA Dual).

---

## Índice

1.  [Visão Geral e Arquitetura](#1-visão-geral-e-arquitetura)
2.  [Instalação e Configuração](#2-instalação-e-configuração)
3.  [Impostos sobre o Consumo (Sistema Atual)](#3-impostos-sobre-o-consumo-sistema-atual)
    *   [3.1 ICMS - Imposto sobre Circulação de Mercadorias e Serviços](#31-icms)
    *   [3.2 ICMS ST - Substituição Tributária](#32-icms-st)
    *   [3.3 ICMS ST Efetivo (CST 60 / CSOSN 500)](#33-icms-st-efetivo)
    *   [3.4 ICMS ST Destinatário (Antecipação)](#34-icms-st-destinatário)
    *   [3.5 IPI - Imposto sobre Produtos Industrializados](#35-ipi)
    *   [3.6 PIS e COFINS](#36-pis-e-cofins)
    *   [3.7 ISSQN - Imposto Sobre Serviços](#37-issqn)
    *   [3.8 Simples Nacional (Anexos e Crédito)](#38-simples-nacional)
4.  [Reforma Tributária (O Novo Sistema 2026+)](#4-reforma-tributária-o-novo-sistema-2026)
    *   [4.1 IBS - Imposto sobre Bens e Serviços](#41-ibs)
    *   [4.2 CBS - Contribuição sobre Bens e Serviços](#42-cbs)
    *   [4.3 IS - Imposto Seletivo](#43-is-imposto-seletivo)
5.  [Tributos Acessórios e Especiais](#5-tributos-acessórios-e-especiais)
    *   [5.1 IBPT (Lei da Transparência)](#51-ibpt)
    *   [5.2 FCP - Fundo de Combate à Pobreza](#52-fcp)
    *   [5.3 DIFAL e Partilha de ICMS](#53-difal-e-partilha-de-icms)
    *   [5.4 II - Imposto de Importação](#54-ii)
    *   [5.5 CPP - Contribuição Previdenciária Patronal](#55-cpp)
    *   [5.6 CIDE](#56-cide)
    *   [5.7 Recursos Avançados (Devolução, Diferimento, Redução)](#57-recursos-avançados)
6.  [Exemplo Prático Completo (Nota Fiscal)](#6-exemplo-prático-completo-nota-fiscal)
7.  [Referência de Propriedades de Retorno](#7-referência-de-propriedades-de-retorno)
8.  [Referência de Testes Unitários](#8-referência-de-testes-unitários)

---

## 1. Visão Geral e Arquitetura

O MCIBr não é apenas uma calculadora; é um **motor de regras de negócio**. Ele foi desenhado para desacoplar a complexidade fiscal do restante da aplicação (ERP), garantindo que as regras fiscais sejam aplicadas de forma consistente e centralizada.

### Conceitos Chave
*   **Motor (`IImpostoMotor`):** O cérebro que orquestra os cálculos. Ele detém as configurações globais da operação (Interestadual, Consumidor Final, Tipo de Ambiente, etc.).
*   **Nota Fiscal (`INotaFiscal`):** O contêiner de dados. Armazena totais, emitente e destinatário.
*   **Emitente (`IEmitente`):** Define quem está vendendo. Fundamental para determinar o Regime Tributário (CRT - Simples Nacional vs Normal) e a UF de origem.
*   **Destinatário (`IDestinatario`):** Define quem está comprando. Crítico para cálculos de DIFAL (Consumidor Final? Contribuinte?) e substituição tributária.
*   **Produto (`IProduto`):** A entidade atômica. Cada produto possui instâncias de interfaces para cada imposto (`IICMS`, `IIBS`, `IPIS`, etc.).
*   **Interface Fluente:** O uso intenso de interfaces (`IInterface`) permite um código limpo e evita vazamento de memória (no Delphi, com exceção de referências circulares, e nativo no ARC).

---

## 2. Instalação e Configuração

Não há componentes visuais para instalar na paleta. O MCIBr é uma biblioteca de código (Library) focada em performance e compatibilidade.

1.  **Clone o repositório:** `git clone https://github.com/isaquepsp/MCIBr.git`
2.  **Library Path:** Adicione a pasta `Source` e todas as suas subpastas ao *Library Path* da sua IDE (Delphi ou Lazarus).
3.  **Uses:** Em sua unit, adicione:
    ```delphi
    uses mcibr.interfaces, mcibr.motor, mcibr.produto, mcibr.enum.cstcsosn, mcibr.enum.regime;
    ```

---

## 3. Impostos sobre o Consumo (Sistema Atual)

### 3.1. ICMS
**Definição:** Imposto estadual sobre a circulação de mercadorias. É não-cumulativo, o que significa que o imposto pago na entrada gera crédito para abater na saída.

**Implementação no MCIBr:**
O framework implementa as regras de CST (Regime Normal) e CSOSN (Simples Nacional) automaticamente baseando-se nas propriedades configuradas.

**Exemplo Prático (CST 00 - Tributada Integralmente):**
```delphi
var
  LMotor: IImpostoMotor;
  LProd: IProduto;
begin
  LMotor := TImpostoMotor.New;
  LMotor.TipoOperacao := doInterna; // Venda dentro do estado
  
  LProd := LMotor.NotaFiscal.AddProduto;
  LProd.PrecoUnitario := 1000.00;
  LProd.Quantidade := 1;
  
  // Configuração ICMS
  LProd.ICMS.CSTCSOSN := cst00;
  LProd.ICMS.Origem := omNacional;
  LProd.ICMS.AliquotaInterna := 18.00; // Alíquota do Estado
  
  LProd.Processar;
  
  // Resultado: Base=1000, Valor=180
  Writeln(LProd.ICMS.AsValor); 
end;
```

### 3.2. ICMS ST
**Definição:** Ocorre quando a responsabilidade pelo recolhimento do imposto de toda a cadeia é atribuída a um único contribuinte (geralmente a indústria). Utiliza-se a MVA (Margem de Valor Agregado) para estimar o preço final.

**Exemplo Prático (CST 10 - Tributada com ST):**
```delphi
// Configuração ICMS Próprio
LProd.ICMS.CSTCSOSN := cst10;
LProd.ICMS.AliquotaInterna := 18.00;

// Configuração ICMS ST
LProd.ICMS.ICMSST.Modalidade := mdMargemValorAgregadoICMS;
LProd.ICMS.ICMSST.MargemValorAgregado := 50.00; // MVA 50%
LProd.ICMS.ICMSST.Aliquota := 18.00; // Alíquota Interna do Destino
LProd.ICMS.ICMSST.BaseReducao := 0.00;

LProd.Processar;

// O motor calcula o ICMS Próprio e o ICMS ST separadamente
// Base ST estimada = 1000 + 50% = 1500
// Valor ST = (1500 * 18%) - ValorICMSProprio
```

### 3.3. ICMS ST Efetivo
**Definição:** Utilizado em operações com CST 60 ou CSOSN 500, onde o imposto foi cobrado anteriormente por Substituição Tributária. O cálculo do "efetivo" serve para demonstrar o quanto de imposto estaria embutido na operação caso ela fosse tributada, exigido por alguns estados (ex: RS, MG) para fins de restituição ou complemento.

**Exemplo Prático (CST 60):**
```delphi
LProd.ICMS.CSTCSOSN := CSTICMS60;
LProd.ICMS.ICMSEfetivo.Aliquota := 18.00;
LProd.ICMS.ICMSEfetivo.BaseReducao := 0.00;
// O sistema calculará o valor efetivo com base no valor líquido do item
```

### 3.4. ICMS ST Destinatário
**Definição:** Refere-se à antecipação do imposto quando a mercadoria entra no estado sem a retenção do ST na origem. O destinatário deve recolher este valor.

```delphi
LProd.ICMS.ICMSSTDest.Aliquota := 18.00;
LProd.ICMS.ICMSSTDest.BaseCalculo := 100.00; // Percentual da base (se houver redução)
// O cálculo considera o valor líquido mais despesas acessórias
```

### 3.5. IPI
**Definição:** Imposto federal sobre produtos industrializados.

**Exemplo Prático:**
```delphi
LProd.IPI.CST := ipi50; // Saída Tributada
LProd.IPI.Aliquota := 10.00;

// O IPI compõe a base do ICMS? Depende da configuração e finalidade (consumo vs revenda)
// O MCIBr trata isso através das regras de negócio internas.
```

### 3.6. PIS e COFINS
**Definição:** Contribuições sociais federais. Podem ser Cumulativas (Lucro Presumido) ou Não-Cumulativas (Lucro Real).

**Recurso Importante: Exclusão do ICMS da Base (Tese do Século)**
O MCIBr suporta nativamente a exclusão do ICMS da base de cálculo do PIS/COFINS.

```delphi
LProd.PIS.CST := pis01;
LProd.PIS.Aliquota := 1.65;
LProd.PIS.ValorBaseExcluirValorICMS := True; // Ativa a exclusão

LProd.COFINS.CST := cofins01;
LProd.COFINS.Aliquota := 7.60;
LProd.COFINS.ValorBaseExcluirValorICMS := True;
```

### 3.7. ISSQN
**Definição:** Imposto municipal sobre serviços.

```delphi
LProd.ISSQN.Aliquota := 5.00;
LProd.ISSQN.BaseReducao := 0.00;
// Base = Valor do Serviço - Deduções Legais
```

### 3.8. Simples Nacional
O MCIBr possui suporte completo ao Simples Nacional, tanto para cálculo do imposto devido (Anexos) quanto para o destaque de crédito (CSOSN 101/201).

**Cálculo pelo Anexo:**
```delphi
LMotor.NotaFiscal.Emitente.RegimeTributario := rtbSimplesNacional;
LProd.ICMS.AnexoSN.Aliquota := 6.00; // Alíquota efetiva da faixa do Simples
// O sistema calculará o valor a recolher na DAS correspondente a esta operação
```

**Cálculo de Crédito (CSOSN 101):**
```delphi
LProd.ICMS.CSTCSOSN := CSOSN101;
LProd.ICMS.CREDITOSN.Aliquota := 2.00; // Alíquota de crédito permitida
// Resultado disponível em LProd.ICMS.CREDITOSN.AsValor
```

---

## 4. Reforma Tributária (O Novo Sistema 2026+)

A partir de 2026, o Brasil inicia a transição para o sistema de **IVA Dual**. O MCIBr já implementa as interfaces e cálculos para este novo cenário.

### Conceito de "Imposto por Fora"
Diferente do ICMS/PIS/COFINS, o IBS e a CBS não compõem a sua própria base de cálculo. O valor do produto é o valor líquido, e os impostos são adicionados a ele.

### 4.1. IBS
**Definição:** Imposto sobre Bens e Serviços (Substitui ICMS e ISS). Competência compartilhada entre Estados e Municípios.

**Exemplo de Implementação:**
```delphi
// Ativar modo Reforma Tributária
TVigenciaReformaTributaria.PermitirOverride := True;
TVigenciaReformaTributaria.UsarNovosImpostos := True;

LProd.IBS.SetCST(IBSCST000); // Tributado Integralmente
LProd.IBS.SetBaseUF(100.00);
LProd.IBS.SetAliquotaUF(17.5); // Alíquota de referência Estadual
LProd.IBS.SetBaseMUN(100.00);
LProd.IBS.SetAliquotaMUN(2.0); // Alíquota de referência Municipal
```

### 4.2. CBS
**Definição:** Contribuição sobre Bens e Serviços (Substitui PIS, COFINS e IPI parcial). Competência Federal.

```delphi
LProd.CBS.SetCST(CBSCST000);
LProd.CBS.SetBase(100.00);
LProd.CBS.SetAliquota(9.0); // Alíquota estimada Federal
```

### 4.3. IS (Imposto Seletivo)
**Definição:** Sobretaxa para produtos nocivos (fumo, álcool, poluentes). Incide uma única vez (monofásico) na produção ou importação.

```delphi
LProd.ISE.SetBase(100.00);
LProd.ISE.SetAliquota(15.0);
```

---

## 5. Tributos Acessórios e Especiais

### 5.1. IBPT
Calcula a carga tributária aproximada para exibição no cupom fiscal (Lei 12.741/2012).

```delphi
LProd.IBPT.AliquotaNacional := 4.2;
LProd.IBPT.AliquotaImportado := 12.0;
LProd.IBPT.AliquotaEstadual := 18.0;
LProd.IBPT.AliquotaMunicipal := 0.0;
LProd.Processar;
// Resultado disponível em LProd.IBPT.AsValorNacional, etc.
```

### 5.2. FCP
Fundo de Combate à Pobreza. Adicional de alíquota (geralmente 2%) sobre o ICMS para produtos supérfluos.

```delphi
LProd.ICMS.FCP.Aliquota := 2.00;
// O valor do FCP é calculado sobre a mesma base do ICMS
```

### 5.3. DIFAL e Partilha de ICMS
Diferencial de Alíquota em operações interestaduais para consumidor final não contribuinte (EC 87/2015). O MCIBr lida com a complexidade da Partilha entre UF de Origem e Destino (que hoje é 100% Destino, mas o código mantém a estrutura histórica e legal).

```delphi
LMotor.TipoOperacao := doInterestadual;
LMotor.NotaFiscal.Destinatario.ConsumidorFinal := True;
LMotor.NotaFiscal.Destinatario.ContribuinteICMS := icmsNaoContribuinte;

LProd.ICMS.DIFAL.AliquotaInter := 12.00; // Origem
LProd.ICMS.DIFAL.AliquotaIntra := 18.00; // Destino
LProd.ICMS.DIFAL.BaseDupla := False; // Depende da legislação da UF destino (Lei 17.470)
```

### 5.4. II
**Definição:** Imposto de Importação.

```delphi
LProd.II.Base := 1000.00; // Valor Aduaneiro
LProd.II.Aliquota := 60.00;
// Calcula também IOF se necessário (via customização)
```

### 5.5. CPP
**Definição:** Contribuição Previdenciária Patronal. Calculada sobre o valor líquido do produto.

```delphi
LProd.CPP.Aliquota := 20.00;
// Resultado: ValorLiquido * 20%
```

### 5.6. CIDE
**Definição:** Contribuição de Intervenção no Domínio Econômico. O framework possui a estrutura de classes (`mcibr.cide.pas`) preparada para implementação de cálculos específicos (Combustíveis, Remessas, etc.), permitindo extensão pelo desenvolvedor.

### 5.7. Recursos Avançados

*   **Devolução Tributária (`IDevTrib`):** Auxilia no cálculo do valor a ser devolvido ao erário ou creditado em operações de devolução de mercadoria, aplicando o percentual sobre a base informada.
*   **Redução de Base (`IRed`):** Interface padronizada para aplicar reduções na base de cálculo de diversos impostos. O sistema valida automaticamente se a redução é maior que 100%.
*   **Diferimento (`IDif`):** Calcula o valor diferido do imposto, ou seja, a parcela do imposto cujo lançamento é postergado para uma etapa posterior da cadeia.

---

## 6. Exemplo Prático Completo (Nota Fiscal)

Este exemplo demonstra como orquestrar o motor para calcular uma Nota Fiscal completa, com múltiplos itens e cenários variados (Revenda e Substituição Tributária), destacando a importância da configuração correta do **Emitente** e **Destinatário**.

```delphi
uses
  mcibr.interfaces, mcibr.motor, mcibr.produto, 
  mcibr.enum.cstcsosn, mcibr.enum.regime, mcibr.enum.piscofins;

procedure CalcularNotaFiscalCompleta;
var
  LMotor: IImpostoMotor;
  LProd1, LProd2: IProduto;
begin
  // 1. Instanciando o Motor
  LMotor := TImpostoMotor.New;
  
  // 2. Configurando EMITENTE (Quem vende)
  // Fundamental para definir se aplica regras do Simples ou Regime Normal
  LMotor.NotaFiscal.Emitente.UFSigla := 'SP';
  LMotor.NotaFiscal.Emitente.RegimeTributario := rtbLucroPresumido; // Regime Normal
  
  // 3. Configurando DESTINATÁRIO (Quem compra)
  // Crítico para DIFAL e validação de operação interestadual
  LMotor.NotaFiscal.Destinatario.UFSigla := 'MG';
  LMotor.NotaFiscal.Destinatario.ContribuinteICMS := icmsContribuinte; // É revendedor?
  LMotor.NotaFiscal.Destinatario.ConsumidorFinal := False; // Não é consumo final
  
  // O motor detecta automaticamente se é Interestadual baseando-se nas UFs
  // SP -> MG = Interestadual
  
  // -----------------------------------------------------------------------
  // ITEM 1: Produto de Revenda (CST 00 - Tributado Integralmente)
  // -----------------------------------------------------------------------
  LProd1 := LMotor.NotaFiscal.AddProduto;
  LProd1.Item := '1';
  LProd1.CFOP := 6102; // Venda interestadual
  LProd1.PrecoUnitario := 100.00;
  LProd1.Quantidade := 10;
  LProd1.Desconto := 10.00; // Desconto no item
  
  // ICMS 
  LProd1.ICMS.CSTCSOSN := cst00;
  LProd1.ICMS.Origem := omNacional;
  LProd1.ICMS.AliquotaInterna := 12.00; // Alíquota interestadual SP->MG (12%)
  
  // PIS/COFINS (Regime Cumulativo - Lucro Presumido)
  LProd1.PIS.CST := pis01;
  LProd1.PIS.Aliquota := 0.65;
  LProd1.COFINS.CST := cofins01;
  LProd1.COFINS.Aliquota := 3.00;
  
  // -----------------------------------------------------------------------
  // ITEM 2: Produto com ST (CST 10 - Tributada com Cobrança de ST)
  // -----------------------------------------------------------------------
  LProd2 := LMotor.NotaFiscal.AddProduto;
  LProd2.Item := '2';
  LProd2.CFOP := 6403; // Venda interestadual com ST
  LProd2.PrecoUnitario := 200.00;
  LProd2.Quantidade := 5;
  
  // ICMS Próprio
  LProd2.ICMS.CSTCSOSN := cst10;
  LProd2.ICMS.Origem := omEstrangeiraImportacaoDireta;
  LProd2.ICMS.AliquotaInterna := 4.00; // Importado Interestadual (4%)
  
  // ICMS ST (Configuração da MVA e Alíquota do Destino)
  LProd2.ICMS.ICMSST.Modalidade := mdMargemValorAgregadoICMS;
  LProd2.ICMS.ICMSST.MargemValorAgregado := 50.00; // MVA Original
  LProd2.ICMS.ICMSST.Aliquota := 18.00; // Alíquota interna de MG
  LProd2.ICMS.ICMSST.BaseReducao := 0.00;
  
  // O MCIBr calcula automaticamente a MVA Ajustada se necessário,
  // baseado na diferença entre alíquota inter (4%) e intra (18%).
  
  // -----------------------------------------------------------------------
  // 4. PROCESSAMENTO E RECUPERAÇÃO DE DADOS
  // -----------------------------------------------------------------------
  // O método Processar() executa a "mágica". Ele percorre todos os itens,
  // aplica as regras do motor (Emitente/Destinatário) e preenche as propriedades "AsValor...".
  LMotor.Processar;
  
  // ONDE PEGAR OS VALORES CALCULADOS?
  // Diferente de componentes visuais, o MCIBr não joga os dados na tela.
  // Você deve ler as propriedades "As..." das interfaces.
  
  // Exemplo de leitura do Item 1:
  var
    vBaseICMS, vValorICMS: Double;
  begin
    vBaseICMS  := LProd1.ICMS.AsValorBase;  // Base de Cálculo
    vValorICMS := LProd1.ICMS.AsValor;      // Valor do Imposto
    // LProd1.ICMS.AliquotaInterna já contém a alíquota usada
  end;
  
  // Exemplo de leitura do Item 2 (ST):
  var
    vBaseST, vValorST: Double;
  begin
    vBaseST  := LProd2.ICMS.ICMSST.AsValorBase;
    vValorST := LProd2.ICMS.ICMSST.AsValor;
  end;
  
  // TOTAIS DA NOTA FISCAL
  // O Motor soma automaticamente os itens processados
  Writeln('Total Produtos: ', LMotor.NotaFiscal.TotalProdutosNF);
  Writeln('Total ICMS: ', LMotor.NotaFiscal.TotalICMS);
  Writeln('Total ST: ', LMotor.NotaFiscal.TotalICMSST);
  Writeln('Total IPI: ', LMotor.NotaFiscal.TotalIPI);
  Writeln('Total PIS: ', LMotor.NotaFiscal.TotalPIS);
  Writeln('Total COFINS: ', LMotor.NotaFiscal.TotalCOFINS);
  Writeln('Total da Nota: ', LMotor.NotaFiscal.TotalNF); // Produtos + IPI + ST + Despesas...
end;
```

---

## 7. Referência de Propriedades de Retorno

Para facilitar a integração, aqui está um resumo de onde buscar cada valor calculado após chamar `Processar()`:

| Imposto | Valor Monetário (`Double`) | Base de Cálculo (`Double`) | Outros Campos Relevantes |
| :--- | :--- | :--- | :--- |
| **ICMS** | `Prod.ICMS.AsValor` | `Prod.ICMS.AsValorBase` | `AsValorIsento`, `AsValorOutras` |
| **ICMS ST** | `Prod.ICMS.ICMSST.AsValor` | `Prod.ICMS.ICMSST.AsValorBase` | `AsValorBaseReduzida` |
| **IPI** | `Prod.IPI.AsValor` | `Prod.IPI.AsValorBase` | - |
| **PIS** | `Prod.PIS.AsValor` | `Prod.PIS.AsValorBase` | `ValorBaseExcluirValorICMS` (config) |
| **COFINS** | `Prod.COFINS.AsValor` | `Prod.COFINS.AsValorBase` | - |
| **IBS** (Reforma) | `Prod.IBS.ValorUF` + `ValorMUN` | `Prod.IBS.BaseUF` | `AliquotaUF`, `AliquotaMUN` |
| **CBS** (Reforma) | `Prod.CBS.Valor` | `Prod.CBS.Base` | `Aliquota` |
| **FCP** | `Prod.ICMS.FCP.AsValor` | `Prod.ICMS.FCP.AsValorBase` | - |
| **DIFAL** | `Prod.ICMS.DIFAL.AsValorDIFAL` | `Prod.ICMS.DIFAL.AsValorBase` | `AsValorDIFALDestinatario` |

---

## 8. Referência de Testes Unitários

Para entender profundamente como cada imposto se comporta em cenários de borda (devoluções, isenções, reduções), recomenda-se a leitura dos arquivos de teste unitário na pasta `Test Delphi`. Eles contêm cenários reais validados.

*   `test.cst00_icms.pas`: Cenários básicos de ICMS.
*   `test.cst10_icmsst.pas`: Cenários complexos de Substituição Tributária.
*   `test.csosn101_anexoSN.pas`: Simples Nacional com cálculo pelo Anexo.
*   `test.csosn101_creditoSN.pas`: Simples Nacional com destaque de crédito.
*   `test.pis_cofins.pas`: Cenários de PIS/COFINS com e sem exclusão do ICMS.
*   `test.ibs.pas` e `test.cbs.pas`: A "bíblia" da implementação da Reforma Tributária no projeto.

---

## 9. 🛡 Sistema de Validação (Validators)

> **"Como garantir que o usuário não informou um NCM inválido ou uma combinação de CST errada?"**

O MCIBr possui um poderoso sistema de validação desacoplado, baseado no padrão **Pipes and Filters**. Isso permite que você valide dados fiscais complexos sem poluir seu código com `if..then..else` infinitos.

### 1. Validadores Nativos Disponíveis
O framework já vem com validadores prontos para uso na pasta `Source\Validations`:

| Validador | Função | Arquivo |
| :--- | :--- | :--- |
| **`TValidatorPercentualMinMax`** | Garante que um valor percentual esteja entre 0 e 100. | `mcibr.validator.percentualminmax.pas` |
| **`TValidatorIsEmpty`** | Verifica se um campo string ou numérico está vazio/zerado. | `mcibr.validator.isempty.pas` |
| **`TMatrixClassTrib`** | Valida combinações de CST e Benefício Fiscal na Reforma Tributária. | `mcibr.validator.matrix.cclasstrib.pas` |

### 2. Usabilidade: Como Aplicar Validações
Você utiliza a interface `IValidationPipes` para encadear validações.

```delphi
uses 
  mcibr.validator.pipe, 
  mcibr.validator.percentualminmax,
  mcibr.interfaces;

procedure ValidarAliquota;
var
  LPipes: IValidationPipes;
  LInfo: IValidationInfo;
begin
  // 1. Cria o Pipeline
  LPipes := TValidationPipes.Create;
  
  // 2. Adiciona uma validação de Percentual (0 a 100)
  LInfo := LPipes.Add;
  LInfo.Validator := TValidatorPercentualMinMax.Create;
  LInfo.Value := 150.00; // Valor inválido propositalmente
  
  // (Opcional) Metadados para mensagem de erro rica
  LInfo.Arguments.FieldName := 'AliquotaICMS';
  LInfo.Arguments.Message := 'A alíquota não pode ser maior que 100%';
  
  // 3. Executa todas as validações
  LPipes.Validate;
  
  // 4. Verifica se houve erros
  if LPipes.IsMessages then
  begin
    // Lista todas as falhas encontradas
    ShowMessage('Erros encontrados: ' + sLineBreak + 
                string.Join(sLineBreak, LPipes.ListMessages.ToArray));
  end;
end;
```

### 3. Criação: Como Criar seu Próprio Validador
Você pode criar regras de negócio específicas para o seu ERP (ex: "Cliente do PR não pode comprar item X") e injetá-las no fluxo.

**Passo 1: Criar a Classe de Regra (`IValidatorConstraint`)**
```delphi
uses mcibr.interfaces, mcibr.validator.constrait;

type
  TValidadorPrecoMinimo = class(TValidatorConstraint)
  public
    function Validate(const Value: TValue; const Args: IValidationArguments): TResultValidation; override;
  end;

function TValidadorPrecoMinimo.Validate(const Value: TValue; Args: IValidationArguments): TResultValidation;
begin
  // Inicializa sucesso (importante chamar inherited se houver lógica base)
  inherited;
  
  // Lógica da validação
  if Value.AsExtended < 10.00 then
  begin
    Result.Success := False;
    Result.Failure := Format('O preço R$ %f está abaixo do mínimo permitido (R$ 10,00).', [Value.AsExtended]);
  end
  else
    Result.Success := True;
end;
```

**Passo 2: Usar no Pipe**
```delphi
LInfo := LPipes.Add;
LInfo.Validator := TValidadorPrecoMinimo.Create;
LInfo.Value := 5.00; // Vai falhar
```

### 4. Validação Avançada (Matrix de Regras - Reforma Tributária)
O validador `TMatrixClassTrib` é um exemplo de validação complexa que cruza dados.

```delphi
uses mcibr.validator.matrix.cclasstrib;

// Verifica se o CST 000 permite o benefício 000001
if not TMatrixClassTrib.IsValid('000', '000001') then
  Raise Exception.Create('Combinação inválida!');

// Verifica se o modelo de documento (NFE) é permitido para esta regra
if not TMatrixClassTrib.IsValidDFe('000', '000001', 'NFE') then
  Raise Exception.Create('NFE não permite este benefício.');
```

---

**MCIBr - Motor de Cálculo de Impostos Brasileiro**
