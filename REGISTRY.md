# Registry — Adresses par Network

Le Registry est le contrat partagé qui mappe `EOA → DA wallet`.

**Un seul Registry par network**, utilisé par tous les projets et relayers.  
Cela signifie qu'un utilisateur crée son DA wallet une seule fois et peut l'utiliser sur tous les projets intégrés.

---

## Adresses

| Network | Registry address |
|---------|-----------------|
| Testnet | `TBD` |
| Mainnet | `TBD` |

---

## Utilisation

Utilisez cette adresse partout :

| Où | Variable |
|----|----------|
| Relayer `.env` | `REGISTRY_ADDRESS` |
| SDK frontend | `registry` dans `getActiveDAOrNull(nodeUrl, { registry, eoa })` |

---

## Notes

- Ne déployez **pas** votre propre Registry sauf si vous créez un écosystème séparé
- Un Registry différent = DA wallets séparés (les utilisateurs devraient créer un nouveau DA)
