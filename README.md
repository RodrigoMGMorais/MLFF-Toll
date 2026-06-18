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
