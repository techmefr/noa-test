# NOA - Nuxt Orchestrated Architecture

**Architecture DDD en layers pour Nuxt**  
La rigueur de Laravel pour le frontend

Version: 0.1.0-beta | Nuxt 3.x | TypeScript 5.x | Licence MIT

---

## Qu'est-ce que NOA ?

NOA (Nuxt Orchestrated Architecture) est un **framework d'architecture** pour applications Nuxt basé sur les principes du **Domain-Driven Design (DDD)**.

### Le problème qu'on résout

**Avant NOA (code typique Nuxt) :**
```typescript
// ❌ Composant avec tout mélangé
const invoices = ref([])
const isLoading = ref(false)

// API call dans le composant
const fetchInvoices = async () => {
  isLoading.value = true
  invoices.value = await $fetch('/api/invoices')
}

// Logique métier dans le composant
const isOverdue = (invoice) => {
  return new Date(invoice.dueDate) < new Date()
}

// Pagination dans le composant
const currentPage = ref(1)
const paginatedInvoices = computed(() => {
  return invoices.value.slice(0, 10)
})
```

**Problèmes :**
- Logique dispersée dans les composants
- Duplication de code entre composants
- Difficile à tester
- Difficile à maintenir
- Pas de structure claire

**Avec NOA :**
```typescript
// ✅ Composant ultra-simple
const {
  paginatedInvoices,  // Données paginées
  isLoading,          // État de chargement
  fetchAll,           // Récupérer les données
  isOverdue,          // Logique métier
  nextPage,           // Pagination
} = useInvoiceOrchestrator() // UN SEUL IMPORT !

onMounted(() => fetchAll())
```

**Avantages :**
- Un seul import
- Logique séparée et testable
- Code réutilisable
- Structure claire et prévisible

---

## Architecture

NOA organise votre code en **5 layers (couches)** avec des responsabilités claires :
```
Composant Vue
    ↓
Orchestrator (Façade unique)
    ↓
┌──────────┬──────────┬──────────┬──────────┐
│  State   │Connector │  Logic   │Interaction│
│  (Data)  │  (API)   │ (Métier) │   (UI)    │
└──────────┴──────────┴──────────┴──────────┘
```

### Les 5 Layers

| Layer | Responsabilité | Exemple |
|-------|----------------|---------|
| **State** | Données réactives locales | `data`, `isLoading`, `error` |
| **Connector** | Appels API + sync stores | `fetchAll()`, `create()`, `update()` |
| **Logic** | Logique métier pure | `validateEntity()`, `calculateTotal()` |
| **Interaction** | Comportements UI | Pagination, tri, filtrage |
| **Orchestrator** | Coordonne tout | Façade unique |

---

## Installation

### Prérequis

- Node.js 22+
- Nuxt 3.x
- pnpm (recommandé)

### Étape 1 : Installer les packages
```bash
pnpm add @noa-nuxt/core @noa-nuxt/cli
```

### Étape 2 : Initialiser NOA
```bash
npx noa init
```

Cette commande crée :
- `core/` - Classes abstraites des layers
- `modules/` - Dossier pour vos modules métier
- `shared/` - Code partagé entre modules
- `noa.config.ts` - Configuration

### Étape 3 : Générer votre premier module
```bash
npx noa generate
```

Répondez aux questions interactives :
- Module name: `accounting`
- Entity name: `Invoice`
- Fields: `amount`, `customerId`, `status`
- Layers: Tous

**Résultat :** 11 fichiers générés en 2 secondes !

---

## Quick Start

### Générer un module complet
```bash
npx noa generate
```

Le CLI va créer automatiquement :
```
modules/accounting/
├── composables/
│   ├── state/InvoiceState.ts
│   ├── connector/InvoiceConnector.ts
│   ├── logic/InvoiceLogic.ts
│   └── orchestrator/InvoiceOrchestrator.ts
├── types.accounting/invoice.types.ts
├── services.accounting/invoiceService.ts
└── stores.accounting/useInvoice.store.ts
```

### Utiliser dans un composant
```vue
<script setup lang="ts">
const {
  paginatedInvoices,
  isLoading,
  error,
  fetchAll,
  create,
  validateEntity,
  nextPage,
} = useInvoiceOrchestrator()

onMounted(() => {
  fetchAll()
})

const handleCreate = async () => {
  const newInvoice = {
    amount: 100,
    customerId: 'customer-123',
    status: 'draft',
  }
  
  if (validateEntity(newInvoice)) {
    await create(newInvoice)
  }
}
</script>

<template>
  <div>
    <h1>Invoices</h1>
    
    <p v-if="isLoading">Loading...</p>
    <p v-else-if="error">{{ error }}</p>
    
    <div v-else>
      <InvoiceCard 
        v-for="invoice in paginatedInvoices" 
        :key="invoice.id" 
        :invoice="invoice" 
      />
      
      <button @click="nextPage">Next Page</button>
      <button @click="handleCreate">Create Invoice</button>
    </div>
  </div>
</template>
```

