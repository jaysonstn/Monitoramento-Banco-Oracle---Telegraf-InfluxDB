# Monitoramento Oracle 19c com Telegraf

## Visão Geral

Este documento descreve o processo de configuração do monitoramento do banco de dados Oracle 19c,, utilizando o Telegraf para coletar métricas via SQLPlus e enviá-las ao InfluxDB.

---

## Pré-requisitos

- Telegraf 1.37.2 [📥 Download Telegraf v1.37.2](https://github.com/influxdata/telegraf/releases/tag/v1.37.2) instalado e rodando como serviço Windows (Utilizo esta versão pelo fato de já rodar essa versão em outros ambientes)
- Oracle 19c instalado na mesma VM
- SQLPlus disponível no PATH do sistema
- Usuário Windows com permissão para executar o SQLPlus
- InfluxDB rodando

---

## 1. Criação do Usuário de Monitoramento no Oracle

Conecte ao banco como SYSDBA e execute:

```sql
CREATE USER telegraf IDENTIFIED BY sua_senha;
GRANT CREATE SESSION TO telegraf;
GRANT SELECT ON v_$session TO telegraf;
GRANT SELECT ON v_$sga TO telegraf;
GRANT SELECT ON v_$pgastat TO telegraf;
GRANT SELECT ON v_$sysstat TO telegraf;
GRANT SELECT ON dba_data_files TO telegraf;
GRANT SELECT ON dba_free_space TO telegraf;
```

> O usuário `telegraf` possui apenas permissões de leitura, sem risco de alteração no banco.

Para remover o usuário futuramente:

```sql
DROP USER telegraf CASCADE;
```

---

## 2. Script SQL de Coleta

Crie o arquivo `oracle_metrics.sql` em:

```
C:\Program Files\InfluxData\telegraf\oracle_metrics.sql
```

Conteúdo:
* Obs: Todos os ** hostname ** devem ser substituidos pelo hostname real do servidor.

```sql
SET PAGESIZE 0 FEEDBACK OFF VERIFY OFF HEADING OFF ECHO OFF TRIMSPOOL ON
SET LINESIZE 1000

-- Sessões ativas (usuários executando algo no momento)
SELECT 'oracle_sessions,host=*hostname* active=' || COUNT(*)
FROM v$session WHERE type='USER' AND status='ACTIVE';

-- Total de conexões abertas (ativas + inativas)
SELECT 'oracle_connections,host=*hostname* total=' || COUNT(*)
FROM v$session WHERE type='USER';

-- Tablespaces: uso percentual, espaço livre e total em MB
SELECT 'oracle_tablespace,host=*hostname*,tablespace=' || t.tablespace_name
    || ' used_pct=' || ROUND((1 - NVL(f.free,0)/t.total)*100,2)
    || ',free_mb=' || ROUND(NVL(f.free,0)/1048576,2)
    || ',total_mb=' || ROUND(t.total/1048576,2)
FROM
    (SELECT tablespace_name, SUM(bytes) total FROM dba_data_files GROUP BY tablespace_name) t
    LEFT JOIN (SELECT tablespace_name, SUM(bytes) free FROM dba_free_space GROUP BY tablespace_name) f
    ON t.tablespace_name = f.tablespace_name;

-- SGA: componentes da System Global Area
SELECT 'oracle_sga,host=*hostname*,component=' || REPLACE(name,' ','_')
    || ' value=' || value
FROM v$sga;

-- PGA: Program Global Area
SELECT 'oracle_pga,host=*hostname*,component=' || REPLACE(name,' ','_')
    || ' value=' || value
FROM v$pgastat
WHERE name IN (
    'total PGA allocated',
    'total PGA used for auto workareas',
    'maximum PGA allocated'
);

-- Cache Hit Ratio: eficiência do buffer cache
SELECT 'oracle_cache,host=*hostname* hit_ratio=' ||
    ROUND((1 - (phyrds / NULLIF(dbgets + consistent_gets, 0))) * 100, 2)
FROM (
    SELECT
        SUM(CASE name WHEN 'physical reads' THEN value END) phyrds,
        SUM(CASE name WHEN 'db block gets' THEN value END) dbgets,
        SUM(CASE name WHEN 'consistent gets' THEN value END) consistent_gets
    FROM v$sysstat
    WHERE name IN ('physical reads','db block gets','consistent gets')
);

EXIT;
```

### Por que `SET LINESIZE 1000`?

Sem essa configuração o SQLPlus quebra linhas longas em múltiplas linhas, corrompendo o formato line protocol do InfluxDB e causando erros de parse no Telegraf.

### Por que um único script com todas as queries?

O SQLPlus abre e fecha apenas **uma vez** por coleta, executando todas as queries em sequência. Isso minimiza o consumo de CPU, memória e conexões ao banco — especialmente importante em ambientes com alto uso de recursos.

---

## 3. Configuração do Logon do Serviço Telegraf

O serviço do Telegraf roda por padrão com a conta **Local System**, que não tem acesso ao SQLPlus do Oracle. É necessário alterar para um usuário Windows com permissão:

1. Abra `services.msc`
2. Localize o serviço **Telegraf**
3. Clique com botão direito → **Propriedades**
4. Aba **Logon** → selecione **Esta conta**
5. Informe o usuário e senha com permissão para executar o SQLPlus
6. Clique em **OK**
7. Reinicie o serviço

---

## 4. Configuração do `telegraf.conf`

Adicione o bloco `inputs.exec` ao `telegraf.conf`:

```toml
[[inputs.exec]]
  commands = [
    "cmd /c sqlplus -S telegraf/sua_senha@nome_banco @\"C:\\Program Files\\InfluxData\\telegraf\\oracle_metrics.sql\""
  ]
  timeout = "30s"
  interval = "300s"
  data_format = "influx"
```

### Detalhes importantes

| Parâmetro | Valor | Motivo |
|-----------|-------|--------|
| `cmd /c` | Obrigatório | Contorna o problema de aspas no caminho com espaços no Windows |
| `timeout` | `30s` | Tempo máximo para o SQLPlus executar e retornar |
| `interval` | `300s` | Coleta a cada 5 minutos — minimiza impacto em VM com alto uso de RAM |
| `data_format` | `influx` | Formato line protocol gerado pelo script SQL |

### Reiniciar o serviço após a alteração

Execute o PowerShell como Administrador:

```powershell
cd "C:\Program Files\InfluxData\telegraf"
.\telegraf.exe --service stop
.\telegraf.exe --service start
```

---

## 5. Queries Flux no Grafana
* Obs: Todos os ** hostname ** devem ser substituidos pelo hostname real do servidor.

### Sessões Ativas

```flux
from(bucket: "Senior")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "oracle_sessions")
  |> filter(fn: (r) => r._field == "active")
  |> filter(fn: (r) => r.host == "*hostname*")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

### Total de Conexões Abertas

```flux
from(bucket: "Senior")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "oracle_connections")
  |> filter(fn: (r) => r._field == "total")
  |> filter(fn: (r) => r.host == "*hostname*")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

