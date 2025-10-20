---
title: SQL Server Extended Events — Debug e Trace Avançado
parent: Microsoft SQL Server
nav_order: 1
tags: [SQL Server, Performance, Troubleshooting, Extended Events, DBA, Ocendeck Docs]
---

# 🔍 SQL Server Extended Events — Debug e Trace Avançado em Tempo Real

## 🧭 Introdução

Em ambientes de missão crítica, é comum precisar **entender o que o SQL Server está realmente executando**, principalmente quando há suspeita de lentidão ou sobrecarga causada por aplicações externas.

O **Extended Events (XEvent)** é a ferramenta nativa do SQL Server para esse tipo de diagnóstico — muito mais leve e flexível que o antigo **SQL Profiler**.

Com ele, é possível:

- Monitorar **chamadas de stored procedures** e **queries diretas**;
- Capturar **textos SQL completos**, **tempo de execução**, **hostname**, **aplicação**, **usuário**, e **duração (ms)**;
- Armazenar resultados **na memória (ring_buffer)** ou **em arquivo (event_file)**;
- Ler os eventos em tempo real, **sem impactar** a performance.

---

## ⚙️ 1. Criando uma Sessão de Debug (Trace) via `ring_buffer`

A sessão `ring_buffer` armazena os eventos **na memória do SQL Server**.  
É ideal para **debug temporário** e inspeção imediata (curto prazo).

### 🧩 Criação

```sql
IF EXISTS (SELECT 1 FROM sys.server_event_sessions WHERE name = 'XE_Debug')
  DROP EVENT SESSION XE_Debug ON SERVER;
GO

CREATE EVENT SESSION XE_Debug ON SERVER
ADD EVENT sqlserver.rpc_completed(
  ACTION(sqlserver.sql_text, sqlserver.client_app_name, sqlserver.client_hostname)
  WHERE (database_name = 'SeuBancoDeDados')  -- opcional, para filtrar um BD
),
ADD EVENT sqlserver.sql_batch_completed(
  ACTION(sqlserver.sql_text, sqlserver.client_app_name, sqlserver.client_hostname)
  WHERE (database_name = 'SeuBancoDeDados')
)
ADD TARGET package0.ring_buffer;
GO

ALTER EVENT SESSION XE_Debug ON SERVER STATE = START;
GO
```

> ✅ **Dica:** O filtro `WHERE (database_name = 'SeuBancoDeDados')` reduz ruído e impacto.  
> Se quiser capturar tudo, basta removê-lo.

---

### 🔎 Leitura do Buffer (em tempo real)

```sql
;WITH rb AS (
  SELECT 
    CAST(t.target_data AS XML) AS x
  FROM sys.dm_xe_sessions s
  JOIN sys.dm_xe_session_targets t
    ON s.address = t.event_session_address
  WHERE s.name = 'XE_Debug'
    AND t.target_name = 'ring_buffer'
)
SELECT
  DATEADD(ms,
          -1*(DATEDIFF(ms, GETUTCDATE(), SYSDATETIMEOFFSET())),
          xevt.value('@timestamp','datetime2'))                 AS ts,
  xevt.value('(data[@name="object_name"]/value)[1]','sysname') AS object_name,
  xevt.value('(action[@name="sql_text"]/value)[1]','nvarchar(max)') AS sql_text,
  xevt.value('(data[@name="duration"]/value)[1]','bigint')/1000.0   AS duration_ms
FROM rb
CROSS APPLY rb.x.nodes('/RingBufferTarget/event') AS q(xevt)
ORDER BY ts DESC;
```

> 📈 Dica: Cada evento indica o **texto SQL**, **tempo de execução** e a **origem (app, host)**.

---

### 🧹 Encerrar e excluir a sessão (importante)

```sql
ALTER EVENT SESSION XE_Debug ON SERVER STATE = STOP;
DROP EVENT SESSION XE_Debug ON SERVER;
```

> 🔻 Sempre pare e apague as sessões quando terminar.  
> Sessões ativas consomem memória, especialmente as de buffer circular.

---

## 💾 2. Criando uma Sessão Persistente (Arquivo `.xel`)

A sessão `event_file` grava os eventos em disco — ideal para **análise posterior**, auditoria ou comparação de performance em períodos longos.

### 🧩 Criação

```sql
IF EXISTS (SELECT 1 FROM sys.server_event_sessions WHERE name = 'XE_FileTrace')
  DROP EVENT SESSION XE_FileTrace ON SERVER;
GO

CREATE EVENT SESSION XE_FileTrace ON SERVER
ADD EVENT sqlserver.rpc_completed(
  ACTION(sqlserver.sql_text, sqlserver.client_app_name, sqlserver.client_hostname)
  WHERE (database_name = 'SeuBancoDeDados')
),
ADD EVENT sqlserver.sql_batch_completed(
  ACTION(sqlserver.sql_text, sqlserver.client_app_name, sqlserver.client_hostname)
  WHERE (database_name = 'SeuBancoDeDados')
)
ADD TARGET package0.event_file(
  SET filename = N'C:\XE\XE_FileTrace',     -- caminho do arquivo
      max_file_size = (50),                 -- MB por arquivo
      max_rollover_files = (5)              -- quantidade de arquivos mantidos
);
GO
ALTER EVENT SESSION XE_FileTrace ON SERVER STATE = START;
GO
```

