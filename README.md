# Test Technique - Développeur Full-Stack (Refonte)

## Contexte du Projet

Vous postulez pour un poste de **Développeur Full-Stack** sur une plateforme de **Live Shopping** (shopping en direct). La plateforme permet aux vendeurs de diffuser des vidéos en direct pour présenter leurs produits, avec un système de chat en temps réel, de gestion de panier, et de paiement.

## Stack Technique Actuelle

- **Backend**: Node.js, Express, MongoDB (Mongoose), Redis, Socket.io, LiveKit
- **Frontend**: Next.js 14, React 18, TypeScript, Tailwind CSS
- **Autres**: Docker, Kubernetes, AWS S3

## ⚠️ Important : Environnement de Développement

**Ce test peut être réalisé avec des bases de données locales ou mockées.**

### Option 1 : Bases de données locales (recommandé)
- **MongoDB** : Installer MongoDB localement ou utiliser Docker
- **Redis** : Installer Redis localement ou utiliser Docker
- Pas besoin de serveur Socket.io externe, vous pouvez créer un serveur local

### Option 2 : Mock des données (si installation difficile)
- Utiliser des données mockées en mémoire pour MongoDB
- Utiliser un cache en mémoire pour Redis
- Simuler Socket.io avec des événements locaux

**Note** : L'objectif est d'évaluer votre code, pas votre capacité à configurer des environnements complexes. Choisissez l'option la plus confortable pour vous.

## Objectif du Test

Ce test évalue vos compétences en développement full-stack pour la **refonte** de certaines fonctionnalités existantes. Vous devrez démontrer votre capacité à :
- Comprendre et améliorer du code existant
- Implémenter des fonctionnalités backend robustes
- Créer des interfaces utilisateur modernes et réactives
- Gérer la communication temps réel (WebSockets)
- Optimiser les performances et la scalabilité

---

## Partie 1 : Backend (Node.js/Express) - 2h

### Exercice 1.1 : API REST - Gestion des Produits Featured (45 min)

**Contexte** : Dans un événement live, un produit peut être "featured" (mis en avant) par le vendeur. Vous devez créer une API REST pour gérer cette fonctionnalité.

**Tâches** :
1. Créer un endpoint `POST /api/live-events/:eventId/products/:productId/feature` qui met en avant un produit
2. Créer un endpoint `DELETE /api/live-events/:eventId/products/:productId/feature` qui retire un produit de la mise en avant
3. Créer un endpoint `GET /api/live-events/:eventId/products/featured` qui retourne le produit actuellement mis en avant
4. Implémenter une validation avec Joi pour vérifier que :
   - L'événement existe et est actif
   - Le produit appartient à l'événement
   - Un seul produit peut être featured à la fois
5. Utiliser Redis pour mettre en cache le produit featured pendant 5 minutes
   - **Si Redis n'est pas disponible** : Utiliser un cache en mémoire (Map avec TTL) ou un package comme `node-cache`
6. Émettre un événement Socket.io `product:featured` et `product:unfeatured` pour notifier les clients en temps réel
   - **Si Socket.io n'est pas configuré** : Simuler avec des événements émis localement ou documenter comment cela fonctionnerait

**Structure attendue** :
```
backend/
  src/
    routes/
      productFeatureRoutes.js
    controllers/
      productFeatureController.js
    services/
      productFeatureService.js
    middleware/
      validation.js (si nécessaire)
```

**Critères d'évaluation** :
- Qualité du code (lisibilité, structure)
- Gestion des erreurs
- Validation des données
- Utilisation appropriée de Redis
- Émission d'événements Socket.io
- Tests unitaires (bonus)

---

### Exercice 1.2 : Optimisation de Requête MongoDB (30 min)

**Contexte** : La requête suivante est lente et doit être optimisée. Analyséz-la et proposez une solution.

**Requête actuelle** :
```javascript
const getLiveEventStats = async (eventId) => {
  const event = await LiveEvent.findById(eventId)
    .populate('participants.sellerId')
    .populate('products.productId')
    .exec();
  
  const stats = {
    totalViewers: event.viewerCount || 0,
    totalSales: 0,
    totalProducts: event.products.length,
    participants: []
  };
  
  for (const participant of event.participants) {
    const orders = await Order.find({ 
      liveEventId: eventId,
      sellerId: participant.sellerId 
    });
    
    const participantSales = orders.reduce((sum, order) => sum + order.total, 0);
    stats.totalSales += participantSales;
    stats.participants.push({
      sellerId: participant.sellerId,
      sales: participantSales
    });
  }
  
  return stats;
};
```

