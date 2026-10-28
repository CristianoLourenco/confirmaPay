# ConfirmaPay

> **Projecto de inovação financeira — documento de concepção**  
> Versão: 0.2  
> Data: 17 de Outubro de 2026  
> Estado: Investigação e requisitos iniciais  
> Nome provisório: ConfirmaPay

## 1. Resumo

O ConfirmaPay é uma proposta de inovação que investiga e demonstra como confirmar os dados do beneficiário antes de o utilizador autorizar uma transferência instantânea. O objectivo é reduzir transferências enviadas por engano e ajudar a prevenir situações de falsa identidade.

A primeira entrega será um **protótipo interactivo com dados simulados**. Não irá movimentar dinheiro, aceder a contas reais nem apresentar dados simulados como se fossem verificados. Uma eventual integração com a infraestrutura de pagamentos angolana dependerá de validação técnica, autorização e colaboração das entidades relevantes.

## 2. Contexto e motivação

A observação inicial do projecto surgiu durante a utilização ou análise de canais bancários, incluindo o MULTICAIXA Express e/ou transferências instantâneas. A hipótese a investigar é que, em determinados fluxos, o utilizador pode não dispor de informação suficiente e fiável para confirmar o destinatário antes de autorizar a operação.

Esta observação ainda não constitui prova de que todos os canais ou fluxos angolanos omitam essa informação. O estado actual do produto deve ser verificado por meio de testes documentados, materiais oficiais e, se possível, esclarecimentos da EMIS e das instituições participantes.

A EMIS descreve o KWiK como um ecossistema integrado de pagamentos com interoperabilidade conta-a-conta e instantânea. O MULTICAIXA Express é descrito como um canal móvel interbancário associado à aplicação móvel MCX Express. Estas soluções não devem ser tratadas como o mesmo produto: a investigação deverá identificar precisamente o canal e o fluxo a que cada observação se refere. [1]

## 3. Definição do problema

Quando um utilizador inicia uma transferência através de um número ou outro identificador, pode cometer um erro de introdução ou ser induzido a enviar dinheiro para uma conta controlada por alguém que se faz passar por outra pessoa ou entidade.

Se o fluxo de pagamento não fornecer uma confirmação fiável do beneficiário antes da autorização, o utilizador pode não detectar a divergência a tempo.

**Problema de investigação:**

> Como podemos permitir que um utilizador confirme, antes de autorizar uma transferência instantânea, que os dados apresentados para o destinatário correspondem aos dados mantidos por uma fonte financeira autorizada?

O projecto aborda dois riscos distintos:

1. **Erro de destinatário:** por exemplo, introduzir um dígito incorrecto e seleccionar uma conta diferente da pretendida.
2. **Falsa identidade:** alguém convence o pagador a transferir para uma conta que não pertence à pessoa ou entidade que o pagador acredita estar a pagar.

A confirmação do nome pode ajudar a detectar divergências, mas não prova que a finalidade do pagamento seja legítima nem elimina fraudes. Uma conta pode estar correctamente identificada e ainda assim ser utilizada por um fraudador.

## 4. Objectivos

### 4.1 Objectivo geral

Conceber e validar, através de um protótipo, uma solução de confirmação do beneficiário aplicável ao contexto dos pagamentos instantâneos em Angola.

### 4.2 Objectivos específicos

- Investigar mecanismos documentados de confirmação do beneficiário em sistemas de pagamentos internacionais.
- Documentar o fluxo real observado nos canais relevantes em Angola, distinguindo factos confirmados de hipóteses.
- Identificar os dados mínimos necessários para permitir uma confirmação útil e preservar a privacidade.
- Conceber uma arquitectura que separe a consulta do beneficiário da execução da transferência.
- Desenvolver um protótipo interactivo com dados fictícios claramente identificados.
- Definir testes e métricas para avaliar a capacidade de detectar destinatários divergentes, a compreensão do utilizador e o tempo acrescentado ao fluxo.

## 5. Investigação preliminar

### 5.1 Pix — Brasil

O Banco Central do Brasil mantém o **Directório de Identificadores de Contas Transaccionais (DICT)**. O directório associa chaves Pix a dados cadastrais e a contas transaccionais, permitindo que o participante consulte informação do recebedor antes de instruir o pagamento. As chaves incluem, entre outras, número de telemóvel, CPF/CNPJ, e-mail e chave aleatória. [2][3]

