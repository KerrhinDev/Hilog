# Guida all'Implementazione del Sistema Hi-log Ottimizzato - Parte 2

## Sviluppo dell'Interfaccia Web

### 1. Creazione del Progetto Frontend

1. **Configurazione dell'Ambiente React**:
   ```bash
   # Creare un nuovo progetto React con TypeScript
   npx create-react-app hilog-web --template typescript
   cd hilog-web
   
   # Installare le dipendenze necessarie
   npm install axios react-router-dom @mui/material @mui/icons-material @emotion/react @emotion/styled recharts formik yup date-fns i18next react-i18next
   ```

2. **Struttura del Progetto Frontend**:
   ```
   hilog-web/
   ├── public/
   │   ├── assets/
   │   │   ├── images/
   │   │   └── icons/
   │   ├── locales/
   │   │   ├── it/
   │   │   └── en/
   │   ├── favicon.ico
   │   └── index.html
   ├── src/
   │   ├── api/
   │   │   ├── apiClient.ts
   │   │   ├── spedizioniApi.ts
   │   │   ├── clientiApi.ts
   │   │   ├── partnerApi.ts
   │   │   └── ...
   │   ├── components/
   │   │   ├── common/
   │   │   ├── layout/
   │   │   ├── spedizioni/
   │   │   ├── clienti/
   │   │   └── ...
   │   ├── contexts/
   │   │   ├── AuthContext.tsx
   │   │   └── ...
   │   ├── hooks/
   │   │   ├── useAuth.ts
   │   │   ├── useApi.ts
   │   │   └── ...
   │   ├── pages/
   │   │   ├── Dashboard.tsx
   │   │   ├── Spedizioni.tsx
   │   │   ├── Clienti.tsx
   │   │   └── ...
   │   ├── types/
   │   │   ├── spedizione.ts
   │   │   ├── cliente.ts
   │   │   └── ...
   │   ├── utils/
   │   │   ├── formatters.ts
   │   │   ├── validators.ts
   │   │   └── ...
   │   ├── App.tsx
   │   ├── index.tsx
   │   └── routes.tsx
   ├── package.json
   └── tsconfig.json
   ```

### 2. Implementazione dell'Autenticazione

1. **Creare il Contesto di Autenticazione**:
   ```typescript
   // src/contexts/AuthContext.tsx
   import React, { createContext, useState, useEffect, ReactNode } from 'react';
   import axios from 'axios';
   import { apiClient } from '../api/apiClient';
   
   interface AuthContextType {
     isAuthenticated: boolean;
     user: any | null;
     login: (username: string, password: string) => Promise<void>;
     logout: () => void;
     loading: boolean;
   }
   
   export const AuthContext = createContext<AuthContextType>({
     isAuthenticated: false,
     user: null,
     login: async () => {},
     logout: () => {},
     loading: true,
   });
   
   interface AuthProviderProps {
     children: ReactNode;
   }
   
   export const AuthProvider: React.FC<AuthProviderProps> = ({ children }) => {
     const [isAuthenticated, setIsAuthenticated] = useState<boolean>(false);
     const [user, setUser] = useState<any | null>(null);
     const [loading, setLoading] = useState<boolean>(true);
   
     useEffect(() => {
       const checkAuth = async () => {
         const token = localStorage.getItem('token');
         if (token) {
           try {
             apiClient.defaults.headers.common['Authorization'] = `Bearer ${token}`;
             const response = await apiClient.get('/api/auth/me');
             setUser(response.data);
             setIsAuthenticated(true);
           } catch (error) {
             localStorage.removeItem('token');
             delete apiClient.defaults.headers.common['Authorization'];
           }
         }
         setLoading(false);
       };
   
       checkAuth();
     }, []);
   
     const login = async (username: string, password: string) => {
       try {
         const response = await apiClient.post('/api/auth/token', {
           username,
           password,
         });
   
         const { access_token, user: userData } = response.data;
         localStorage.setItem('token', access_token);
         apiClient.defaults.headers.common['Authorization'] = `Bearer ${access_token}`;
         setUser(userData);
         setIsAuthenticated(true);
       } catch (error) {
         throw error;
       }
     };
   
     const logout = () => {
       localStorage.removeItem('token');
       delete apiClient.defaults.headers.common['Authorization'];
       setUser(null);
       setIsAuthenticated(false);
     };
   
     return (
       <AuthContext.Provider value={{ isAuthenticated, user, login, logout, loading }}>
         {children}
       </AuthContext.Provider>
     );
   };
   ```