**Tâches** :
1. Identifier les problèmes de performance
2. Réécrire la fonction en utilisant des agrégations MongoDB
   - **Si MongoDB n'est pas disponible** : Utiliser des données mockées et simuler les agrégations avec du JavaScript
3. Ajouter des index appropriés (définir les index nécessaires)
   - Documenter les index à créer même si vous utilisez des données mockées
4. Implémenter un système de cache avec Redis (TTL de 1 minute)
   - **Si Redis n'est pas disponible** : Utiliser un cache en mémoire
5. Documenter votre approche

**Critères d'évaluation** :
- Compréhension des problèmes de performance
- Utilisation efficace des agrégations MongoDB
- Définition d'index appropriés
- Mise en cache intelligente
- Documentation claire

---

### Exercice 1.3 : Middleware de Rate Limiting Avancé (45 min)

**Contexte** : Vous devez créer un middleware de rate limiting intelligent qui :
- Limite les requêtes par IP
- Limite les requêtes par utilisateur authentifié
- Applique des limites différentes selon le type d'endpoint
- Bloque temporairement les IPs suspectes après plusieurs violations

**Tâches** :
1. Créer un middleware `advancedRateLimiter.js` qui utilise Redis
2. Implémenter 3 niveaux de limitation :
   - **Strict** : 10 req/min (endpoints sensibles comme paiement)
   - **Normal** : 60 req/min (endpoints standards)
   - **Loose** : 200 req/min (endpoints publics)
3. Implémenter un système de "ban" temporaire :
   - Après 5 violations dans une fenêtre de 15 minutes, bloquer l'IP pendant 30 minutes
   - Logger les tentatives de requêtes bloquées
4. Créer un endpoint `GET /api/admin/rate-limit/stats` pour visualiser les statistiques (admin uniquement)
5. Ajouter des headers HTTP appropriés (`X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`)

**Structure attendue** :
```javascript
// Exemple d'utilisation
router.post('/api/payment/process', 
  advancedRateLimiter('strict'),
  auth,
  processPayment
);

router.get('/api/products', 
  advancedRateLimiter('normal'),
  getProducts
);
```

**Critères d'évaluation** :
- Architecture du middleware
- Gestion de Redis pour le rate limiting
- Système de ban intelligent
- Headers HTTP standards
- Gestion des cas limites

---

## Partie 2 : Frontend (Next.js/React/TypeScript) - 2h30

### Exercice 2.1 : Composant de Chat Temps Réel Amélioré (1h)

**Contexte** : Vous devez améliorer le composant de chat existant en ajoutant de nouvelles fonctionnalités.

**Tâches** :
1. Créer un composant `EnhancedChat.tsx` avec les fonctionnalités suivantes :
   - Affichage des messages en temps réel via Socket.io
   - Système de réponses (reply) aux messages
   - Indicateur de "typing..." quand un utilisateur tape
   - Badge de messages non lus
   - Auto-scroll vers le dernier message
   - Filtrage des messages par utilisateur (recherche)
   - Émojis réactions sur les messages (👍, ❤️, 😂, 🔥)
   - Mode sombre/clair
   
2. Gérer les états de chargement et d'erreur
3. Optimiser les performances avec `useMemo` et `useCallback`
4. Implémenter la virtualisation pour les longues listes de messages (bonus)
5. Ajouter des animations fluides avec Framer Motion ou CSS transitions

**Structure attendue** :
```
frontend/
  components/
    chat/
      EnhancedChat.tsx
      ChatMessage.tsx
      ChatInput.tsx
      EmojiPicker.tsx
      TypingIndicator.tsx
  hooks/
    useChat.ts
    useSocket.ts
```

**Critères d'évaluation** :
- Architecture des composants React
- Gestion d'état (hooks personnalisés)
- Intégration Socket.io
- Performance et optimisations
- UX/UI moderne et responsive
- Gestion des erreurs

---

### Exercice 2.2 : Page de Dashboard Vendeur (1h)

**Contexte** : Créer une page de dashboard pour les vendeurs avec des statistiques et graphiques.

**Tâches** :
1. Créer une page `/dashboard/seller` avec :
   - Vue d'ensemble des statistiques (ventes totales, événements live, produits vendus)
   - Graphique des ventes sur les 30 derniers jours (Chart.js ou Recharts)
   - Liste des événements live à venir avec possibilité de les modifier
   - Tableau des produits les plus vendus
   - Section de notifications récentes
   
2. Utiliser Server-Side Rendering (SSR) pour les données initiales
3. Implémenter la pagination pour les tableaux
4. Ajouter des filtres (date, statut, etc.)
5. Créer des composants réutilisables pour les cartes de statistiques
6. Responsive design (mobile-first)

