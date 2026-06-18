# Implantação e Arquitetura de Sistemas MLFF (Multi-Lane Free Flow)

<p align="center">
  <img src="https://img.shields.io/badge/Architecture-Distributed_Systems-blue?style=for-the-badge" alt="Architecture">
  <img src="https://img.shields.io/badge/Domain-ITS_%26_Tolling-orange?style=for-the-badge" alt="Domain">
  <img src="https://img.shields.io/badge/Status-Production_Ready-green?style=for-the-badge" alt="Status">
</p>

Este repositório apresenta uma visão técnica, pragmática e profunda sobre a engenharia de ponta a ponta envolvida na implantação, operação, arquitetura de sistemas e ciclo de manutenção de soluções **MLFF (Multi-Lane Free Flow)**. O foco está concentrado em sistemas de pedagiamento eletrônico em fluxo livre sob rigorosos padrões de interoperabilidade de mercado.

---

## 🚀 1. Engenharia de Implantação de Pista (Infraestrutura e RSS)

A implantação física do MLFF substitui completamente as praças de pedágio convencionais por **Pórticos Inteligentes** — estruturas metálicas elevadas sobre as faixas de rolamento. O processo de engenharia de pista divide-se em três pilares:

* **Engenharia Civil e Pavimentação:** Alinhamento geométrico da rodovia para garantir a estabilidade do fluxo dos veículos. Intervenções na pista, como a **fresagem**, são críticas para eliminar imperfeições no asfalto, evitando oscilações verticais nos veículos que possam comprometer a precisão de leitura dos sensores.
* **Instalação do RSS (Roadside System):** Montagem do ecossistema de hardware de borda (*Edge computing*) diretamente na estrutura do pórtico:
  * **Antenas DSRC/RFID (Radiofrequência):** Leitores de alta performance responsáveis pela captura e processamento dos IDs de TAGs ativas.
  * **Câmeras de OCR/ANPR:** Captura de imagens em alta resolução (dianteira e traseira dos veículos) com acionamento por infravermelho para reconhecimento óptico de caracteres (placas).
  * **Perfiladores Laser/Lidar:** Sensores ópticos aéreos tridimensionais que geram nuvens de pontos em tempo real para inferir volumetria, altura, detecção de eixos e classificação automática da categoria do veículo sem necessidade de balanças físicas.
* **Gargalo Crítico de Infraestrutura:** Instalação de servidores locais blindados na base do pórtico com redundância de conectividade via **Fibra Óptica**, associada a um sistema rigoroso de sincronização temporal via **NTP (Network Time Protocol)** para garantir que os *timestamps* de todos os sensores estejam milimetricamente idênticos.

---

## 🏗️ 2. Arquitetura de Sistemas e Integração de Dados

O fluxo de dados do MLFF baseia-se em uma arquitetura desacoplada e orientada a eventos, movendo a transação do processamento de borda (Edge) até a conciliação financeira no Back Office.

[ VEÍCULO ] 🚗💨
│
▼
[ PÓRTICO (RSS) ] ──( Sensores / OCR / Lidar / Antenas )
│
▼ (Mensageria Assíncrona: ActiveMQ / RabbitMQ)
[ BACK OFFICE CENTRAL (CBO/OBO) ]
│
├─► [ InteropFlex (IFX) ] ──► Validação de Regras (ARTESP) ──► [ OPERADORAS (OSA) ]
├─► [ TOMO ] ───────────────► Processamento de Imagens e Auditoria de Vídeo
└─► [ MeuPedágio Portal ] ──► Arrecadação de Passagens Sem TAG (PIX / Cartão)


### Fluxo de Mensageria e Back-Office

