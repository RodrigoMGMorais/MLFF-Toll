# Estrutura de Dados e Estratégias de Consulta (SQL / Auditoria)

Este documento descreve as premissas de modelagem e as consultas SQL analíticas utilizadas no ecossistema do Back Office (BOS) para auditoria operacional, identificação de erros críticos e relatórios de faturamento.

## 1. Índices Críticos e Performance de Gravação

Dada a alta concorrência de escritas por segundo em um sistema MLFF (Multi-Lane Free Flow), a tabela principal de transações (`tb_passagem`) adota estratégias estritas de indexação para evitar *Deadlocks* e otimizar buscas do Back Office:

* **Chave Primária:** UUID ou ID Sequencial associado ao Timestamp.
* **Índices Compostos Recomendados:**
    * `idx_passagem_placa_data` -> `(ds_placa, dt_passagem)` (Otimiza buscas no Portal MeuPedágio).
    * `idx_passagem_tag_status` -> `(cd_tag, id_status_transacao)` (Otimiza processamento do InteropFlex).

## 2. Queries de Auditoria Operacional (Troubleshooting)

### Consulta A: Identificação de Potencial Dessincronização de Relógio ou Clonagem (Erro 701)
Esta query analisa passagens consecutivas da mesma TAG em pórticos distintos em janelas de tempo curtas incompatíveis com os limites de velocidade da rodovia.

```sql
SELECT 
    p1.cd_tag,
    p1.id_portico AS portico_origem,
    p1.dt_passagem AS hora_origem,
    p2.id_portico AS portico_destino,
    p2.dt_passagem AS hora_destino,
    EXTRACT(EPOCH FROM (p2.dt_passagem - p1.dt_passagem)) AS intervalo_segundos
FROM tb_passagem p1
INNER JOIN tb_passagem p2 ON p1.cd_tag = p2.cd_tag 
    AND p2.dt_passagem > p1.dt_passagem
    AND p1.id_portico <> p2.id_portico
WHERE p1.dt_passagem >= NOW() - INTERVAL '1 HOUR'
  AND EXTRACT(EPOCH FROM (p2.dt_passagem - p1.dt_passagem)) < 120 -- Menos de 2 minutos entre pórticos distantes
ORDER BY p1.cd_tag, p1.dt_passagem;