> ⚠️ Crie a pasta (`C:\XE\`) e garanta que o **serviço do SQL Server** tenha **permissão de escrita**.

---

### 🔎 Leitura do arquivo `.xel`

```sql
SELECT
  DATEADD(ms,-1*(DATEDIFF(ms,GETUTCDATE(),SYSDATETIMEOFFSET())),
          CAST(event_data AS XML).value('(event/@timestamp)[1]','datetime2')) AS ts,
  CAST(event_data AS XML).value('(event/data[@name="object_name"]/value)[1]','sysname') AS object_name,
  CAST(event_data AS XML).value('(event/action[@name="sql_text"]/value)[1]','nvarchar(max)') AS sql_text,
  CAST(event_data AS XML).value('(event/data[@name="duration"]/value)[1]','bigint')/1000.0 AS duration_ms
FROM sys.fn_xe_file_target_read_file('C:\XE\XE_FileTrace*.xel', NULL, NULL, NULL)
ORDER BY ts DESC;
```

> 🗂️ Os arquivos podem ser abertos também pelo **SQL Server Management Studio** (SSMS → “File → Open → File…” → selecione o `.xel`).

---

### 🧹 Encerrar e excluir

```sql
ALTER EVENT SESSION XE_FileTrace ON SERVER STATE = STOP;
DROP EVENT SESSION XE_FileTrace ON SERVER;
```

---

## 🧭 3. Gerenciamento e Administração

### 🔍 Listar todas as sessões existentes

```sql
SELECT name, startup_state, event_retention_mode, max_memory, max_dispatch_latency, event_file_path =
    (SELECT TOP 1 t.target_name
     FROM sys.dm_xe_session_targets t
     WHERE t.event_session_address = s.address)
FROM sys.dm_xe_sessions s
ORDER BY name;
```

### 🔎 Listar as **sessões criadas (mesmo que paradas)**

```sql
SELECT name, event_session_id, startup_state
FROM sys.server_event_sessions
ORDER BY name;
```

### 🧩 Ver targets (onde armazenam os dados)

```sql
SELECT
  s.name AS SessionName,
  t.target_name AS TargetType,
  CAST(t.target_data AS XML) AS TargetData
FROM sys.dm_xe_sessions s
JOIN sys.dm_xe_session_targets t
  ON s.address = t.event_session_address;
```

---

## ❓ Perguntas Frequentes

### ➤ “Essas sessões são por banco ou por servidor?”
- São **por instância do SQL Server**, não por banco.
- Você pode filtrar por banco via `WHERE (database_name = 'MeuBanco')`.
- Portanto, **impactam a instância inteira** enquanto ativas (mesmo que só capturem um banco).

### ➤ “Isso afeta a performance?”
- Muito pouco — o impacto é **mínimo** quando filtrado por banco ou app.  
- Evite deixar sessões rodando indefinidamente.  
- Sempre `STOP` e `DROP` após o uso.

### ➤ “Posso deixar uma sessão ativa permanente?”
- Sim, mas use `event_file` com limites definidos (`max_file_size`, `max_rollover_files`) e monitore espaço em disco.
- Não é recomendado usar `ring_buffer` em sessões permanentes.

---

## 🧰 4. Boas Práticas

| Item | Recomendação |
|------|---------------|
| 🎯 **Filtro** | Sempre filtrar por banco (`database_name`) ou app (`client_app_name`) |
| 🧩 **Target** | Use `ring_buffer` para debug rápido e `event_file` para histórico |
| ⏱️ **Tempo de vida** | Pare e apague (`STOP` + `DROP`) após análise |
| 💾 **Permissões** | Garanta `VIEW SERVER STATE` ao login que vai ler |
| 🧹 **Limpeza** | Revise sessões antigas com `sys.server_event_sessions` |
| 🧩 **Arquivo** | Evite salvar logs em `C:\Program Files`; prefira `C:\XE\` ou drive de dados |

---

## 🚀 Exemplo prático de uso (troubleshooting real)

Imagine que um sistema web está lento, mas a query SQL em si é rápida no SSMS.

Você cria uma sessão rápida:

```sql
CREATE EVENT SESSION XE_AppDebug ON SERVER
ADD EVENT sqlserver.rpc_completed(
  ACTION(sqlserver.sql_text, sqlserver.client_hostname, sqlserver.client_app_name)
  WHERE (database_name = 'affinin_cob')
)
ADD TARGET package0.ring_buffer;
ALTER EVENT SESSION XE_AppDebug ON SERVER STATE = START;
```

Abre a tela do sistema → lê o buffer:

```sql
;WITH rb AS (...)
SELECT ts, sql_text, duration_ms
FROM rb
ORDER BY ts DESC;
```

Descobre que há **centenas de queries pequenas** sendo chamadas em sequência.  
Conclusão: o problema está no código da aplicação (N+1 lookups), não no SQL.  
Resultado: performance resolvida sem tocar no banco.

---

## 🧭 Conclusão

O **Extended Events** é o método mais limpo e profissional de diagnosticar comportamento do SQL Server em tempo real, **sem depender de ferramentas externas**.  

Saber usá-lo diferencia um **desenvolvedor comum** de um **arquiteto de missão crítica**, pois ele revela *o que realmente acontece entre a aplicação e o banco*.

---

### ✍️ Autor
**Inventtis | Ocendeck Cloud Engineering**  
*Inspire. What inspires you to invent?*  

📘 Publicado em: [https://docs.ocendeck.com/sqlserver/xevents-trace-debug](https://docs.ocendeck.com/sqlserver/xevents-trace-debug)