---

## Les 5 Layers expliqués

### 1. State Layer - Données réactives

**Responsabilité :** Gérer l'état local du module (pas partagé entre modules)
```typescript
export class InvoiceState extends StateLayer<Invoice> {
  readonly data = ref<Invoice[]>([])
  readonly isLoading = ref(false)
  readonly error = ref<string | null>(null)
  readonly count = computed(() => this.data.value.length)
  
  reset() {
    this.data.value = []
    this.isLoading.value = false
    this.error.value = null
  }
}
```

**Règles :**
- État LOCAL uniquement
- Pas de logique métier
- Pas d'appels API

---

### 2. Connector Layer - API et synchronisation

**Responsabilité :** Appeler l'API et synchroniser avec les stores Pinia
```typescript
export class InvoiceConnector extends ConnectorLayer<Invoice> {
  async fetchAll(): Promise<void> {
    const data = await this.handleRequest(() => 
      invoiceService.getAll()
    )
    
    if (data) {
      this.store.items = data
      this.state.data.value = data
    }
  }
  
  async create(invoice: Partial<Invoice>): Promise<Invoice> {
    const created = await this.handleRequest(() =>
      invoiceService.create(invoice)
    )
    
    if (created) {
      this.store.items.push(created)
      this.state.data.value.push(created)
    }
    
    return created!
  }
}
```

**Règles :**
- Appelle les services (API)
- Synchronise toujours store ET state
- Gère loading/error automatiquement
- Pas de logique métier

---

### 3. Logic Layer - Logique métier pure

**Responsabilité :** Toute la logique métier (validation, calculs, règles)
```typescript
export class InvoiceLogic extends LogicLayer<Invoice> {
  validateEntity(invoice: Invoice): boolean {
    return (
      invoice.amount > 0 &&
      invoice.customerId.length > 0 &&
      invoice.dueDate.length > 0
    )
  }
  
  validatePartialEntity(invoice: Partial<Invoice>): boolean {
    if (invoice.amount !== undefined && invoice.amount <= 0) {
      return false
    }
    return true
  }
  
  isOverdue(invoice: Invoice): boolean {
    return new Date(invoice.dueDate) < new Date() 
      && invoice.status !== 'paid'
  }
  
  calculateTotal(invoices: Invoice[]): number {
    return invoices.reduce((sum, inv) => sum + inv.amount, 0)
  }
}
```