Os requisitos mínimos de experiência do utilizador do Pix prevêem que a selecção de uma chave devolva informação do recebedor para conferência, incluindo o nome completo e identificadores parcialmente mascarados, e permita cancelar antes da confirmação. Também prevêem informar o utilizador quando há falha de comunicação com o directório. [4]

**Lição para o ConfirmaPay:** a resolução do identificador deve recorrer a uma fonte autorizada; a interface deve apresentar dados úteis para a conferência; e uma falha de consulta não pode ser confundida com uma validação bem-sucedida.

### 5.2 Confirmation of Payee — referência internacional

Uma nota técnica do Banco Mundial descreve o *Confirmation of Payee* (CoP) como um serviço em que o prestador de serviços de pagamento do pagador pede ao prestador do beneficiário que verifique se os dados apresentados — por exemplo, número de conta e nome — correspondem aos dados registados. A resposta pode indicar correspondência, ausência de correspondência, correspondência aproximada ou impossibilidade de verificação. [5]

**Lição para o ConfirmaPay:** a consulta pode ser modelada como um serviço distinto da transferência, com respostas normalizadas e tratamento explícito de resultados inconclusivos.

### 5.3 Contexto angolano — ponto por validar

A EMIS apresenta publicamente o KWiK como ecossistema de pagamentos conta-a-conta instantâneos e interoperáveis, e o MULTICAIXA Express como canal móvel interbancário. [1]

A pesquisa pública realizada até ao momento não permite concluir se todos os fluxos relevantes já apresentam o nome do beneficiário, em que condições isso acontece, nem que interfaces de consulta estão disponíveis para integração.

Estas perguntas ficam registadas como **questões em aberto**, não como falhas comprovadas do sistema actual.

## 6. Hipótese de solução

O ConfirmaPay irá demonstrar um fluxo em que o utilizador introduz um identificador e recebe uma resposta de consulta antes de autorizar uma transferência simulada.

Fluxo proposto:

1. O utilizador introduz o identificador do destinatário.
2. O serviço de consulta procura o identificador numa fonte de dados de demonstração.
3. O sistema devolve um resultado explícito: correspondência encontrada, não encontrada, divergência ou consulta indisponível.
4. A interface apresenta os dados permitidos para conferência e indica que se trata de uma simulação.
5. O utilizador pode confirmar ou cancelar a operação demonstrativa.
6. O protótipo regista o resultado do teste, sem iniciar qualquer transferência real.

A arquitectura de produção não será considerada definida até se conhecerem as interfaces autorizadas, os dados disponíveis, as regras aplicáveis e as responsabilidades das instituições participantes.

## 7. Âmbito da primeira versão

### Incluído

- Protótipo interactivo com dados inteiramente fictícios.
- Pesquisa por identificador simulado.
- Ecrãs de correspondência, divergência, destinatário inexistente e indisponibilidade do serviço.
- Confirmação e cancelamento antes da conclusão do fluxo demonstrativo.
- Avisos claros de que o protótipo não está ligado à EMIS, a bancos ou a sistemas de pagamento reais.
- Testes de usabilidade e registo de métricas sem recolha desnecessária de dados pessoais.

### Excluído

- Movimentação de dinheiro real.
- Consulta de contas ou dados bancários reais.
- Recolha de números de telefone ou identificadores reais de clientes.
- Integração não autorizada com a EMIS, bancos ou aplicações bancárias.
- Garantia de que um nome coincidente significa que o pagamento é seguro.
- Reversão automática de transferências ou resolução de disputas financeiras.

## 8. Requisitos funcionais iniciais

| ID | Requisito | Critério de aceitação inicial |
|---|---|---|
| RF-01 | Introduzir identificador | O utilizador pode introduzir um identificador de teste e iniciar a consulta. |
| RF-02 | Consultar dados simulados | O protótipo procura o identificador num conjunto de dados fictícios. |
| RF-03 | Apresentar correspondência | Quando há correspondência, mostra o nome fictício associado e assinala que os dados são simulados. |
| RF-04 | Tratar identificador inexistente | Apresenta uma mensagem inequívoca de que não foi encontrado um beneficiário. |
| RF-05 | Tratar divergência | Se o utilizador indicar um nome esperado para comparação, o protótipo apresenta a divergência sem declarar que a identidade foi provada. |
| RF-06 | Tratar indisponibilidade | Simula uma falha de consulta e impede que esta seja apresentada como verificação bem-sucedida. |
| RF-07 | Confirmar ou cancelar | O utilizador pode confirmar ou cancelar a operação demonstrativa antes da conclusão. |
| RF-08 | Não executar pagamentos | Nenhum percurso do protótipo envia dinheiro ou comunica com sistemas bancários reais. |
| RF-09 | Identificar a simulação | Todas as vistas com dados fictícios indicam claramente que se trata de uma demonstração. |

