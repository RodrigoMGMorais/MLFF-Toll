# Arquitetura Técnica e Topologia de Redes (MLFF / BOS)

Este documento detalha o design de arquitetura, topologia e topologia de rede distribuída utilizada para sustentar o ecossistema Multi-Lane Free Flow com processamento tolerante a falhas.

## 1. Topologia de Rede e Comunicação da Borda (Edge)

Cada pórtico opera de maneira isolada no nível de pista (RSS) para garantir que oscilações na rede WAN não parem a arrecadação.

* **Camada Física de Borda (Pórtico):**
    * **Controlador Local de Pista:** Servidor industrial (*ruggedized*) rodando Linux com alta redundância de hardware (RAID 1, fontes redundantes).
    * **Rede Local (LAN do Pórtico):** Equipamentos conectados via switches industriais gerenciáveis através de protocolos redundantes (como RSTP).
* **Conectividade WAN (Pórtico ao Data Center Central):**
    * Link principal via **Anel de Fibra Óptica** da concessionária (baixa latência).
    * Failover automático via link de backup celular de dados móveis criptografados (VPN IPSec).

## 2. Padrão de Mensageria e Concorrência

O coração da integração entre o RSS e o Commercial Back Office (CBO) é assíncrono para suportar os picos de tráfego (ex: feriados prolongados), evitando perda de payloads e gargalos em bancos de dados relacionais.

* **Protocolo de Transporte:** AMQP / JMS.
* **Mecanismo de Fila:** Mensagens empacotadas em formato JSON/XML trafegando por filas dedicadas no broker (ex: `fila.passagem.bruta`).
* **Resiliência (Dead Letter Queue - DLQ):** Transações que falham sistematicamente nas validações do `ArtespConsumer` ou do `InteropFlex` devido a corrupção de payload são desviadas para uma **DLQ** para análise forense dos analistas de sistemas, garantindo que a fila principal nunca fique travada.

## 3. Estrutura Estatística do Fluxo de Dados

Abaixo está o mapeamento conceitual da pipeline de processamento:

[RSS Sensor Captures] -> [Local Edge JSON Payload] -> [WAN Criptografada]
│
▼
[ActiveMQ/RabbitMQ Queue] <------------------------ [ArtespConsumer Node Group]
│
├─► [VALIDO] ──► [InteropFlex Engine] ──► [OSA API / Cleared Payment]
│
└─► [EXCEÇÃO] ─► [TOMO (Auditoria AI/Vídeo)] ou [MeuPedágio Portal]

