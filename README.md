# challenge_partage-de_fichiers_windows

# 📁 Partage de fichiers sécurisé avec droits NTFS et Active Directory

## 🖥️ Objectif
Mettre en place un partage réseau sur Windows Server 2022 avec des permissions NTFS et de partage adaptées aux groupes du domaine. Connecter ce partage depuis un poste client Windows 10.

---

## 🛠️ Partie 1 – Configuration sur le serveur

### 1. Installer le rôle de serveur de fichiers (si non déjà fait)
```powershell
Install-WindowsFeature FS-FileServer
```

### 2. Créer l’arborescence des dossiers
```powershell
New-Item -Path "C:\" -Name "Documents_Entreprise" -ItemType Directory
New-Item -Path "C:\Documents_Entreprise" -Name "RH" -ItemType Directory
New-Item -Path "C:\Documents_Entreprise" -Name "Comptabilité" -ItemType Directory
New-Item -Path "C:\Documents_Entreprise" -Name "Direction" -ItemType Directory
```

### 3. Créer les groupes Active Directory (si nécessaires)
```powershell
New-ADGroup -Name "RH" -GroupScope Global -Path "OU=Groupes,DC=lab,DC=lan"
New-ADGroup -Name "Comptabilité" -GroupScope Global -Path "OU=Groupes,DC=lab,DC=lan"
New-ADGroup -Name "Direction" -GroupScope Global -Path "OU=Groupes,DC=lab,DC=lan"
```

### 4. Appliquer les permissions NTFS
```powershell
# Accès lecture/écriture pour chaque groupe sur son dossier
icacls "C:\Documents_Entreprise\RH" /grant "LAB\RH:(OI)(CI)M"
icacls "C:\Documents_Entreprise\Comptabilité" /grant "LAB\Comptabilité:(OI)(CI)M"
icacls "C:\Documents_Entreprise\Direction" /grant "LAB\Direction:(OI)(CI)M"

# Accès en lecture seule pour tous les utilisateurs du domaine
icacls "C:\Documents_Entreprise" /grant "LAB\Domain Users:(OI)(CI)R"
```

### 5. Créer le partage réseau
```powershell
New-SmbShare -Name "Docs" -Path "C:\Documents_Entreprise" `
  -FullAccess "LAB\Direction" `
  -ReadAccess "LAB\Domain Users"
```

### 6. Vérifier les partages existants
```powershell
Get-SmbShare
```

### 7. Vérifier les permissions de partage
```powershell
Get-SmbShareAccess -Name "Docs"
```

---

## 💻 Partie 2 – Connexion depuis un poste client Windows 10

### 1. Lancer PowerShell en tant qu’utilisateur du domaine

### 2. Connecter un lecteur réseau
```powershell
New-PSDrive -Name "Z" -PSProvider FileSystem -Root "\\SRV-FICHIERS\Docs" -Persist
```

> Remplace `SRV-FICHIERS` par le nom DNS ou NetBIOS de ton serveur.

### 3. Vérifier l'accès
```powershell
Set-Location Z:
Get-ChildItem
```

### 4. Tester les permissions
Selon le groupe de l'utilisateur (`RH`, `Comptabilité`, `Direction` ou juste `Domain Users`), tester :
- Lecture seule ou lecture/écriture dans les bons dossiers
- Interdiction d’accès aux dossiers non autorisés

---

## ✅ Résultat attendu

| Groupe             | Accès au dossier RH | Comptabilité | Direction | Racine (`Z:`) |
|--------------------|---------------------|--------------|-----------|----------------|
| RH                 | Lecture/Écriture    | ❌           | ❌        | Lecture seule  |
| Comptabilité       | ❌                  | Lecture/Écriture | ❌     | Lecture seule  |
| Direction          | Lecture/Écriture    | Lecture/Écriture | Lecture/Écriture | Lecture seule |
| Domain Users (autres) | ❌              | ❌           | ❌        | Lecture seule  |

---

## 🗂️ Remarques

- Il est recommandé de placer les groupes dans une OU spécifique (`OU=Groupes`) pour une gestion plus claire.
- Les utilisateurs doivent être ajoutés aux groupes via `Add-ADGroupMember`.
- Le partage `Docs` peut être déployé automatiquement avec une GPO pour tous les postes.