2. **Implementare il Client API**:
   ```typescript
   // src/api/apiClient.ts
   import axios from 'axios';
   
   export const apiClient = axios.create({
     baseURL: process.env.REACT_APP_API_URL || 'http://localhost:5000',
     headers: {
       'Content-Type': 'application/json',
     },
   });
   
   apiClient.interceptors.response.use(
     (response) => response,
     async (error) => {
       const originalRequest = error.config;
   
       // Se il token è scaduto e non abbiamo già provato a rinnovarlo
       if (error.response.status === 401 && !originalRequest._retry) {
         originalRequest._retry = true;
   
         try {
           const refreshToken = localStorage.getItem('refreshToken');
           const response = await axios.post('/api/auth/refresh', {
             refresh_token: refreshToken,
           });
   
           const { access_token } = response.data;
           localStorage.setItem('token', access_token);
           apiClient.defaults.headers.common['Authorization'] = `Bearer ${access_token}`;
   
           return apiClient(originalRequest);
         } catch (refreshError) {
           // Se il refresh token è scaduto, logout
           localStorage.removeItem('token');
           localStorage.removeItem('refreshToken');
           window.location.href = '/login';
           return Promise.reject(refreshError);
         }
       }
   
       return Promise.reject(error);
     }
   );
   ```

### 3. Implementazione delle Pagine Principali

1. **Dashboard**:
   ```typescript
   // src/pages/Dashboard.tsx
   import React, { useEffect, useState } from 'react';
   import { Grid, Paper, Typography, Box } from '@mui/material';
   import { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip, Legend, ResponsiveContainer } from 'recharts';
   import { apiClient } from '../api/apiClient';
   
   const Dashboard: React.FC = () => {
     const [stats, setStats] = useState<any>({
       spedizioniTotali: 0,
       spedizioniInCorso: 0,
       spedizioniConsegnate: 0,
       clientiAttivi: 0,
     });
     const [chartData, setChartData] = useState<any[]>([]);
     const [loading, setLoading] = useState<boolean>(true);
   
     useEffect(() => {
       const fetchData = async () => {
         try {
           const [statsResponse, chartResponse] = await Promise.all([
             apiClient.get('/api/dashboard/stats'),
             apiClient.get('/api/dashboard/chart-data'),
           ]);
   
           setStats(statsResponse.data);
           setChartData(chartResponse.data);
         } catch (error) {
           console.error('Errore durante il recupero dei dati della dashboard:', error);
         } finally {
           setLoading(false);
         }
       };
   
       fetchData();
     }, []);
   
     return (
       <Box p={3}>
         <Typography variant="h4" gutterBottom>
           Dashboard
         </Typography>
   
         <Grid container spacing={3}>
           {/* Statistiche */}
           <Grid item xs={12} sm={6} md={3}>
             <Paper sx={{ p: 2, display: 'flex', flexDirection: 'column', height: 140 }}>
               <Typography variant="h6" color="textSecondary">
                 Spedizioni Totali
               </Typography>
               <Typography component="p" variant="h4">
                 {stats.spedizioniTotali}
               </Typography>
             </Paper>
           </Grid>
           <Grid item xs={12} sm={6} md={3}>
             <Paper sx={{ p: 2, display: 'flex', flexDirection: 'column', height: 140 }}>
               <Typography variant="h6" color="textSecondary">
                 Spedizioni in Corso
               </Typography>
               <Typography component="p" variant="h4">
                 {stats.spedizioniInCorso}
               </Typography>
             </Paper>
           </Grid>
           <Grid item xs={12} sm={6} md={3}>
             <Paper sx={{ p: 2, display: 'flex', flexDirection: 'column', height: 140 }}>
               <Typography variant="h6" color="textSecondary">
                 Spedizioni Consegnate
               </Typography>
               <Typography component="p" variant="h4">
                 {stats.spedizioniConsegnate}
               </Typography>
             </Paper>
           </Grid>
           <Grid item xs={12} sm={6} md={3}>
             <Paper sx={{ p: 2, display: 'flex', flexDirection: 'column', height: 140 }}>
               <Typography variant="h6" color="textSecondary">
                 Clienti Attivi
               </Typography>
               <Typography component="p" variant="h4">
                 {stats.clientiAttivi}
               </Typography>
             </Paper>
           </Grid>
   
           {/* Grafico */}
           <Grid item xs={12}>
             <Paper sx={{ p: 2 }}>
               <Typography variant="h6" gutterBottom>
                 Spedizioni per Mese
               </Typography>
               <ResponsiveContainer width="100%" height={300}>
                 <BarChart
                   data={chartData}
                   margin={{
                     top: 5,
                     right: 30,
                     left: 20,
                     bottom: 5,
                   }}
                 >
                   <CartesianGrid strokeDasharray="3 3" />
                   <XAxis dataKey="mese" />
                   <YAxis />
                   <Tooltip />
                   <Legend />
                   <Bar dataKey="spedizioni" fill="#8884d8" name="Spedizioni" />
                   <Bar dataKey="consegne" fill="#82ca9d" name="Consegne" />
                 </BarChart>
               </ResponsiveContainer>
             </Paper>
           </Grid>
         </Grid>
       </Box>
     );
   };
   
   export default Dashboard;
   ```