1. **Geração do Registro Bruto (RSS):** Ao cruzar o pórtico, os dados coletados (TAG, placa capturada via OCR e categoria do Lidar) são consolidados localmente no controlador de pista em um registro digital bruto de passagem.
2. **Camada de Transporte e Mensageria:** O registro é enviado via redes seguras para o barramento do Back Office utilizando filas de mensageria assíncronas de alta resiliência (ex: **ActiveMQ / RabbitMQ**). O uso de consumidores de mensageria estruturados garante tolerância a falhas e o represamento seguro dos dados em lote caso ocorra oscilação de rede.
3. **Processamento no CBO/OBO (Commercial & Operational Back Office):** O núcleo do sistema da Kapsch orquestra os submódulos especializados:
   * **InteropFlex (IFX):** O motor de interoperabilidade. Aplica as regras de negócio regulatórias (como os padrões **ARTESP**) e gerencia o envio/recebimento de mensagens com as operadoras de TAGs (OSAs como Sem Parar, Veloe, ConectCar). O IFX processa e valida as transações brutas em transações financeiras líquidas.
   * **TOMO:** Módulo especializado em Inteligência Computacional e processamento de imagens. Realiza a revisão automatizada e manual (auditoria de vídeo) de transações onde o OCR ou a classificação de categoria gerou dubiedades.
   * **Portal MeuPedágio:** Módulo de autoatendimento integrado ao BOS que atua como rede de segurança para usuários sem TAG ativa. Se a transação não encontra uma TAG ou o envio falhar definitivamente, o débito fica disponível no portal para pagamento avulso por placa via PIX/Cartão, evitando a evasão fiscal de trânsito no prazo legal.

---

## 🛠️ 3. Regras de Negócio e Tratamento de Exceções Técnicas

No ambiente de produção do MLFF, a consistência dos dados é mantida através do mapeamento estrito de códigos de retorno das operadoras financeiras. Dois cenários críticos ilustram a inteligência aplicada ao sistema:

> ⚠️ **Divergência de Categoria (Código de Erro / MotivoNaoComp: 5)**
> Ocorre quando a classificação tridimensional do Lidar (ex: caminhão de 3 eixos) diverge do cadastro da TAG retornado pela operadora (ex: carro de passeio). O sistema retém a transação no IFX e aciona uma flag de auditoria. Caso o operador ou a IA de imagem confirme o erro da TAG do usuário, o sistema executa uma ação de reenvio com categoria confirmada para forçar a tarifação correta.

> 🔴 **Horário Incompatível / Viagem Impossível (Código de Erro / MotivoNaoComp: 701)**
> Ocorre quando uma mesma TAG registra passagens em pórticos sequenciais em um intervalo de tempo fisicamente impossível de ser percorrido dentro da velocidade regulamentar da via. Tecnicamente, este erro aciona alertas severos na infraestrutura, pois frequentemente indica **dessincronização de relógio (NTP)** entre servidores de pórticos distintos, picos de represamento e envio tardio de mensagens em lote na fila do consumidor, ou suspeita de clonagem de TAG.

---

## 📊 4. Engenharia de Manutenção e Suporte Operacional

Garantir o **SLA (Service Level Agreement)** de um sistema MLFF exige monitoramento preditivo e preventivo contínuo, dividindo-se em três pilares analíticos:

* **Manutenção de Infraestrutura e Calibração:** Inspeções regulares nos lasers/Lidars do pórtico para evitar descalibrações ópticas decorrentes de trepidação estrutural. Monitoramento do desgaste do pavimento (fresagem corretiva) para manter a acurácia da volumetria.
* **Monitoramento de Sistemas (Application & Network Performance):** Uso de ferramentas de observabilidade (como **Grafana / Icinga**) para acompanhar o *throughput* das filas de mensagens, latência no processamento do `ArtespConsumer`, taxas de erro de rede e consumo de CPU/Memória dos servidores de aplicação.
* **Suporte em Banco de Dados e Logs:** Análise pragmática de logs de aplicação para rastreio de *payloads* e queries SQL estruturadas para auditoria de transações presas ou duplicadas. O foco analítico está em identificar e mitigar comportamentos anômalos que possam inflar os índices de evasão *fake* (falsos positivos de não pagamento causados por falha técnica de leitura) ou bitributação (cobrança dupla na TAG e no portal de autoatendimento).
