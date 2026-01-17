# Mappage de Lecteur réseau Personnel

**OBJECTIF** :

Mettre en place un lecteur réseau personnel `(\Perso$)` mappé automatiquement pour chaque utilisateur, **avec création automatique du dossier utilisateur à la connexion**, dans un domaine Active Directory.

---

## Contexte

- Serveur AD / fichiers (exemple) : **STBN-AD01**
- Domaine : **e-ge.lab**
- Partage : **Perso\$**
- Chemin réel sur le serveur : `C:\Shares\Perso`
- Lettre de lecteur : **P:**

⚠️ **Remplace** `STBN-AD01` **et** `e-ge.lab` **par ton nom de serveur et de domaine.**

---

## Problème rencontré

- Le mappage via GPO (`\\serveur\Perso$\%username%`) fonctionne **uniquement si le dossier existe déjà**.
- La GPO *Lecteurs mappés* **ne crée pas de dossier de manière fiable**.

➡️ Solution : **script PowerShell exécuté à l’ouverture de session utilisateur**. 🎉

---

## 1. Configuration NTFS du dossier Perso

Crée un dossier : `C:\Shares\Perso`

### Autorisations NTFS (paramètres de sécurité avancés)

(Clic droit, `propriétés`, `Sécurité`, puis `Avancé`. Supprime toutex les entrées qui ne figure pas, et ajoute `Utilisateurs authentifiés` puis conserve **uniquement** les entrées suivantes. :

| Principal                     | Droits                  | S’applique à                          |
| ----------------------------- | ----------------------- | ------------------------------------- |
| Administrateurs               | Contrôle total          | Ce dossier, sous-dossiers et fichiers |
| SYSTEM                        | Contrôle total          | Ce dossier, sous-dossiers et fichiers |
| **Utilisateurs authentifiés** | Autorisations spéciales | **Ce dossier uniquement**             |
| **CREATOR OWNER**             | Contrôle total          | Sous-dossiers et fichiers             |

*Les captures suivantes montrent la configuration attendue :*
<details>
  <summary>📸︲Cofiguration NTFS du dossier Perso</summary>

---

<img src="capture/proprietes_perso.jpg" />

*Propriétés de : Perso* 


<img src="capture/proprietes_avancee_perso.jpg" />

*Paramètres de sécurité avancés pour Perso*


</details>

---


### Détails pour *Utilisateurs authentifiés*

- Parcours du dossier
- Liste du dossier
- **Création de dossier / ajout de données**
- Lecture des attributs
- Lecture des autorisations
- **S’applique à : Ce dossier uniquement**


*Les captures suivantes montrent la configuration attendue :*
<details>
  <summary>📸︲Autorisations pour Utilisateur authentifiés</summary>

---

<img src="capture/autorisation_utlisateurAuthentifie_perso.jpg" />

*Détails pour Utilisateur authentifiés* 


</details>

---

## 2. Partage du dossier

Partage : **Perso\$**

- Autorisations de partage :
  - `Authenticated Users` ou `Everyone` → **Contrôle total** 
  
Pour ce faire, clique sur `Partage` (dans `Propriétés`), puis `Partage avancé`. Et ensuite, `Autorisations`  \

👉 La sécurité est gérée **uniquement via NTFS**.

*Les captures suivantes montrent la configuration attendue :*
<details>
  <summary>📸︲Autorisation de Partage avancé</summary>

---

<img src="capture/partage_avancee.jpg" />

*Détails pour Partage avancé* 

<img src="capture/partage_avancee_autorisation.png" />

*Autorisation de Partage avancé*

</details>

---

## 3. Script PowerShell de création + mappage

### Emplacement du script

```text
\\STBN-AD01\SYSVOL\e-ge.lab\scripts\
````
Créé ici le fichier `MapPerso.ps1` 

`⚠️` Remplace **STBN-AD01**, par le nom de ta machine.

### Contenu du script

```powershell
# Racine du partage
$ShareRoot = "\\STBN-AD01\Perso$"

# Dossier utilisateur
$UserFolder = Join-Path $ShareRoot $env:USERNAME

# Création du dossier si absent
if (-not (Test-Path $UserFolder)) {
    New-Item -Path $UserFolder -ItemType Directory -Force | Out-Null
}

# Mappage du lecteur P:
if (-not (Get-PSDrive -Name P -ErrorAction SilentlyContinue)) {
    New-PSDrive -Name P -PSProvider FileSystem -Root $UserFolder -Persist -ErrorAction SilentlyContinue
}
```

*Les captures suivantes montrent la configuration attendue :*
<details>
  <summary>📸︲Le Script PowerShell</summary>

---

<img src="capture/script_powershell.jpg" />

*Script PowerShell* 

</details>

---

## 4. Configuration de la GPO

### Emplacement exact

```
Configuration utilisateur
 → Stratégies
   → Paramètres Windows
     → Scripts (ouverture/fermeture de session)
       → Ouverture de session
```

### Paramétrage

- **Script** :

```
powershell.exe
```

- **Paramètres du script** :

```
-ExecutionPolicy Bypass -NoProfile -File "\\STBN-AD01\SYSVOL\e-ge.lab\scripts\MapPerso.ps1"
```
`⚠️` Remplace **STBN-AD01**, par le nom de ta machine.\
`❌` Ne PAS utiliser *Lecteurs mappés* en parallèle.


<details>
  <summary>📸︲Paramétrage Script</summary>

---

<img src="capture/script_propriete_ouvertureSession.jpg" />

*Paramétrage* 

</details>

---

## 5. Test et validation

1. Sur le serveur :

```bash
gpupdate /force
```
2. Sur le pc client :

```bash
gpupdate /force
```
et pour se déconnecter
```bash
logoff
```

### Résultat attendu

- Le dossier au nom du user, dans `C:\Shares\Perso\...` est créé automatiquement
- Le lecteur **P:** est mappé sur `\\STBN-AD01\Perso$\test`
- L’utilisateur n’a accès qu’à **son** dossier

---

## Conclusion

- `✔` Création automatique du répertoire personnel de l'utilisateur
- `✔` Mappage du lecteur à chaque connexion
- `✔` L’accès aux données est strictement limité au dossier personnel de chaque utilisateur, sans visibilité sur les autres répertoires.