2. **Pagina Spedizioni**:
   ```typescript
   // src/pages/Spedizioni.tsx
   import React, { useState, useEffect } from 'react';
   import {
     Box,
     Paper,
     Typography,
     Table,
     TableBody,
     TableCell,
     TableContainer,
     TableHead,
     TableRow,
     TablePagination,
     Button,
     TextField,
     Grid,
     IconButton,
     Chip,
   } from '@mui/material';
   import { Add as AddIcon, Search as SearchIcon, Visibility as VisibilityIcon } from '@mui/icons-material';
   import { useNavigate } from 'react-router-dom';
   import { format } from 'date-fns';
   import { it } from 'date-fns/locale';
   import { apiClient } from '../api/apiClient';
   
   const Spedizioni: React.FC = () => {
     const navigate = useNavigate();
     const [spedizioni, setSpedizioni] = useState<any[]>([]);
     const [loading, setLoading] = useState<boolean>(true);
     const [page, setPage] = useState<number>(0);
     const [rowsPerPage, setRowsPerPage] = useState<number>(10);
     const [totalItems, setTotalItems] = useState<number>(0);
     const [searchTerm, setSearchTerm] = useState<string>('');
   
     useEffect(() => {
       fetchSpedizioni();
     }, [page, rowsPerPage, searchTerm]);
   
     const fetchSpedizioni = async () => {
       try {
         setLoading(true);
         const response = await apiClient.get('/api/spedizioni', {
           params: {
             page: page + 1,
             per_page: rowsPerPage,
             search: searchTerm || undefined,
           },
         });
   
         setSpedizioni(response.data.data);
         setTotalItems(response.data.meta.total_items);
       } catch (error) {
         console.error('Errore durante il recupero delle spedizioni:', error);
       } finally {
         setLoading(false);
       }
     };
   
     const handleChangePage = (event: unknown, newPage: number) => {
       setPage(newPage);
     };
   
     const handleChangeRowsPerPage = (event: React.ChangeEvent<HTMLInputElement>) => {
       setRowsPerPage(parseInt(event.target.value, 10));
       setPage(0);
     };
   
     const handleSearch = (event: React.ChangeEvent<HTMLInputElement>) => {
       setSearchTerm(event.target.value);
       setPage(0);
     };
   
     const getStatusColor = (status: string) => {
       switch (status) {
         case 'CREATA':
           return 'default';
         case 'IN_TRANSITO':
           return 'primary';
         case 'IN_CONSEGNA':
           return 'info';
         case 'CONSEGNATA':
           return 'success';
         case 'PROBLEMI':
           return 'error';
         default:
           return 'default';
       }
     };
   
     return (
       <Box p={3}>
         <Box display="flex" justifyContent="space-between" alignItems="center" mb={3}>
           <Typography variant="h4">Spedizioni</Typography>
           <Button
             variant="contained"
             color="primary"
             startIcon={<AddIcon />}
             onClick={() => navigate('/spedizioni/nuova')}
           >
             Nuova Spedizione
           </Button>
         </Box>
   
         <Paper sx={{ mb: 3, p: 2 }}>
           <Grid container spacing={2} alignItems="center">
             <Grid item xs={12} sm={6} md={4}>
               <TextField
                 fullWidth
                 label="Cerca"
                 variant="outlined"
                 value={searchTerm}
                 onChange={handleSearch}
                 InputProps={{
                   endAdornment: <SearchIcon />,
                 }}
               />
             </Grid>
           </Grid>
         </Paper>
   
         <TableContainer component={Paper}>
           <Table>
             <TableHead>
               <TableRow>
                 <TableCell>Numero Spedizione</TableCell>
                 <TableCell>Cliente</TableCell>
                 <TableCell>Data Spedizione</TableCell>
                 <TableCell>Stato</TableCell>
                 <TableCell>Destinatario</TableCell>
                 <TableCell>Partner</TableCell>
                 <TableCell>Azioni</TableCell>
               </TableRow>
             </TableHead>
             <TableBody>
               {loading ? (
                 <TableRow>
                   <TableCell colSpan={7} align="center">
                     Caricamento...
                   </TableCell>
                 </TableRow>
               ) : spedizioni.length === 0 ? (
                 <TableRow>
                   <TableCell colSpan={7} align="center">
                     Nessuna spedizione trovata
                   </TableCell>
                 </TableRow>
               ) : (
                 spedizioni.map((spedizione) => (
                   <TableRow key={spedizione.spedizione_id}>
                     <TableCell>{spedizione.numero_spedizione}</TableCell>
                     <TableCell>{spedizione.cliente?.ragione_sociale}</TableCell>
                     <TableCell>
                       {format(new Date(spedizione.data_spedizione), 'dd/MM/yyyy', { locale: it })}
                     </TableCell>
                     <TableCell>
                       <Chip
                         label={spedizione.stato.descrizione}
                         color={getStatusColor(spedizione.stato.codice)}
                       />
                     </TableCell>
                     <TableCell>{spedizione.destinatario.nome}</TableCell>
                     <TableCell>{spedizione.partner?.nome || '-'}</TableCell>
                     <TableCell>
                       <IconButton
                         color="primary"
                         onClick={() => navigate(`/spedizioni/${spedizione.spedizione_id}`)}
                       >
                         <VisibilityIcon />
                       </IconButton>
                     </TableCell>
                   </TableRow>
                 ))
               )}
             </TableBody>
           </Table>
           <TablePagination
             rowsPerPageOptions={[5, 10, 25, 50]}
             component="div"
             count={totalItems}
             rowsPerPage={rowsPerPage}
             page={page}
             onPageChange={handleChangePage}
             onRowsPerPageChange={handleChangeRowsPerPage}
             labelRowsPerPage="Righe per pagina:"
             labelDisplayedRows={({ from, to, count }) => `${from}-${to} di ${count}`}
           />
         </TableContainer>
       </Box>
     );
   };
   
   export default Spedizioni;
   ```