### Tablespace — Uso (%)

```flux
from(bucket: "Senior")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "oracle_tablespace")
  |> filter(fn: (r) => r._field == "used_pct")
  |> filter(fn: (r) => r.host == "*hostname*")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
```

### Tablespace — Espaço Livre (MB)

```flux
from(bucket: "Senior")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "oracle_tablespace")
  |> filter(fn: (r) => r._field == "free_mb")
  |> filter(fn: (r) => r.host == "*hostname*")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
```

### Tablespace — Total (MB)

```flux
from(bucket: "Senior")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "oracle_tablespace")
  |> filter(fn: (r) => r._field == "total_mb")
  |> filter(fn: (r) => r.host == "*hostname*")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
```

### SGA por Componente (GB)

```flux
from(bucket: "Senior")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "oracle_sga")
  |> filter(fn: (r) => r._field == "value")
  |> filter(fn: (r) => r.host == "*hostname*")
  |> map(fn: (r) => ({r with _value: r._value / 1073741824.0}))
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
```

### PGA por Componente (GB)

```flux
from(bucket: "Senior")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "oracle_pga")
  |> filter(fn: (r) => r._field == "value")
  |> filter(fn: (r) => r.host == "*hostname*")
  |> map(fn: (r) => ({r with _value: r._value / 1073741824.0}))
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
```

### Cache Hit Ratio (%)

```flux
from(bucket: "Senior")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "oracle_cache")
  |> filter(fn: (r) => r._field == "hit_ratio")
  |> filter(fn: (r) => r.host == "*hostname*")
  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)
```

---

## 6. Unidades Recomendadas no Grafana

| Painel | Tipo | Unidade |
|--------|------|---------|
| Sessões ativas | Stat | `short` |
| Total conexões | Stat | `short` |
| Tablespace uso | Gauge | `Percent (0-100)` |
| Tablespace livre/total | Table | `Megabytes` |
| SGA / PGA | Time series | `gigabytes` |
| Cache Hit Ratio | Gauge | `Percent (0-100)` |

> Para painéis de tablespace, SGA e PGA use o **Display name**: `${__field.labels.tablespace}` ou `${__field.labels.component}` para exibir o nome de cada item.

---

## 7. Measurements Gerados no InfluxDB

| Measurement | Campo | Descrição |
|-------------|-------|-----------|
| `oracle_sessions` | `active` | Quantidade de sessões de usuário ativas |
| `oracle_connections` | `total` | Total de conexões abertas |
| `oracle_tablespace` | `used_pct`, `free_mb`, `total_mb` | Uso de espaço por tablespace |
| `oracle_sga` | `value` | Componentes da System Global Area |
| `oracle_pga` | `value` | Alocação da Program Global Area |
| `oracle_cache` | `hit_ratio` | Eficiência do buffer cache (%) |
