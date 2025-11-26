# Configuration de l'Environnement Local

Ce guide vous aide à configurer un environnement local pour réaliser le test technique sans dépendances externes.

## Option 1 : Avec Docker (Recommandé - Le plus simple)

### Prérequis
- Docker et Docker Compose installés

### Setup rapide

1. Créer un fichier `docker-compose.yml` à la racine de votre projet :

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7
    container_name: test-mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - mongodb_data:/data/db

  redis:
    image: redis:7-alpine
    container_name: test-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  mongodb_data:
```

2. Démarrer les services :
```bash
docker-compose up -d
```

3. Vérifier que les services sont démarrés :
```bash
docker ps
```

4. Dans votre code, utiliser les connexions suivantes :
```javascript
// MongoDB
mongoose.connect('mongodb://admin:password@localhost:27017/liveshopping?authSource=admin');

// Redis
const redis = require('redis');
const client = redis.createClient({
  url: 'redis://localhost:6379'
});
```

---

## Option 2 : Installation Locale

### MongoDB

**macOS** :
```bash
brew tap mongodb/brew
brew install mongodb-community
brew services start mongodb-community
```

**Linux** :
```bash
sudo apt-get install mongodb
sudo systemctl start mongodb
```

**Windows** :
Télécharger depuis https://www.mongodb.com/try/download/community

### Redis

**macOS** :
```bash
brew install redis
brew services start redis
```

**Linux** :
```bash
sudo apt-get install redis-server
sudo systemctl start redis
```

**Windows** :
Utiliser WSL ou Docker

---

## Option 3 : Mock des Services (Sans Installation)

Si vous préférez ne pas installer MongoDB et Redis, voici comment les mocker :

### Mock MongoDB avec données en mémoire

```javascript
// mock-database.js
class MockDatabase {
  constructor() {
    this.collections = {
      liveEvents: [],
      orders: [],
      products: []
    };
  }

  async findById(collection, id) {
    return this.collections[collection].find(item => item._id === id);
  }

  async find(collection, query = {}) {
    let results = this.collections[collection];
    
    // Filtrage simple
    if (query.liveEventId) {
      results = results.filter(item => item.liveEventId === query.liveEventId);
    }
    
    return results;
  }

  async aggregate(collection, pipeline) {
    // Implémentation simplifiée des agrégations
    let results = [...this.collections[collection]];
    
    for (const stage of pipeline) {
      if (stage.$match) {
        // Filtrer selon $match
      }
      if (stage.$group) {
        // Grouper selon $group
      }
      // ... autres stages
    }
    
    return results;
  }

  async save(collection, data) {
    if (!data._id) {
      data._id = `mock_${Date.now()}_${Math.random()}`;
    }
    this.collections[collection].push(data);
    return data;
  }

  async update(collection, query, update) {
    const index = this.collections[collection].findIndex(
      item => item._id === query._id
    );
    if (index >= 0) {
      this.collections[collection][index] = {
        ...this.collections[collection][index],
        ...update
      };
      return this.collections[collection][index];
    }
    return null;
  }
}

module.exports = new MockDatabase();
```

### Mock Redis avec Map

```javascript
// mock-redis.js
class MockRedis {
  constructor() {
    this.data = new Map();
    this.ttl = new Map(); // Map<key, expirationTimestamp>
  }

  async get(key) {
    if (this.ttl.has(key) && this.ttl.get(key) < Date.now()) {
      this.data.delete(key);
      this.ttl.delete(key);
      return null;
    }
    return this.data.get(key) || null;
  }

  async set(key, value, options = {}) {
    this.data.set(key, value);
    
    if (options.EX) {
      // EX = expiration en secondes
      this.ttl.set(key, Date.now() + (options.EX * 1000));
    }
    
    return 'OK';
  }

  async del(key) {
    this.data.delete(key);
    this.ttl.delete(key);
    return 1;
  }

  async exists(key) {
    return this.data.has(key) ? 1 : 0;
  }
}

module.exports = new MockRedis();
```

### Utilisation dans le code

```javascript
// config/database.js
const isDevelopment = process.env.NODE_ENV !== 'production';
const useMock = process.env.USE_MOCK_DB === 'true';

if (useMock || isDevelopment) {
  const mockDb = require('./mock-database');
  module.exports = mockDb;
} else {
  const mongoose = require('mongoose');
  // Connexion MongoDB réelle
}

// config/redis.js
const isDevelopment = process.env.NODE_ENV !== 'production';
const useMock = process.env.USE_MOCK_REDIS === 'true';