## Migrazione dei Dati

### 1. Analisi dei Dati Esistenti

1. **Esaminare la Struttura dei Database Esistenti**:
   - Analizzare le tabelle del database Microsoft Access
   - Analizzare le tabelle del database SQL Server esistente
   - Identificare le relazioni tra le tabelle

2. **Mappare i Dati al Nuovo Schema**:
   - Creare un documento di mappatura che mostra come i dati esistenti si mapperanno al nuovo schema
   - Identificare eventuali trasformazioni necessarie
   - Identificare eventuali dati mancanti o inconsistenti

### 2. Sviluppo degli Script di Migrazione

1. **Creare Script di Estrazione**:
   ```sql
   -- Esempio di script per estrarre dati da SQL Server esistente
   SELECT 
       c.CodiceCliente,
       c.RagioneSociale,
       c.Indirizzo,
       c.CAP,
       c.Citta,
       c.Provincia,
       c.Nazione,
       c.PartitaIVA,
       c.CodiceFiscale,
       c.Telefono,
       c.Email,
       c.DataCreazione,
       c.DataModifica,
       c.Attivo
   FROM ClientiOld c
   ORDER BY c.CodiceCliente;
   ```

2. **Creare Script di Trasformazione**:
   ```csharp
   // Esempio di codice C# per trasformare i dati
   public class DataMigrationService
   {
       private readonly ILogger<DataMigrationService> _logger;
       private readonly string _connectionStringOld;
       private readonly string _connectionStringNew;
       
       public DataMigrationService(
           ILogger<DataMigrationService> logger,
           IConfiguration configuration)
       {
           _logger = logger;
           _connectionStringOld = configuration.GetConnectionString("OldDatabase");
           _connectionStringNew = configuration.GetConnectionString("NewDatabase");
       }
       
       public async Task MigrateClientiAsync()
       {
           _logger.LogInformation("Inizio migrazione clienti");
           
           try
           {
               using var connectionOld = new SqlConnection(_connectionStringOld);
               using var connectionNew = new SqlConnection(_connectionStringNew);
               
               await connectionOld.OpenAsync();
               await connectionNew.OpenAsync();
               
               // Lettura dei clienti dal vecchio database
               var clienti = await connectionOld.QueryAsync<ClienteOld>("SELECT * FROM ClientiOld");
               
               foreach (var cliente in clienti)
               {
                   // Trasformazione
                   var nuovoCliente = new
                   {
                       CodiceCliente = cliente.CodiceCliente,
                       RagioneSociale = cliente.RagioneSociale,
                       Indirizzo = cliente.Indirizzo,
                       CAP = cliente.CAP,
                       Citta = cliente.Citta,
                       Provincia = cliente.Provincia,
                       Nazione = cliente.Nazione ?? "Italia",
                       PartitaIVA = cliente.PartitaIVA,
                       CodiceFiscale = cliente.CodiceFiscale,
                       Telefono = cliente.Telefono,
                       Email = cliente.Email,
                       DataCreazione = cliente.DataCreazione ?? DateTime.UtcNow,
                       DataModifica = cliente.DataModifica,
                       Attivo = cliente.Attivo ?? true
                   };
                   
                   // Inserimento nel nuovo database
                   await connectionNew.ExecuteAsync(@"
                       INSERT INTO dbo.Clienti (
                           CodiceCliente, RagioneSociale, Indirizzo, CAP, Citta, Provincia, Nazione,
                           PartitaIVA, CodiceFiscale, Telefono, Email, DataCreazione, DataModifica, Attivo
                       ) VALUES (
                           @CodiceCliente, @RagioneSociale, @Indirizzo, @CAP, @Citta, @Provincia, @Nazione,
                           @PartitaIVA, @CodiceFiscale, @Telefono, @Email, @DataCreazione, @DataModifica, @Attivo
                       )", nuovoCliente);
               }
               
               _logger.LogInformation("Migrazione clienti completata con successo");
           }
           catch (Exception ex)
           {
               _logger.LogError(ex, "Errore durante la migrazione dei clienti");
               throw;
           }
       }
   }
   ```

