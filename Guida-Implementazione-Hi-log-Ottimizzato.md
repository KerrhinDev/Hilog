# Guida all'Implementazione del Sistema Hi-log Ottimizzato

Questa guida fornisce istruzioni dettagliate per implementare il sistema Hi-log ottimizzato, migrando dall'architettura ibrida attuale (Microsoft Access + SQL Server) a una soluzione moderna completamente basata su SQL Server con API RESTful.

## Indice

1. [Prerequisiti](#prerequisiti)
2. [Configurazione dell'Ambiente](#configurazione-dellambiente)
3. [Installazione e Configurazione di SQL Server](#installazione-e-configurazione-di-sql-server)
4. [Creazione del Database](#creazione-del-database)
5. [Sviluppo dell'API RESTful](#sviluppo-dellapi-restful)
6. [Implementazione del Sistema di Caching](#implementazione-del-sistema-di-caching)
7. [Configurazione dell'Elaborazione Asincrona](#configurazione-dellelaborazione-asincrona)
8. [Sviluppo dell'Interfaccia Web](#sviluppo-dellinterfaccia-web)
9. [Migrazione dei Dati](#migrazione-dei-dati)
10. [Test e Verifica](#test-e-verifica)
11. [Messa in Produzione](#messa-in-produzione)
12. [Monitoraggio e Manutenzione](#monitoraggio-e-manutenzione)

## Prerequisiti

### Software Necessario

1. **SQL Server 2022** (o versione più recente)
   - Edizione: Standard o Enterprise
   - Download: [Microsoft SQL Server](https://www.microsoft.com/it-it/sql-server/sql-server-downloads)

2. **SQL Server Management Studio (SSMS)**
   - Download: [SSMS](https://docs.microsoft.com/it-it/sql/ssms/download-sql-server-management-studio-ssms)

3. **Visual Studio 2022** (o versione più recente)
   - Edizione: Professional o Enterprise
   - Workload: ASP.NET e sviluppo web, Elaborazione dati e archiviazione
   - Download: [Visual Studio](https://visualstudio.microsoft.com/it/downloads/)

4. **Redis**
   - Versione: 7.0 o più recente
   - Download: [Redis](https://redis.io/download)
   - Alternativa Windows: [Redis Windows](https://github.com/microsoftarchive/redis/releases)

5. **RabbitMQ**
   - Versione: 3.12 o più recente
   - Download: [RabbitMQ](https://www.rabbitmq.com/download.html)

6. **Node.js** (per lo sviluppo frontend)
   - Versione LTS
   - Download: [Node.js](https://nodejs.org/it/download/)

7. **Git**
   - Download: [Git](https://git-scm.com/downloads)

### Hardware Consigliato

- **Server Database**:
  - CPU: 16+ core
  - RAM: 64+ GB
  - Storage: SSD in configurazione RAID per prestazioni ottimali
  - Rete: 10 Gbps

- **Server Applicativi**:
  - CPU: 8+ core
  - RAM: 32+ GB
  - Storage: SSD
  - Rete: 10 Gbps

- **Server Cache**:
  - CPU: 4+ core
  - RAM: 16+ GB
  - Storage: SSD
  - Rete: 10 Gbps

## Configurazione dell'Ambiente

### 1. Creazione dell'Ambiente di Sviluppo

1. **Configurazione di Visual Studio**:
   - Aprire Visual Studio Installer
   - Selezionare "Modifica" per l'installazione esistente
   - Assicurarsi che siano installati i seguenti componenti:
     - Sviluppo ASP.NET e web
     - Elaborazione dati e archiviazione
     - Sviluppo multipiattaforma .NET Core

2. **Creazione del Repository Git**:
   ```bash
   mkdir HiLogOttimizzato
   cd HiLogOttimizzato
   git init
   ```

3. **Struttura del Progetto**:
   ```
   HiLogOttimizzato/
   ├── src/
   │   ├── HiLog.API/
   │   ├── HiLog.Core/
   │   ├── HiLog.Infrastructure/
   │   ├── HiLog.Web/
   │   └── HiLog.Worker/
   ├── tests/
   │   ├── HiLog.API.Tests/
   │   ├── HiLog.Core.Tests/
   │   └── HiLog.Infrastructure.Tests/
   ├── scripts/
   │   ├── database/
   │   └── deployment/
   └── docs/
   ```

### 2. Configurazione dell'Ambiente di Produzione

1. **Preparazione dei Server**:
   - Installare Windows Server 2022 o Linux (Ubuntu Server 22.04 LTS)
   - Configurare firewall e rete
   - Impostare aggiornamenti automatici
   - Configurare backup

2. **Configurazione del Load Balancer**:
   - Installare e configurare NGINX o HAProxy
   - Configurare SSL/TLS con certificati
   - Impostare health checks

## Installazione e Configurazione di SQL Server

### 1. Installazione di SQL Server

1. **Eseguire il programma di installazione di SQL Server**:
   - Selezionare "Installazione" > "Nuova installazione autonoma di SQL Server"
   - Selezionare l'edizione (Standard o Enterprise)
   - Selezionare le funzionalità:
     - Servizi Motore di Database
     - Connettività strumenti client
     - Servizi di integrazione
     - Servizi di analisi (opzionale)

2. **Configurazione dell'Istanza**:
   - Nome istanza: HILOGDB (o personalizzato)
   - Account di servizio: Utilizzare un account di servizio dedicato
   - Modalità di autenticazione: Mista
   - Amministratori: Aggiungere gli utenti amministratori

3. **Configurazione del Motore di Database**:
   - Modalità di recupero: Completa
   - Abilitare FILESTREAM per l'accesso Transact-SQL
   - Configurare la directory dei dati su un volume SSD dedicato
   - Configurare la directory dei log su un volume SSD separato

### 2. Configurazione Post-Installazione

1. **Configurazione della Memoria**:
   ```sql
   EXEC sp_configure 'show advanced options', 1;
   RECONFIGURE;
   EXEC sp_configure 'max server memory (MB)', 16384; -- Adattare in base alla RAM disponibile
   EXEC sp_configure 'min server memory (MB)', 4096;  -- Garantire una base minima
   RECONFIGURE;
   ```

2. **Configurazione del Parallelismo**:
   ```sql
   EXEC sp_configure 'max degree of parallelism', 4; -- Ottimizzare in base al numero di core
   EXEC sp_configure 'cost threshold for parallelism', 50; -- Aumentare la soglia per evitare parallelismo eccessivo
   RECONFIGURE;
   ```

3. **Ottimizzazione TempDB**:
   - Creare un file TempDB per ogni core CPU (max 8)
   - Posizionare TempDB su un volume SSD dedicato
   - Impostare la dimensione iniziale uguale per tutti i file

   ```sql
   ALTER DATABASE tempdb MODIFY FILE (NAME = 'tempdev', FILENAME = 'E:\MSSQL\DATA\tempdb.mdf', SIZE = 8GB, FILEGROWTH = 1GB);
   ALTER DATABASE tempdb ADD FILE (NAME = 'tempdev2', FILENAME = 'E:\MSSQL\DATA\tempdb2.ndf', SIZE = 8GB, FILEGROWTH = 1GB);
   ALTER DATABASE tempdb ADD FILE (NAME = 'tempdev3', FILENAME = 'E:\MSSQL\DATA\tempdb3.ndf', SIZE = 8GB, FILEGROWTH = 1GB);
   ALTER DATABASE tempdb ADD FILE (NAME = 'tempdev4', FILENAME = 'E:\MSSQL\DATA\tempdb4.ndf', SIZE = 8GB, FILEGROWTH = 1GB);
   ```

4. **Configurazione dei Backup**:
   - Configurare backup completi settimanali
   - Configurare backup differenziali giornalieri
   - Configurare backup dei log delle transazioni ogni 15 minuti
   - Abilitare la compressione dei backup

## Creazione del Database

### 1. Creazione del Database HILOGDB

1. **Creare il Database**:
   ```sql
   USE master;
   GO

   CREATE DATABASE HILOGDB
   ON PRIMARY 
   (
       NAME = 'HILOGDB_Data',
       FILENAME = 'D:\SQLData\HILOGDB_Data.mdf',
       SIZE = 100GB,
       MAXSIZE = UNLIMITED,
       FILEGROWTH = 10GB
   )
   LOG ON
   (
       NAME = 'HILOGDB_Log',
       FILENAME = 'L:\SQLLogs\HILOGDB_Log.ldf',
       SIZE = 50GB,
       MAXSIZE = UNLIMITED,
       FILEGROWTH = 5GB
   );
   GO
   ```

2. **Configurare i Filegroup**:
   ```sql
   ALTER DATABASE HILOGDB ADD FILEGROUP FG_DATA_CURRENT;
   ALTER DATABASE HILOGDB ADD FILEGROUP FG_DATA_ARCHIVE;
   ALTER DATABASE HILOGDB ADD FILEGROUP FG_INDEXES;
   ALTER DATABASE HILOGDB ADD FILEGROUP FG_BLOBS;

   ALTER DATABASE HILOGDB ADD FILE (
       NAME = 'HILOGDB_Data_Current',
       FILENAME = 'D:\SQLData\HILOGDB_Data_Current.ndf',
       SIZE = 50GB,
       MAXSIZE = UNLIMITED,
       FILEGROWTH = 5GB
   ) TO FILEGROUP FG_DATA_CURRENT;

   ALTER DATABASE HILOGDB ADD FILE (
       NAME = 'HILOGDB_Data_Archive',
       FILENAME = 'D:\SQLData\HILOGDB_Data_Archive.ndf',
       SIZE = 100GB,
       MAXSIZE = UNLIMITED,
       FILEGROWTH = 10GB
   ) TO FILEGROUP FG_DATA_ARCHIVE;

   ALTER DATABASE HILOGDB ADD FILE (
       NAME = 'HILOGDB_Indexes',
       FILENAME = 'D:\SQLData\HILOGDB_Indexes.ndf',
       SIZE = 20GB,
       MAXSIZE = UNLIMITED,
       FILEGROWTH = 2GB
   ) TO FILEGROUP FG_INDEXES;

   ALTER DATABASE HILOGDB ADD FILE (
       NAME = 'HILOGDB_Blobs',
       FILENAME = 'D:\SQLData\HILOGDB_Blobs.ndf',
       SIZE = 50GB,
       MAXSIZE = UNLIMITED,
       FILEGROWTH = 5GB
   ) TO FILEGROUP FG_BLOBS;
   ```

### 2. Creazione dello Schema del Database

1. **Eseguire lo Script di Creazione del Database**:
   - Utilizzare lo script `Schema-Database.sql` fornito nella documentazione
   - Eseguire lo script in SQL Server Management Studio o tramite sqlcmd

2. **Verificare la Creazione delle Tabelle**:
   ```sql
   USE HILOGDB;
   GO
   
   SELECT TABLE_SCHEMA, TABLE_NAME
   FROM INFORMATION_SCHEMA.TABLES
   WHERE TABLE_TYPE = 'BASE TABLE'
   ORDER BY TABLE_SCHEMA, TABLE_NAME;
   ```

3. **Verificare la Creazione degli Indici**:
   ```sql
   USE HILOGDB;
   GO
   
   SELECT i.name AS IndexName, 
          OBJECT_NAME(i.object_id) AS TableName,
          i.type_desc AS IndexType
   FROM sys.indexes i
   WHERE i.object_id > 100 AND i.type > 0
   ORDER BY OBJECT_NAME(i.object_id), i.name;
   ```

### 3. Configurazione del Partizionamento

1. **Creare le Funzioni di Partizionamento**:
   ```sql
   -- Creazione della funzione di partizionamento per data
   CREATE PARTITION FUNCTION PF_Data_Mensile (DATETIME)
   AS RANGE RIGHT FOR VALUES (
       '2025-01-01', '2025-02-01', '2025-03-01', '2025-04-01',
       '2025-05-01', '2025-06-01', '2025-07-01', '2025-08-01',
       '2025-09-01', '2025-10-01', '2025-11-01', '2025-12-01'
   );

   -- Creazione dello schema di partizionamento
   CREATE PARTITION SCHEME PS_Data_Mensile
   AS PARTITION PF_Data_Mensile
   TO (
       FG_DATA_CURRENT, FG_DATA_CURRENT, FG_DATA_CURRENT, FG_DATA_CURRENT,
       FG_DATA_CURRENT, FG_DATA_CURRENT, FG_DATA_CURRENT, FG_DATA_CURRENT,
       FG_DATA_CURRENT, FG_DATA_CURRENT, FG_DATA_CURRENT, FG_DATA_CURRENT,
       FG_DATA_CURRENT
   );
   ```

2. **Applicare il Partizionamento alle Tabelle**:
   ```sql
   -- Applicazione del partizionamento alla tabella Spedizioni
   CREATE CLUSTERED INDEX CIX_Spedizioni_DataSpedizione
   ON dbo.Spedizioni (DataSpedizione)
   WITH (DROP_EXISTING = ON)
   ON PS_Data_Mensile(DataSpedizione);
   ```

## Sviluppo dell'API RESTful

### 1. Creazione del Progetto API

1. **Creare la Soluzione e i Progetti**:
   ```bash
   dotnet new sln -n HiLog
   
   # Progetti principali
   dotnet new classlib -n HiLog.Core -o src/HiLog.Core
   dotnet new classlib -n HiLog.Infrastructure -o src/HiLog.Infrastructure
   dotnet new webapi -n HiLog.API -o src/HiLog.API
   dotnet new worker -n HiLog.Worker -o src/HiLog.Worker
   
   # Progetti di test
   dotnet new xunit -n HiLog.Core.Tests -o tests/HiLog.Core.Tests
   dotnet new xunit -n HiLog.Infrastructure.Tests -o tests/HiLog.Infrastructure.Tests
   dotnet new xunit -n HiLog.API.Tests -o tests/HiLog.API.Tests
   
   # Aggiungere i progetti alla soluzione
   dotnet sln add src/HiLog.Core/HiLog.Core.csproj
   dotnet sln add src/HiLog.Infrastructure/HiLog.Infrastructure.csproj
   dotnet sln add src/HiLog.API/HiLog.API.csproj
   dotnet sln add src/HiLog.Worker/HiLog.Worker.csproj
   dotnet sln add tests/HiLog.Core.Tests/HiLog.Core.Tests.csproj
   dotnet sln add tests/HiLog.Infrastructure.Tests/HiLog.Infrastructure.Tests.csproj
   dotnet sln add tests/HiLog.API.Tests/HiLog.API.Tests.csproj
   ```

2. **Aggiungere i Riferimenti tra Progetti**:
   ```bash
   dotnet add src/HiLog.Infrastructure/HiLog.Infrastructure.csproj reference src/HiLog.Core/HiLog.Core.csproj
   dotnet add src/HiLog.API/HiLog.API.csproj reference src/HiLog.Core/HiLog.Core.csproj
   dotnet add src/HiLog.API/HiLog.API.csproj reference src/HiLog.Infrastructure/HiLog.Infrastructure.csproj
   dotnet add src/HiLog.Worker/HiLog.Worker.csproj reference src/HiLog.Core/HiLog.Core.csproj
   dotnet add src/HiLog.Worker/HiLog.Worker.csproj reference src/HiLog.Infrastructure/HiLog.Infrastructure.csproj
   ```

3. **Installare i Pacchetti NuGet Necessari**:
   ```bash
   # HiLog.Core
   dotnet add src/HiLog.Core/HiLog.Core.csproj package Microsoft.Extensions.DependencyInjection.Abstractions
   dotnet add src/HiLog.Core/HiLog.Core.csproj package Microsoft.Extensions.Logging.Abstractions
   
   # HiLog.Infrastructure
   dotnet add src/HiLog.Infrastructure/HiLog.Infrastructure.csproj package Microsoft.EntityFrameworkCore.SqlServer
   dotnet add src/HiLog.Infrastructure/HiLog.Infrastructure.csproj package Microsoft.Extensions.Caching.StackExchangeRedis
   dotnet add src/HiLog.Infrastructure/HiLog.Infrastructure.csproj package RabbitMQ.Client
   dotnet add src/HiLog.Infrastructure/HiLog.Infrastructure.csproj package Dapper
   
   # HiLog.API
   dotnet add src/HiLog.API/HiLog.API.csproj package Microsoft.AspNetCore.Authentication.JwtBearer
   dotnet add src/HiLog.API/HiLog.API.csproj package Microsoft.EntityFrameworkCore.Design
   dotnet add src/HiLog.API/HiLog.API.csproj package Swashbuckle.AspNetCore
   dotnet add src/HiLog.API/HiLog.API.csproj package AspNetCoreRateLimit
   
   # HiLog.Worker
   dotnet add src/HiLog.Worker/HiLog.Worker.csproj package Microsoft.Extensions.Hosting
   dotnet add src/HiLog.Worker/HiLog.Worker.csproj package Microsoft.EntityFrameworkCore.Design
   ```

### 2. Implementazione del Core Domain

1. **Definire le Entità di Dominio**:
   - Creare le classi per le entità principali (Spedizione, Cliente, Partner, ecc.)
   - Implementare le interfacce dei repository
   - Definire i servizi di dominio

2. **Implementare i DTO (Data Transfer Objects)**:
   - Creare classi DTO per le API
   - Implementare i mapper tra entità e DTO

3. **Definire le Interfacce dei Servizi**:
   - Definire le interfacce per i servizi applicativi
   - Implementare la logica di business nei servizi

### 3. Implementazione dell'Infrastruttura

1. **Configurare Entity Framework Core**:
   - Implementare il DbContext
   - Configurare le mappature delle entità
   - Implementare i repository

2. **Implementare i Servizi di Infrastruttura**:
   - Servizio di caching
   - Servizio di messaggistica
   - Servizio di logging

### 4. Implementazione dell'API

1. **Configurare l'API**:
   - Implementare Startup.cs
   - Configurare l'autenticazione JWT
   - Configurare Swagger
   - Configurare il rate limiting

2. **Implementare i Controller**:
   - Controller per le spedizioni
   - Controller per i clienti
   - Controller per i partner
   - Controller per i documenti
   - Controller per il tracking
   - Controller per i report

3. **Implementare i Middleware**:
   - Middleware per la gestione degli errori
   - Middleware per il logging delle richieste
   - Middleware per la compressione delle risposte

### 5. Configurazione dell'API Gateway

1. **Installare e Configurare NGINX**:
   - Installare NGINX
   - Configurare il reverse proxy
   - Configurare SSL/TLS
   - Configurare il load balancing

2. **Configurare il Routing**:
   - Configurare le rotte per l'API
   - Configurare le rotte per l'interfaccia web
   - Configurare le rotte per Swagger

## Implementazione del Sistema di Caching

### 1. Installazione e Configurazione di Redis

1. **Installare Redis**:
   - Windows: Scaricare e installare Redis per Windows
   - Linux: `sudo apt-get install redis-server`

2. **Configurare Redis**:
   - Modificare il file di configurazione redis.conf
   - Impostare la memoria massima
   - Configurare la persistenza
   - Configurare la sicurezza

3. **Avviare Redis**:
   - Windows: Avviare il servizio Redis
   - Linux: `sudo systemctl start redis-server`

### 2. Implementazione del Servizio di Caching

1. **Creare l'Interfaccia ICacheService**:
   ```csharp
   public interface ICacheService
   {
       Task<T> GetAsync<T>(string key);
       Task SetAsync<T>(string key, T value, TimeSpan? expiry = null);
       Task RemoveAsync(string key);
       Task RemoveByPatternAsync(string pattern);
   }
   ```

2. **Implementare RedisCacheService**:
   ```csharp
   public class RedisCacheService : ICacheService
   {
       private readonly IConnectionMultiplexer _redis;
       private readonly IDatabase _db;
       
       public RedisCacheService(IConnectionMultiplexer redis)
       {
           _redis = redis;
           _db = redis.GetDatabase();
       }
       
       public async Task<T> GetAsync<T>(string key)
       {
           var value = await _db.StringGetAsync(key);
           if (value.IsNull)
           {
               return default;
           }
           
           return JsonSerializer.Deserialize<T>(value);
       }
       
       public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null)
       {
           var serializedValue = JsonSerializer.Serialize(value);
           await _db.StringSetAsync(key, serializedValue, expiry);
       }
       
       public async Task RemoveAsync(string key)
       {
           await _db.KeyDeleteAsync(key);
       }
       
       public async Task RemoveByPatternAsync(string pattern)
       {
           var endpoints = _redis.GetEndPoints();
           var server = _redis.GetServer(endpoints.First());
           var keys = server.Keys(pattern: pattern);
           
           foreach (var key in keys)
           {
               await _db.KeyDeleteAsync(key);
           }
       }
   }
   ```

3. **Registrare il Servizio di Caching**:
   ```csharp
   services.AddSingleton<IConnectionMultiplexer>(sp =>
   {
       var configuration = sp.GetRequiredService<IConfiguration>();
       var connectionString = configuration.GetConnectionString("RedisConnection");
       return ConnectionMultiplexer.Connect(connectionString);
   });
   
   services.AddSingleton<ICacheService, RedisCacheService>();
   ```

### 3. Implementazione del Caching nei Servizi

1. **Implementare il Pattern Cache-Aside**:
   ```csharp
   public async Task<SpedizioneDto> GetSpedizioneAsync(int spedizioneId)
   {
       string cacheKey = $"spedizione:{spedizioneId}";
       
       // Tentativo di recupero dalla cache
       var cachedSpedizione = await _cacheService.GetAsync<SpedizioneDto>(cacheKey);
       if (cachedSpedizione != null)
       {
           return cachedSpedizione;
       }
       
       // Se non in cache, recupera dal database
       var spedizione = await _spedizioneRepository.GetByIdAsync(spedizioneId);
       if (spedizione == null)
       {
           return null;
       }
       
       var spedizioneDto = _mapper.Map<SpedizioneDto>(spedizione);
       
       // Salva in cache con scadenza
       await _cacheService.SetAsync(cacheKey, spedizioneDto, TimeSpan.FromMinutes(30));
       
       return spedizioneDto;
   }
   ```

2. **Implementare l'Invalidazione della Cache**:
   ```csharp
   public async Task UpdateSpedizioneAsync(SpedizioneDto spedizioneDto)
   {
       var spedizione = _mapper.Map<Spedizione>(spedizioneDto);
       await _spedizioneRepository.UpdateAsync(spedizione);
       
       // Invalida la cache
       string cacheKey = $"spedizione:{spedizione.SpedizioneId}";
       await _cacheService.RemoveAsync(cacheKey);
       
       // Invalida anche le liste che potrebbero contenere questa spedizione
       await _cacheService.RemoveByPatternAsync("spedizioni:cliente:*");
       await _cacheService.RemoveByPatternAsync("spedizioni:partner:*");
   }
   ```

## Configurazione dell'Elaborazione Asincrona

### 1. Installazione e Configurazione di RabbitMQ

1. **Installare RabbitMQ**:
   - Windows: Scaricare e installare RabbitMQ
   - Linux: `sudo apt-get install rabbitmq-server`

2. **Configurare RabbitMQ**:
   - Abilitare il plugin di gestione: `rabbitmq-plugins enable rabbitmq_management`
   - Creare un utente amministratore: `rabbitmqctl add_user admin password`
   - Assegnare i permessi: `rabbitmqctl set_user_tags admin administrator`
   - Configurare i permessi: `rabbitmqctl set_permissions -p / admin ".*" ".*" ".*"`

3. **Avviare RabbitMQ**:
   - Windows: Avviare il servizio RabbitMQ
   - Linux: `sudo systemctl start rabbitmq-server`

### 2. Implementazione del Servizio di Messaggistica

1. **Creare l'Interfaccia IMessageService**:
   ```csharp
   public interface IMessageService
   {
       Task PublishAsync<T>(string queue, T message);
       Task SubscribeAsync<T>(string queue, Func<T, Task> handler);
   }
   ```

2. **Implementare RabbitMQService**:
   ```csharp
   public class RabbitMQService : IMessageService, IDisposable
   {
       private readonly IConnection _connection;
       private readonly IModel _channel;
       private readonly ILogger<RabbitMQService> _logger;
       
       public RabbitMQService(IConnectionFactory connectionFactory, ILogger<RabbitMQService> logger)
       {
           _connection = connectionFactory.CreateConnection();
           _channel = _connection.CreateModel();
           _logger = logger;
       }
       
       public async Task PublishAsync<T>(string queue, T message)
       {
           _channel.QueueDeclare(
               queue: queue,
               durable: true,
               exclusive: false,
               autoDelete: false,
               arguments: null);
           
           var serializedMessage = JsonSerializer.Serialize(message);
           var body = Encoding.UTF8.GetBytes(serializedMessage);
           
           var properties = _channel.CreateBasicProperties();
           properties.Persistent = true;
           
           _channel.BasicPublish(
               exchange: "",
               routingKey: queue,
               basicProperties: properties,
               body: body);
           
           await Task.CompletedTask;
       }
       
       public Task SubscribeAsync<T>(string queue, Func<T, Task> handler)
       {
           _channel.QueueDeclare(
               queue: queue,
               durable: true,
               exclusive: false,
               autoDelete: false,
               arguments: null);
           
           _channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);
           
           var consumer = new EventingBasicConsumer(_channel);
           consumer.Received += async (model, ea) =>
           {
               var body = ea.Body.ToArray();
               var serializedMessage = Encoding.UTF8.GetString(body);
               
               try
               {
                   var message = JsonSerializer.Deserialize<T>(serializedMessage);
                   await handler(message);
                   
                   _channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
               }
               catch (Exception ex)
               {
                   _logger.LogError(ex, "Errore durante l'elaborazione del messaggio: {Message}", serializedMessage);
                   _channel.BasicNack(deliveryTag: ea.DeliveryTag, multiple: false, requeue: true);
               }
           };
           
           _channel.BasicConsume(
               queue: queue,
               autoAck: false,
               consumer: consumer);
           
           return Task.CompletedTask;
       }
       
       public void Dispose()
       {
           _channel?.Dispose();
           _connection?.Dispose();
       }
   }
   ```

3. **Registrare il Servizio di Messaggistica**:
   ```csharp
   services.AddSingleton<IConnectionFactory>(sp =>
   {
       var configuration = sp.GetRequiredService<IConfiguration>();
       return new ConnectionFactory
       {
           HostName = configuration["RabbitMQ:HostName"],
           UserName = configuration["RabbitMQ:UserName"],
           Password = configuration["RabbitMQ:Password"],
           VirtualHost = configuration["RabbitMQ:VirtualHost"]
       };
   });
   
   services.AddSingleton<IMessageService, RabbitMQService>();
   ```

###