if (useMock || isDevelopment) {
  const mockRedis = require('./mock-redis');
  module.exports = mockRedis;
} else {
  const redis = require('redis');
  // Connexion Redis réelle
}
```

### Mock Socket.io

Pour les tests, vous pouvez simuler Socket.io :

```javascript
// mock-socket.js
class MockSocket {
  constructor() {
    this.events = new Map();
    this.rooms = new Set();
  }

  on(event, callback) {
    if (!this.events.has(event)) {
      this.events.set(event, []);
    }
    this.events.get(event).push(callback);
  }

  emit(event, data) {
    console.log(`[Socket.io Mock] Emitting: ${event}`, data);
    // Dans un vrai serveur, cela enverrait aux clients connectés
  }

  join(room) {
    this.rooms.add(room);
  }

  leave(room) {
    this.rooms.delete(room);
  }

  to(room) {
    return {
      emit: (event, data) => {
        console.log(`[Socket.io Mock] To room ${room}: ${event}`, data);
      }
    };
  }
}

class MockIO {
  constructor() {
    this.sockets = [];
  }

  on(event, callback) {
    if (event === 'connection') {
      // Simuler une connexion
      const socket = new MockSocket();
      this.sockets.push(socket);
      callback(socket);
    }
  }

  emit(event, data) {
    console.log(`[Socket.io Mock] Broadcast: ${event}`, data);
  }
}

module.exports = new MockIO();
```

---

## Données de Test

Créez un fichier `seed-data.js` pour initialiser vos données :

```javascript
// seed-data.js
const mockDb = require('./mock-database');

async function seedData() {
  // Événements live
  await mockDb.save('liveEvents', {
    _id: 'evt_001',
    title: 'Vente Flash - Mode Été',
    status: 'live',
    startTime: new Date(),
    participants: [
      {
        sellerId: 123,
        storeName: 'Fashion Store',
        sellerName: 'Marie Dupont',
        status: 'active'
      }
    ],
    products: [
      {
        productId: 456,
        name: 'Robe d\'été',
        price: 59.99,
        salePrice: 39.99,
        featuredTime: null
      }
    ],
    viewerCount: 150,
    totalSales: 0
  });

  // Commandes
  await mockDb.save('orders', {
    _id: 'order_001',
    liveEventId: 'evt_001',
    sellerId: 123,
    userId: 456,
    total: 45.98,
    status: 'completed',
    createdAt: new Date()
  });

  console.log('✅ Données de test initialisées');
}

module.exports = seedData;
```

---

## Structure de Projet Recommandée

```
test-fullstack/
├── backend/
│   ├── src/
│   │   ├── config/
│   │   │   ├── database.js (ou mock-database.js)
│   │   │   └── redis.js (ou mock-redis.js)
│   │   ├── controllers/
│   │   ├── services/
│   │   └── routes/
│   ├── docker-compose.yml (optionnel)
│   └── package.json
└── frontend/
    └── ...
```

---

## Commandes Utiles

### Avec Docker
```bash
# Démarrer les services
docker-compose up -d

# Voir les logs
docker-compose logs -f

# Arrêter les services
docker-compose down

# Réinitialiser les données
docker-compose down -v
docker-compose up -d
```

### Avec MongoDB local
```bash
# Se connecter à MongoDB
mongosh mongodb://localhost:27017

# Voir les bases de données
show dbs

# Utiliser une base
use liveshopping

# Voir les collections
show collections
```

### Avec Redis local
```bash
# Se connecter à Redis
redis-cli

# Tester une commande
GET mykey

# Voir toutes les clés
KEYS *
```

---

## Variables d'Environnement

Créez un fichier `.env` :

```env
# Option 1 : Avec services réels
MONGODB_URI=mongodb://admin:password@localhost:27017/liveshopping?authSource=admin
REDIS_URL=redis://localhost:6379
NODE_ENV=development

# Option 2 : Avec mocks
USE_MOCK_DB=true
USE_MOCK_REDIS=true
USE_MOCK_SOCKET=true
```

---

## Notes Importantes

1. **Documentation** : Si vous utilisez des mocks, documentez clairement comment remplacer par les vrais services
2. **Tests** : Les mocks facilitent l'écriture de tests unitaires
3. **Performance** : Les mocks peuvent être plus rapides pour le développement, mais testez aussi avec les vrais services si possible
4. **Socket.io** : Pour le frontend, vous pouvez créer un serveur Socket.io local simple pour tester

---

## Aide Supplémentaire

Si vous rencontrez des difficultés avec l'installation :
- Utilisez les mocks fournis ci-dessus
- Documentez votre approche dans le README
- Expliquez comment migrer vers les vrais services

L'objectif est d'évaluer votre code, pas votre capacité à configurer des environnements complexes ! 🚀