3. **Creare Script di Caricamento**:
   ```sql
   -- Esempio di script per caricare dati nel nuovo database
   INSERT INTO dbo.Clienti (
       CodiceCliente, RagioneSociale, Indirizzo, CAP, Citta, Provincia, Nazione,
       PartitaIVA, CodiceFiscale, Telefono, Email, DataCreazione, DataModifica, Attivo
   )
   SELECT 
       c.CodiceCliente,
       c.RagioneSociale,
       c.Indirizzo,
       c.CAP,
       c.Citta,
       c.Provincia,
       ISNULL(c.Nazione, 'Italia'),
       c.PartitaIVA,
       c.CodiceFiscale,
       c.Telefono,
       c.Email,
       ISNULL(c.DataCreazione, GETDATE()),
       c.DataModifica,
       ISNULL(c.Attivo, 1)
   FROM #TempClienti c;
   ```

### 3. Esecuzione della Migrazione

1. **Preparare l'Ambiente di Migrazione**:
   - Creare un ambiente di test per la migrazione
   - Eseguire backup dei database esistenti
   - Configurare i server di origine e destinazione

2. **Eseguire la Migrazione di Test**:
   - Eseguire gli script di migrazione in ambiente di test
   - Verificare i dati migrati
   - Correggere eventuali problemi

3. **Eseguire la Migrazione in Produzione**:
   - Pianificare un periodo di inattività
   - Eseguire backup dei database di produzione
   - Eseguire gli script di migrazione
   - Verificare i dati migrati

4. **Verificare l'Integrità dei Dati**:
   - Eseguire controlli di integrità dei dati
   - Verificare le relazioni tra le tabelle
   - Verificare i conteggi dei record

## Test e Verifica

### 1. Test Unitari

1. **Configurare il Framework di Test**:
   ```bash
   # Installare i pacchetti necessari
   dotnet add tests/HiLog.Core.Tests/HiLog.Core.Tests.csproj package Moq
   dotnet add tests/HiLog.Core.Tests/HiLog.Core.Tests.csproj package FluentAssertions
   ```

