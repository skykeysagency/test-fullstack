# Exercices Détaillés - Test Full-Stack

## Structure du Projet de Test

```
test-fullstack/
├── README.md (instructions générales)
├── EXERCICES.md (ce fichier)
├── backend-starter/ (structure de base fournie)
│   ├── src/
│   │   ├── models/
│   │   │   └── LiveEvent.js (modèle de base)
│   │   ├── config/
│   │   │   ├── database.js
│   │   │   └── redis.js
│   │   └── server.js
│   └── package.json
└── frontend-starter/ (structure de base fournie)
    ├── app/
    ├── components/
    └── package.json
```

---

## Modèles de Données de Référence

### LiveEvent Model (simplifié)
```javascript
const LiveEventSchema = new Schema({
  title: String,
  description: String,
  startTime: Date,
  endTime: Date,
  status: {
    type: String,
    enum: ['scheduled', 'live', 'ended'],
    default: 'scheduled'
  },
  participants: [{
    sellerId: Number,
    storeName: String,
    sellerName: String,
    status: String
  }],
  products: [{
    productId: Number,
    name: String,
    price: Number,
    salePrice: Number,
    thumbnail: String,
    featuredTime: Date, // null si non featured
    salesCount: Number,
    salesAmount: Number
  }],
  featuredProduct: {
    productId: Number,
    featuredAt: Date
  },
  viewerCount: Number,
  totalSales: Number
});
```

### Order Model (pour les exercices)
```javascript
const OrderSchema = new Schema({
  orderId: String,
  liveEventId: Schema.Types.ObjectId,
  sellerId: Number,
  userId: Number,
  items: [{
    productId: Number,
    quantity: Number,
    price: Number
  }],
  total: Number,
  status: {
    type: String,
    enum: ['pending', 'completed', 'cancelled'],
    default: 'pending'
  },
  createdAt: Date
});
```

---

## Exemples de Code de Référence

### Configuration Socket.io (Backend)
```javascript
// server.js
const { Server } = require('socket.io');
const io = new Server(server, {
  cors: {
    origin: process.env.FRONTEND_URL,
    methods: ['GET', 'POST']
  }
});

io.on('connection', (socket) => {
  console.log('User connected:', socket.id);
  
  socket.on('join:live-event', (eventId) => {
    socket.join(`live-event:${eventId}`);
  });
  
  socket.on('disconnect', () => {
    console.log('User disconnected:', socket.id);
  });
});
```

### Hook Socket.io (Frontend)
```typescript
// hooks/useSocket.ts
import { useEffect, useState } from 'react';
import { io, Socket } from 'socket.io-client';

export const useSocket = (url: string) => {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    const newSocket = io(url);
    
    newSocket.on('connect', () => {
      setConnected(true);
    });
    
    newSocket.on('disconnect', () => {
      setConnected(false);
    });
    
    setSocket(newSocket);
    
    return () => {
      newSocket.close();
    };
  }, [url]);

  return { socket, connected };
};
```

---

## Données de Test

### Exemple de LiveEvent
```json
{
  "_id": "507f1f77bcf86cd799439011",
  "title": "Vente Flash - Mode Été",
  "status": "live",
  "startTime": "2024-01-15T10:00:00Z",
  "participants": [
    {
      "sellerId": 123,
      "storeName": "Fashion Store",
      "sellerName": "Marie Dupont",
      "status": "active"
    }
  ],
  "products": [
    {
      "productId": 456,
      "name": "Robe d'été",
      "price": 59.99,
      "salePrice": 39.99,
      "thumbnail": "https://example.com/robe.jpg",
      "featuredTime": null,
      "salesCount": 0,
      "salesAmount": 0
    }
  ],
  "viewerCount": 150,
  "totalSales": 0
}
```

---

## Checklist de Validation

### Backend
- [ ] Endpoints REST fonctionnels
- [ ] Validation des données (Joi)
- [ ] Gestion des erreurs appropriée
- [ ] Redis configuré et utilisé
- [ ] Socket.io émet les événements
- [ ] Tests unitaires (bonus)
- [ ] Documentation API

### Frontend
- [ ] Composants React fonctionnels
- [ ] TypeScript sans erreurs
- [ ] Intégration Socket.io
- [ ] Responsive design
- [ ] Gestion des états de chargement
- [ ] Optimisations de performance
- [ ] Accessibilité de base

### Intégration
- [ ] Communication temps réel fonctionnelle
- [ ] Gestion des erreurs réseau
- [ ] Reconnexion automatique Socket.io

---

## Ressources Utiles

- [Express.js Documentation](https://expressjs.com/)
- [MongoDB Aggregation](https://www.mongodb.com/docs/manual/aggregation/)
- [Socket.io Documentation](https://socket.io/docs/v4/)
- [Next.js Documentation](https://nextjs.org/docs)
- [React Hooks](https://react.dev/reference/react)
- [Redis Commands](https://redis.io/commands/)

---

## Conseils

1. **Commencez par comprendre** le contexte avant de coder
2. **Planifiez votre architecture** avant d'implémenter
3. **Testez régulièrement** votre code
4. **Documentez** vos choix techniques
5. **Optimisez** progressivement, pas tout d'un coup
6. **Gérez les erreurs** de manière élégante
7. **Pensez à la scalabilité** dès le début

Bon courage ! 💪