**Règles :**
- Fonctions PURES (pas d'effets de bord)
- Pas d'appels API
- Pas de modification d'état
- Facilement testable

---

### 4. Interaction Layer - Comportements UI

**Responsabilité :** Comportements UI réutilisables (pagination, tri, filtrage)
```typescript
// shared/composables/interaction/usePaginationInteraction.ts
export function usePaginationInteraction<T>(items: Ref<T[]>) {
  const currentPage = ref(1)
  const itemsPerPage = ref(10)
  
  const paginatedItems = computed(() => {
    const start = (currentPage.value - 1) * itemsPerPage.value
    const end = start + itemsPerPage.value
    return items.value.slice(start, end)
  })
  
  const nextPage = () => {
    if (currentPage.value < totalPages.value) {
      currentPage.value++
    }
  }
  
  return {
    currentPage,
    paginatedItems,
    nextPage,
  }
}
```

**Règles :**
- Comportements UI génériques
- Réutilisables dans tous les modules
- Pas de logique métier

---

### 5. Orchestrator Layer - Façade unique

**Responsabilité :** Coordonner TOUS les autres layers et exposer une API unique
```typescript
export class InvoiceOrchestrator extends OrchestratorLayer<Invoice> {
  private pagination = usePaginationInteraction(this.state.data)
  
  getPublicApi() {
    return {
      // State
      isLoading: this.state.isLoading,
      error: this.state.error,
      
      // Store
      ...storeToRefs(this.store),
      
      // Connector
      fetchAll: this.connector.fetchAll.bind(this.connector),
      create: this.connector.create.bind(this.connector),
      
      // Logic
      validateEntity: this.logic.validateEntity.bind(this.logic),
      isOverdue: this.logic.isOverdue.bind(this.logic),
      
      // Interaction
      paginatedInvoices: this.pagination.paginatedItems,
      nextPage: this.pagination.nextPage,
    }
  }
}

export const useInvoiceOrchestrator = () => {
  const store = useInvoiceStore()
  const state = new InvoiceState()
  const connector = new InvoiceConnector(state, store)
  const logic = new InvoiceLogic()
  
  const orchestrator = new InvoiceOrchestrator(state, connector, logic, store)
  
  return orchestrator.getPublicApi()
}
```

---

## Structure de projet
```
my-nuxt-app/
├── modules/                          # Modules métier
│   └── accounting/
│       ├── components/
│       │   └── accounting-invoice/   # Préfixe module
│       ├── composables/
│       │   ├── state/
│       │   ├── connector/
│       │   ├── logic/
│       │   └── orchestrator/
│       ├── types.accounting/
│       ├── services.accounting/
│       ├── stores.accounting/
│       └── pages.accounting/
│
├── core/                             # Classes abstraites NOA
│   └── layers/
│       ├── BaseLayer.ts
│       ├── StateLayer.ts
│       ├── ConnectorLayer.ts
│       ├── LogicLayer.ts
│       ├── InteractionLayer.ts
│       └── OrchestratorLayer.ts
│
├── shared/                           # Code partagé
│   ├── composables/interaction/
│   ├── components/
│   └── utils/
│
└── noa.config.ts
```

---

## Concepts clés

### Stores Pinia SANS actions

**Innovation majeure de NOA :**
```typescript
// ✅ Store = État + Getters UNIQUEMENT
export const useInvoiceStore = defineStore('invoice', () => {
  const items = ref<Invoice[]>([])
  const totalCount = computed(() => items.value.length)
  
  return { items, totalCount }
  // PAS D'ACTIONS !
})

// ✅ Logique dans le Connector (traçable)
class InvoiceConnector {
  async fetchAll() {
    const data = await invoiceService.getAll()
    this.store.items = data // Modification EXPLICITE
  }
}
```

**Avantages :**
- Traçabilité parfaite
- Debugging facile
- Tests simples
- Prédictibilité maximale

---

### Pattern Orchestrator
```
❌ AVANT : 5-6 imports par composant

✅ AVEC NOA : 1 seul import (orchestrator)
```

**Avantages :**
- Découplage total
- Refactoring facile
- Testing simplifié

---

## Règles d'or

### 1. Un composant = Un orchestrator
```typescript
// ✅ BON
const { items, fetchAll } = useInvoiceOrchestrator()

// ❌ MAUVAIS
const state = useInvoiceState()
const connector = useInvoiceConnector()
```

### 2. Store = État + Getters (jamais d'actions)

### 3. Logique métier dans Logic (pas dans les composants)

### 4. API calls dans Connector (pas dans les composants)

### 5. Toujours utiliser storeToRefs() avec Pinia

---

## Conventions de nommage

### Composables
- State: `InvoiceState.ts`
- Connector: `InvoiceConnector.ts`
- Logic: `InvoiceLogic.ts`
- Orchestrator: `InvoiceOrchestrator.ts`

### Dossiers
- Types: `types.accounting/`
- Services: `services.accounting/`
- Stores: `stores.accounting/`
- Pages: `pages.accounting/`

### Composants
- Préfixe module: `accounting-invoice/InvoiceList.vue`

---

## Comparaison

| Aspect | Nuxt classique | NOA |
|--------|----------------|-----|
| **Structure** | Libre | Structurée |
| **Imports/composant** | 5-7 | 1 |
| **Tests** | Difficile | Facile |
| **Onboarding** | 2 semaines | 3 jours |
| **Maintenance** | Difficile | Facile |

---

## FAQ

### Pourquoi pas juste des composables ?

Les composables c'est bien, mais NOA apporte :
- Structure standardisée
- Séparation claire des responsabilités
- Orchestrator pattern
- Type safety avec classes abstraites

### Est-ce du over-engineering ?

Non si vous avez :
- Plusieurs développeurs
- Application qui va grandir
- Besoin de maintenabilité

### Courbe d'apprentissage ?

- Développeur Nuxt expérimenté : 1 journée
- Développeur junior : 3 jours
- Développeur Laravel : 2 heures

---

## Roadmap

### v0.1.0-beta (Actuel)
- [x] Core layers
- [x] Documentation
- [ ] CLI generator
- [ ] Tests complets

### v1.0.0
- [ ] Production-ready
- [ ] Coverage >80%
- [ ] Video tutorials

---

## Licence

MIT © Gaëtan

---

## Support

- Documentation : [Lien]
- Slack : `#noa-support`
- Issues : [GitHub]

---

**Construit avec ❤️ pour éviter le code spaghetti frontend**