2. **Implementare i Test Unitari**:
   ```csharp
   // Esempio di test unitario per un servizio
   public class SpedizioneServiceTests
   {
       private readonly Mock<ISpedizioneRepository> _mockRepository;
       private readonly Mock<ICacheService> _mockCacheService;
       private readonly Mock<IMapper> _mockMapper;
       private readonly SpedizioneService _service;
       
       public SpedizioneServiceTests()
       {
           _mockRepository = new Mock<ISpedizioneRepository>();
           _mockCacheService = new Mock<ICacheService>();
           _mockMapper = new Mock<IMapper>();
           
           _service = new SpedizioneService(
               _mockRepository.Object,
               _mockCacheService.Object,
               _mockMapper.Object);
       }
       
       [Fact]
       public async Task GetSpedizioneAsync_WithValidId_ReturnsSpedizione()
       {
           // Arrange
           var spedizioneId = 1;
           var spedizione = new Spedizione { SpedizioneID = spedizioneId, NumeroSpedizione = "SP12345" };
           var spedizioneDto = new SpedizioneDto { SpedizioneID = spedizioneId, NumeroSpedizione = "SP12345" };
           
           _mockRepository.Setup(r => r.GetByIdAsync(spedizioneId))
               .ReturnsAsync(spedizione);
               
           _mockCacheService.Setup(c => c.GetAsync<SpedizioneDto>($"spedizione:{spedizioneId}"))
               .ReturnsAsync((SpedizioneDto)null);
               
           _mockMapper.Setup(m => m.Map<SpedizioneDto>(spedizione))
               .Returns(spedizioneDto);
           
           // Act
           var result = await _service.GetSpedizioneAsync(spedizioneId);
           
           // Assert
           result.Should().NotBeNull();
           result.SpedizioneID.Should().Be(spedizioneId);
           result.NumeroSpedizione.Should().Be("SP12345");
           
           _mockCacheService.Verify(c => c.SetAsync(
               $"spedizione:{spedizioneId}",
               It.IsAny<SpedizioneDto>(),
               It.IsAny<TimeSpan?>()),
               Times.Once);
       }
       
       [Fact]
       public async Task GetSpedizioneAsync_WithInvalidId_ReturnsNull()
       {
           // Arrange
           var spedizioneId = 999;
           
           _mockRepository.Setup(r => r.GetByIdAsync(spedizioneId))
               .ReturnsAsync((Spedizione)null);
               
           _mockCacheService.Setup(c => c.GetAsync<SpedizioneDto>($"spedizione:{spedizioneId}"))
               .ReturnsAsync((SpedizioneDto)null);
           
           // Act
           var result = await _service.GetSpedizioneAsync(spedizioneId);
           
           // Assert
           result.Should().BeNull();
           
           _mockCacheService.Verify(c => c.SetAsync(
               It.IsAny<string>(),
               It.IsAny<SpedizioneDto>(),
               It.IsAny<TimeSpan?>()),
               Times.Never);
       }
   }
   ```

### 2. Test di Integrazione

1. **Configurare il Database di Test**:
   ```csharp
   // Esempio di configurazione per i test di integrazione
   public class TestDatabaseFixture : IDisposable
   {
       private const string ConnectionString = "Server=(localdb)\\mssqllocaldb;Database=HiLogTestDb;Trusted_Connection=True;";
       
       public TestDatabaseFixture()
       {
           var options = new DbContextOptionsBuilder<HiLogDbContext>()
               .UseSqlServer(ConnectionString)
               .Options;
           
           using (var context = new HiLogDbContext(options))
           {
               context.Database.EnsureDeleted();
               context.Database.EnsureCreated();
               
               // Seed del database di test
               SeedDatabase(context);
           }
       }
       
       public HiLogDbContext CreateContext()
       {
           var options = new DbContextOptionsBuilder<HiLogDbContext>()
               .UseSqlServer(ConnectionString)
               .Options;
               
           return new HiLogDbContext(options);
       }
       
       private void SeedDatabase(HiLogDbContext context)
       {
           // Aggiungere dati di test
           context.StatoSpedizioni.AddRange(
               new StatoSpedizione { Codice = "CREATA", Descrizione = "Spedizione Creata" },
               new StatoSpedizione { Codice = "IN_TRANSITO", Descrizione = "In Transito" },
               new StatoSpedizione { Codice = "CONSEGNATA", Descrizione = "Consegnata" }
           );
           
           context.Clienti.Add(new Cliente
           {
               CodiceCliente = "TEST001",
               RagioneSociale = "Cliente Test",
               Indirizzo = "Via Test 123",
               CAP = "00100",
               Citta = "Roma",
               Provincia = "RM",
