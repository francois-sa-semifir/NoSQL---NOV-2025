# CREDIS - Correction

## 1. Instanciation du jeu de données (Produits)

```text
HSET product:p1 name "Clavier mécanique" price 79
HSET product:p2 name "Souris gamer" price 59
HSET product:p3 name "Écran 27 pouces" price 229
HSET product:p4 name "Casque audio" price 129
```

---

## 2. Application du scénario

### Rappel des règles lorsque l'utilisateur ajoute un produit

Si `SADD cart:{user} {product}` retourne **1** (nouveau produit) :

* `INCRBY cart:{user}:total {prix}`
* `LPUSH cart:{user}:events "{user} added {product}"`
* `ZINCRBY leaderboard:spenders {prix} {user}`

---

## 3. Rejouons tout le scénario

### Utilisateur user:001 ajoute p1 et p2

#### Produit p1 (79€) - user:001

```text
SADD cart:user:001 p1          → 1
INCRBY cart:user:001:total 79
LPUSH cart:user:001:events "user:001 added p1"
ZINCRBY leaderboard:spenders 79 user:001
```

#### Produit p2 (59€) - user:001

```text
SADD cart:user:001 p2          → 1
INCRBY cart:user:001:total 59
LPUSH cart:user:001:events "user:001 added p2"
ZINCRBY leaderboard:spenders 59 user:001
```

**Total user:001 = 79 + 59 = 138€**

---

### Utilisateur user:002 ajoute p3, p4, p1

#### Produit p3 (229€) - user:002

```text
SADD cart:user:002 p3          → 1
INCRBY cart:user:002:total 229
LPUSH cart:user:002:events "user:002 added p3"
ZINCRBY leaderboard:spenders 229 user:002
```

#### Produit p4 (129€) - user:002

```text
SADD cart:user:002 p4          → 1
INCRBY cart:user:002:total 129
LPUSH cart:user:002:events "user:002 added p4"
ZINCRBY leaderboard:spenders 129 user:002
```

#### Produit p1 (79€) - user:002

```text
SADD cart:user:002 p1          → 1
INCRBY cart:user:002:total 79
LPUSH cart:user:002:events "user:002 added p1"
ZINCRBY leaderboard:spenders 79 user:002
```

**Total user:002 = 229 + 129 + 79 = 437€**

---

### Utilisateur user:003 ajoute p2, p2, puis p4

#### Premier ajout de p2 (59€) - user:003

```text
SADD cart:user:003 p2          → 1
INCRBY cart:user:003:total 59
LPUSH cart:user:003:events "user:003 added p2"
ZINCRBY leaderboard:spenders 59 user:003
```

#### Deuxième ajout de p2 → ignoré

```text
SADD cart:user:003 p2          → 0
```

Aucun compteur ou total n’est mis à jour.

#### Ajout de p4 (129€) - user:003

```text
SADD cart:user:003 p4          → 1
INCRBY cart:user:003:total 129
LPUSH cart:user:003:events "user:003 added p4"
ZINCRBY leaderboard:spenders 129 user:003
```

**Total user:003 = 59 + 129 = 188€**

---

## 4. Résultats attendus

### Question 1 - Totaux par utilisateur

```text
GET cart:user:001:total   → 138
GET cart:user:002:total   → 437
GET cart:user:003:total   → 188
```

**Réponse :**

* user:001 = **138€**
* user:002 = **437€**
* user:003 = **188€**

**Le client le plus dépensier est user:002**

---

### Question 2 - Vérifier les paniers (Sets)

Exemple pour user:001 :

```text
SMEMBERS cart:user:001     → p1, p2
SCARD cart:user:001        → 2
```

Résultats attendus :

* user:001 → {p1, p2}
* user:002 → {p3, p4, p1}
* user:003 → {p2, p4}

Aucun doublon grâce au Set.

---

### Question 3 - Classement des meilleurs dépensiers (Sorted Set)

Commande :

```text
ZREVRANGE leaderboard:spenders 0 -1 WITHSCORES
```

Résultat attendu :

```text
1) "user:002"  → 437
2) "user:003"  → 188
3) "user:001"  → 138
```

**Classement final :**

1. user:002 (437€)
2. user:003 (188€)
3. user:001 (138€)

---

### Question 4 - Historique d’un utilisateur (List)

Exemple pour user:002 :

```text
LRANGE cart:user:002:events 0 10
```

Résultat attendu (ordre chronologique inversé car LPUSH) :

```REDIS
1) "user:002 added p1"
2) "user:002 added p4"
3) "user:002 added p3"
```

Les événements sont bien enregistrés dans le bon ordre.