## 9. Requisitos não funcionais iniciais

- **Privacidade:** usar dados fictícios e evitar recolher dados pessoais desnecessários.
- **Segurança:** não guardar credenciais bancárias, PINs, códigos de autenticação ou dados reais de pagamento.
- **Clareza:** distinguir visualmente correspondência, divergência, ausência de registo e falha técnica.
- **Acessibilidade:** utilizar linguagem simples, contraste adequado e mensagens compreensíveis.
- **Rastreabilidade:** documentar decisões, pressupostos, casos de teste e limitações.
- **Extensibilidade:** manter separada a interface, a lógica de consulta e a fonte de dados simulada para permitir uma futura integração autorizada.

## 10. Critérios para avaliar o protótipo

O protótipo deverá ser testado com cenários controlados. Os resultados não devem ser apresentados como prova de redução de fraude em produção.

- **Detecção de divergências:** proporção de participantes que identificam correctamente um destinatário diferente do esperado.
- **Decisão correcta:** proporção de participantes que confirmam nos cenários de correspondência e cancelam nos cenários de divergência.
- **Tempo de tarefa:** tempo necessário para concluir cada cenário.
- **Compreensão:** proporção de participantes que entendem o significado de correspondência, divergência e consulta indisponível.
- **Falsos sinais de confiança:** participantes que interpretam uma correspondência de nome como garantia de legitimidade do pagamento.

Os valores-alvo serão definidos depois de estabelecer o protocolo de teste e a dimensão da amostra.

## 11. Questões em aberto

1. Qual é exactamente o canal e o fluxo em que a informação do beneficiário parece insuficiente?
2. Que informação é apresentada actualmente antes da confirmação: nome, instituição, parte do identificador ou outra?
3. O comportamento varia entre KWiK, MULTICAIXA Express e aplicações bancárias?
4. Existe uma API ou serviço autorizado para consultar os dados do beneficiário?
5. Quem seria responsável por manter os dados actualizados e responder por divergências?
6. Que dados podem ser apresentados sem expor informação pessoal excessiva?
7. O que deve acontecer se a consulta falhar: impedir a operação, permitir continuar com aviso ou aplicar uma política definida pela instituição?
8. Que enquadramento regulatório, contratual e de protecção de dados se aplicaria a uma implementação real?

## 12. Próximos passos

1. Documentar o fluxo observado com passos concretos e, se possível, capturas de ecrã sem dados pessoais.
2. Confirmar em fontes oficiais o comportamento actual dos canais relevantes.
3. Desenhar os ecrãs e os estados do protótipo.
4. Definir o modelo de dados fictícios e os casos de teste.
5. Implementar a demonstração e avaliar os resultados.
6. Só depois formular uma proposta de integração com a EMIS e os participantes relevantes.

## 13. Referências

[1] EMIS. *Soluções e marcas: KWiK e MULTICAIXA Express*. https://emis.ao/  
[2] Banco Central do Brasil. *Directório de Identificadores de Contas Transaccionais (DICT)*. https://www.bcb.gov.br/estabilidadefinanceira/dict  
[3] Banco Central do Brasil. *DICT API*. https://www.bcb.gov.br/estabilidadefinanceira/pagamentosinstantaneos  
[4] Banco Central do Brasil. *Requisitos mínimos para a experiência do utilizador no Pix*. https://bcb.gov.br/content/estabilidadefinanceira/pix/Regulamento_Pix/IV_RequisitosMinimosparaExperienciadoUsuario.pdf  
[5] World Bank. (2025). *Prominent Overlay Services in Fast Payment Systems: Request to Pay and Confirmation of Payee — Technical Note*. https://documents.worldbank.org/en/publication/documents-reports/documentdetail/099092425113021637

---

**Nota de rigor:** este documento separa factos sustentados por fontes públicas, observações iniciais do projecto e hipóteses por validar. Não afirma que a EMIS carece de um mecanismo de confirmação; essa conclusão exige verificação do fluxo actual e, idealmente, confirmação junto das entidades responsáveis.
