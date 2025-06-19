# Conectividade e CRUD - ColdTech

## 📋 Visão Geral

O sistema ColdTech implementa uma arquitetura híbrida de dados, utilizando **Supabase** como banco principal e **LocalStorage** como fallback, garantindo disponibilidade e performance através de operações CRUD completas.

## 🏗️ Arquitetura de Dados

```mermaid
graph TB
    A[Frontend React] --> B[DatabaseService]
    B --> C{Conexão Disponível?}
    C -->|Sim| D[Supabase PostgreSQL]
    C -->|Não| E[LocalStorage Fallback]
    
    D --> F[Tabelas Relacionais]
    F --> G[agendamentos]
    F --> H[clientes]
    F --> I[servicos]
    F --> J[usuarios]
    
    E --> K[JSON Local]
    K --> L[database.json]
    K --> M[localStorage]
    
    B --> N[CRUD Operations]
    N --> O[Create]
    N --> P[Read]
    N --> Q[Update]
    N --> R[Delete]
    
    style D fill:#4ade80
    style E fill:#fbbf24
    style B fill:#3b82f6
```

## 🗄️ Estrutura do Banco de Dados

### Tabelas Principais

#### 1. **agendamentos**
```sql
CREATE TABLE agendamentos (
  id SERIAL PRIMARY KEY,
  cliente_id INTEGER REFERENCES clientes(id),
  servico_id INTEGER REFERENCES servicos(id),
  data DATE NOT NULL,
  time TIME NOT NULL,
  local TEXT NOT NULL,
  preco_personalizado DECIMAL(10,2),
  status VARCHAR(20) DEFAULT 'pendente',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### 2. **clientes**
```sql
CREATE TABLE clientes (
  id SERIAL PRIMARY KEY,
  nome VARCHAR(255) NOT NULL,
  contato VARCHAR(50) NOT NULL,
  endereco TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

#### 3. **servicos**
```sql
CREATE TABLE servicos (
  id SERIAL PRIMARY KEY,
  tipo VARCHAR(255) NOT NULL,
  descricao TEXT,
  preco_base DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT NOW()
);
```

#### 4. **usuarios**
```sql
CREATE TABLE usuarios (
  id SERIAL PRIMARY KEY,
  nome VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  senha VARCHAR(255) NOT NULL,
  ultimo_login TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW()
);
```

## 🔌 Configuração de Conectividade

### Supabase Client
```javascript
// supabaseClient.js
import { createClient } from '@supabase/supabase-js';

const supabaseUrl = import.meta.env.VITE_SUPABASE_URL;
const supabaseAnonKey = import.meta.env.VITE_SUPABASE_ANON_KEY;

const supabase = createClient(supabaseUrl, supabaseAnonKey);
export default supabase;
```

### Variáveis de Ambiente
```env
VITE_SUPABASE_URL=https://seu-projeto.supabase.co
VITE_SUPABASE_ANON_KEY=sua-chave-anonima
```

## 🛠️ DatabaseService - Camada de Abstração

### Estrutura da Classe

```mermaid
classDiagram
    class DatabaseService {
        -database: Object
        +initializeData()
        +initializeSupabaseData()
        +useFallbackData()
        +saveToLocalStorage()
        
        // Agendamentos
        +getAgendamentos()
        +getAgendamentosByDate(date)
        +getAgendamentosPendentes()
        +getAgendamentosConcluidos()
        +addAgendamento(agendamento)
        +updateAgendamento(index, agendamento)
        +deleteAgendamento(index, id)
        
        // Clientes
        +getClientes()
        +addCliente(cliente)
        +updateCliente(id, cliente)
        +deleteCliente(id)
        
        // Serviços
        +getServicos()
        
        // Estatísticas
        +getEstatisticas()
    }
```

## 📊 Operações CRUD Detalhadas

### 1. **CREATE (Criar)**

#### Agendamentos
```javascript
async addAgendamento(agendamento) {
  try {
    // 1. Buscar ou criar cliente
    let cliente_id;
    const { data: clienteExistente } = await supabase
      .from('clientes')
      .select('id')
      .eq('nome', agendamento.cliente)
      .single();
    
    if (clienteExistente) {
      cliente_id = clienteExistente.id;
    } else {
      // Criar novo cliente
      const { data: novoCliente } = await supabase
        .from('clientes')
        .insert({
          nome: agendamento.cliente,
          contato: agendamento.contato,
          endereco: agendamento.local
        })
        .select('id')
        .single();
      
      cliente_id = novoCliente.id;
    }
    
    // 2. Buscar serviço
    const { data: servico } = await supabase
      .from('servicos')
      .select('id')
      .eq('tipo', agendamento.servico)
      .single();
    
    // 3. Inserir agendamento
    const { data, error } = await supabase
      .from('agendamentos')
      .insert({
        cliente_id,
        servico_id: servico.id,
        data: agendamento.data,
        time: agendamento.time,
        local: agendamento.local,
        preco_personalizado: agendamento.preco,
        status: agendamento.status
      })
      .select(`
        *,
        clientes (nome, contato),
        servicos (tipo)
      `)
      .single();
    
    return transformedData;
  } catch (error) {
    // Fallback para localStorage
    return this.fallbackCreate(agendamento);
  }
}
```

### 2. **READ (Ler)**

#### Fluxo de Leitura
```mermaid
sequenceDiagram
    participant F as Frontend
    participant D as DatabaseService
    participant S as Supabase
    participant L as LocalStorage
    
    F->>D: getAgendamentos()
    D->>S: SELECT com JOINs
    S-->>D: Dados relacionais
    D->>D: Transform data
    D-->>F: Dados formatados
    
    Note over D,S: Se falhar
    D->>L: Buscar fallback
    L-->>D: JSON local
    D-->>F: Dados locais
```

#### Consultas com Relacionamentos
```javascript
async getAgendamentos() {
  try {
    const { data, error } = await supabase
      .from('agendamentos')
      .select(`
        *,
        clientes (nome, contato),
        servicos (tipo)
      `)
      .order('data', { ascending: true });
    
    // Transformar para formato da aplicação
    return data.map(item => ({
      id: item.id,
      cliente: item.clientes?.nome || '',
      contato: item.clientes?.contato || '',
      servico: item.servicos?.tipo || '',
      data: item.data,
      time: item.time,
      local: item.local,
      preco: item.preco_personalizado,
      status: item.status
    }));
  } catch (error) {
    return this.database?.agendamentos || [];
  }
}
```

### 3. **UPDATE (Atualizar)**

#### Processo de Atualização
```javascript
async updateAgendamento(index, agendamento) {
  try {
    // Validar ID
    if (!agendamento.id) {
      throw new Error('ID do agendamento não fornecido');
    }
    
    // Buscar serviço atualizado
    const { data: servico } = await supabase
      .from('servicos')
      .select('id')
      .eq('tipo', agendamento.servico)
      .single();
    
    // Atualizar no banco
    const { data, error } = await supabase
      .from('agendamentos')
      .update({
        data: agendamento.data,
        time: agendamento.time,
        servico_id: servico.id,
        preco_personalizado: agendamento.preco,
        status: agendamento.status
      })
      .eq('id', agendamento.id)
      .select()
      .single();
    
    return transformedData;
  } catch (error) {
    // Fallback local
    return this.fallbackUpdate(index, agendamento);
  }
}
```

### 4. **DELETE (Excluir)**

#### Exclusão Segura
```javascript
async deleteAgendamento(index, id) {
  try {
    if (!id) {
      throw new Error('ID do agendamento não fornecido');
    }
    
    const { error } = await supabase
      .from('agendamentos')
      .delete()
      .eq('id', id);
    
    if (error) throw error;
    return true;
  } catch (error) {
    // Fallback para exclusão local
    return this.fallbackDelete(index);
  }
}
```

## 🔄 Sistema de Fallback

### Estratégia Híbrida
```mermaid
graph TD
    A[Operação CRUD] --> B{Supabase Online?}
    B -->|Sim| C[Executar no Supabase]
    B -->|Não| D[Executar no LocalStorage]
    
    C --> E{Sucesso?}
    E -->|Sim| F[Retornar Dados]
    E -->|Não| G[Fallback para Local]
    
    D --> H[Atualizar JSON Local]
    G --> H
    H --> I[Sincronizar quando Online]
    
    style C fill:#4ade80
    style D fill:#fbbf24
    style I fill:#8b5cf6
```

### Implementação do Fallback
```javascript
useFallbackData() {
  const savedData = localStorage.getItem('coldtech_database');
  if (savedData) {
    this.database = JSON.parse(savedData);
  } else {
    this.database = databaseData; // JSON inicial
    this.saveToLocalStorage();
  }
}

saveToLocalStorage() {
  localStorage.setItem('coldtech_database', JSON.stringify(this.database));
}
```

## 📈 Estatísticas e Relatórios

### Consultas Agregadas
```javascript
async getEstatisticas() {
  try {
    const hoje = new Date().toISOString().split('T')[0];
    
    // Múltiplas consultas paralelas
    const [
      { count: totalAgendamentos },
      { count: pendentes },
      { count: concluidos },
      { count: clientesUnicos },
      { count: agendamentosHoje }
    ] = await Promise.all([
      supabase.from('agendamentos').select('*', { count: 'exact', head: true }),
      supabase.from('agendamentos').select('*', { count: 'exact', head: true }).eq('status', 'pendente'),
      supabase.from('agendamentos').select('*', { count: 'exact', head: true }).eq('status', 'concluido'),
      supabase.from('clientes').select('*', { count: 'exact', head: true }),
      supabase.from('agendamentos').select('*', { count: 'exact', head: true }).eq('data', hoje)
    ]);
    
    return {
      totalAgendamentos: totalAgendamentos || 0,
      pendentes: pendentes || 0,
      concluidos: concluidos || 0,
      clientesUnicos: clientesUnicos || 0,
      agendamentosHoje: agendamentosHoje || 0
    };
  } catch (error) {
    return this.calculateLocalStats();
  }
}
```

## 🔍 Filtros e Consultas Avançadas

### Filtros por Data
```javascript
async getAgendamentosByDate(date) {
  const { data } = await supabase
    .from('agendamentos')
    .select(`
      *,
      clientes (nome, contato),
      servicos (tipo)
    `)
    .eq('data', date)
    .order('time', { ascending: true });
  
  return this.transformData(data);
}
```

### Filtros por Status
```javascript
async getAgendamentosPendentes() {
  return this.getAgendamentosByStatus('pendente');
}

async getAgendamentosConcluidos() {
  return this.getAgendamentosByStatus('concluido');
}
```

## 🚀 Otimizações de Performance

### 1. **Caching**
```javascript
class DatabaseService {
  constructor() {
    this.cache = new Map();
    this.cacheTimeout = 5 * 60 * 1000; // 5 minutos
  }
  
  async getCachedData(key, fetchFunction) {
    const cached = this.cache.get(key);
    if (cached && Date.now() - cached.timestamp < this.cacheTimeout) {
      return cached.data;
    }
    
    const data = await fetchFunction();
    this.cache.set(key, { data, timestamp: Date.now() });
    return data;
  }
}
```

### 2. **Batch Operations**
```javascript
async batchUpdateAgendamentos(updates) {
  const promises = updates.map(update => 
    this.updateAgendamento(update.index, update.data)
  );
  
  return Promise.allSettled(promises);
}
```

### 3. **Lazy Loading**
```javascript
async getAgendamentosPaginated(page = 1, limit = 10) {
  const offset = (page - 1) * limit;
  
  const { data } = await supabase
    .from('agendamentos')
    .select(`
      *,
      clientes (nome, contato),
      servicos (tipo)
    `)
    .range(offset, offset + limit - 1)
    .order('data', { ascending: false });
  
  return this.transformData(data);
}
```

## 🔒 Segurança e Validação

### Row Level Security (RLS)
```sql
-- Política para agendamentos
CREATE POLICY "Usuários podem ver seus agendamentos" ON agendamentos
  FOR SELECT USING (auth.uid() = user_id);

CREATE POLICY "Usuários podem inserir agendamentos" ON agendamentos
  FOR INSERT WITH CHECK (auth.uid() = user_id);
```

### Validação de Dados
```javascript
validateAgendamento(agendamento) {
  const errors = [];
  
  if (!agendamento.cliente) errors.push('Cliente é obrigatório');
  if (!agendamento.data) errors.push('Data é obrigatória');
  if (!agendamento.time) errors.push('Horário é obrigatório');
  
  if (errors.length > 0) {
    throw new Error(errors.join(', '));
  }
}
```

## 📊 Monitoramento e Logs

### Error Tracking
```javascript
logError(operation, error, context = {}) {
  console.error(`DatabaseService.${operation}:`, {
    error: error.message,
    stack: error.stack,
    context,
    timestamp: new Date().toISOString()
  });
  
  // Enviar para serviço de monitoramento
  // analytics.track('database_error', { operation, error: error.message });
}
```

### Performance Monitoring
```javascript
async withPerformanceTracking(operation, fn) {
  const startTime = performance.now();
  
  try {
    const result = await fn();
    const duration = performance.now() - startTime;
    
    console.log(`${operation} completed in ${duration.toFixed(2)}ms`);
    return result;
  } catch (error) {
    this.logError(operation, error);
    throw error;
  }
}
```

---

**ColdTech** - Sistema de Dados Robusto e Escalável
*Versão 1.0 - Supabase + LocalStorage Hybrid Architecture*