**Structure attendue** :
```
frontend/
  app/
    (dashboard)/
      seller/
        page.tsx
  components/
    dashboard/
      StatsCard.tsx
      SalesChart.tsx
      UpcomingEvents.tsx
      TopProducts.tsx
      NotificationsList.tsx
```

**Critères d'évaluation** :
- Architecture Next.js (SSR, API routes)
- Composants réutilisables
- Visualisation de données
- Responsive design
- Performance

---

### Exercice 2.3 : Optimisation de Performance (30 min)

**Contexte** : Optimiser une page qui charge lentement.

**Code à optimiser** :
```tsx
'use client';

export default function ProductListPage() {
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  
  useEffect(() => {
    fetch('/api/products')
      .then(res => res.json())
      .then(data => {
        setProducts(data);
        setLoading(false);
      });
  }, []);
  
  const filteredProducts = products.filter(p => 
    p.name.toLowerCase().includes(searchTerm.toLowerCase())
  );
  
  return (
    <div>
      {filteredProducts.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
}
```

**Tâches** :
1. Identifier les problèmes de performance
2. Optimiser le code avec :
   - Code splitting
   - Lazy loading
   - Mémoïsation
   - Debouncing pour la recherche
   - Virtualisation si nécessaire
3. Implémenter le SSR ou SSG si approprié
4. Ajouter un système de cache côté client

**Critères d'évaluation** :
- Identification des problèmes
- Solutions d'optimisation appropriées
- Mesures de performance (Lighthouse score)

---

## Partie 3 : Intégration Temps Réel (Socket.io) - 1h

### Exercice 3.1 : Système de Notifications Temps Réel (1h)

**Contexte** : Créer un système de notifications en temps réel pour les événements live.

**Tâches** :
1. **Backend** : Créer un service de notifications Socket.io qui émet :
   - `notification:new-order` : quand une nouvelle commande est passée
   - `notification:product-featured` : quand un produit est mis en avant
   - `notification:viewer-joined` : quand un nouveau viewer rejoint (pour le vendeur)
   - `notification:low-stock` : quand le stock d'un produit est faible
   
2. **Frontend** : Créer un composant `NotificationCenter.tsx` qui :
   - Affiche les notifications en temps réel
   - Groupe les notifications similaires
   - Permet de marquer comme lues
   - Sauvegarde les notifications dans le localStorage
   - Affiche un badge avec le nombre de notifications non lues
   - Son optionnel pour les notifications importantes

3. Créer un hook `useNotifications.ts` pour gérer l'état des notifications

**Structure attendue** :
```
backend/
  src/
    services/
      notificationService.js
    socket/
      notificationHandlers.js

frontend/
  components/
    notifications/
      NotificationCenter.tsx
      NotificationItem.tsx
  hooks/
    useNotifications.ts
```

**Critères d'évaluation** :
- Architecture Socket.io
- Gestion d'état côté client
- UX des notifications
- Performance (éviter les re-renders inutiles)

---

## Livrables Attendus

1. **Code source complet** dans un dossier `solution/`
2. **README.md** expliquant :
   - Comment lancer le projet
   - Configuration de l'environnement (MongoDB, Redis, etc.)
   - Les choix techniques effectués
   - Les difficultés rencontrées
   - Les améliorations possibles
3. **Documentation** des APIs créées (format OpenAPI/Swagger ou Markdown)
4. **Tests** (unitaires et d'intégration) - bonus mais fortement apprécié

**Note** : Si vous utilisez des mocks au lieu de vrais services, documentez clairement comment migrer vers les services réels. Consultez `SETUP_LOCAL.md` pour des exemples de configuration.

---

## Instructions de Soumission

1. Créer un repository Git (GitHub, GitLab, etc.)
2. Commiter votre code avec des messages clairs
3. Envoyer le lien du repository + un README détaillé
4. Temps estimé total : **5h30** (vous pouvez répartir sur plusieurs jours)

---

## Critères Généraux d'Évaluation

- ✅ **Code Quality** : Lisibilité, structure, conventions
- ✅ **Architecture** : Organisation, séparation des responsabilités
- ✅ **Performance** : Optimisations, scalabilité
- ✅ **Sécurité** : Validation, authentification, rate limiting
- ✅ **Tests** : Couverture, qualité des tests
- ✅ **Documentation** : Clarté, exhaustivité
- ✅ **UX/UI** : Design moderne, responsive, accessibilité

---

## Questions ?

N'hésitez pas à poser des questions si quelque chose n'est pas clair. Nous valorisons la communication et la compréhension du besoin avant l'implémentation.

**Bonne chance ! 🚀**

