Basándome en la información que proporcionaste, aquí está como quedaría tu GitHub Actions workflow:

name: Upload to Google Drive

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  # También puedes configurarlo para que se ejecute manualmente
  workflow_dispatch:

jobs:
  upload-to-drive:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
      
    # Aquí puedes agregar pasos para generar/preparar tu archivo
    # Por ejemplo, si necesitas compilar algo primero:
    # - name: Build project
    #   run: |
    #     # tus comandos de build aquí
    
    - name: Upload file to Google Drive
      uses: willo32/google-drive-upload-action@v1
      with:
        target: ruta/a/tu/archivo.txt  # Cambia esto por la ruta de tu archivo
        credentials: ${{ secrets.GOOGLE_DRIVE_CREDENTIALS }}
        parent_folder_id: 1UtrSpk5nEtdBUk68Rfr21WaMiYKhDTwP
        name: mi-archivo-subido.txt  # Nombre opcional para el archivo en Drive
        # child_folder: v1.0  # Opcional: subcarpeta donde subir el archivo

## Puntos importantes a considerar:

### 1. **Configuración del secreto**
Asegúrate de que tu secreto en GitHub se llame exactamente `GOOGLE_DRIVE_CREDENTIALS` y contenga las credenciales JSON codificadas en base64.

### 2. **Compartir la carpeta**
Necesitas compartir la carpeta de Google Drive con el service account:
- Ve a la carpeta en Google Drive
- Clic derecho → "Compartir"
- Agrega: `github-actions@fleet-garage-468103-s7.iam.gserviceaccount.com`
- Dale permisos de "Editor"

### 3. **Personalización del workflow**

Puedes personalizar varios aspectos:

**Para subir múltiples archivos:**
```yaml
- name: Upload multiple files
  uses: willo32/google-drive-upload-action@v1
  with:
    target: archivo1.txt
    credentials: ${{ secrets.GOOGLE_DRIVE_CREDENTIALS }}
    parent_folder_id: 1UtrSpk5nEtdBUk68Rfr21WaMiYKhDTwP

- name: Upload another file
  uses: willo32/google-drive-upload-action@v1
  with:
    target: archivo2.pdf
    credentials: ${{ secrets.GOOGLE_DRIVE_CREDENTIALS }}
    parent_folder_id: 1UtrSpk5nEtdBUk68Rfr21WaMiYKhDTwP
```

**Para organizar por versiones:**
```yaml
- name: Upload with version folder
  uses: willo32/google-drive-upload-action@v1
  with:
    target: mi-app.zip
    credentials: ${{ secrets.GOOGLE_DRIVE_CREDENTIALS }}
    parent_folder_id: 1UtrSpk5nEtdBUk68Rfr21WaMiYKhDTwP
    child_folder: ${{ github.ref_name }}  # Usa el nombre de la rama
```

### 4. **Triggers comunes**

Puedes cambiar cuándo se ejecuta:
```yaml
# Solo en releases
on:
  release:
    types: [published]

# Solo en tags
on:
  push:
    tags:
      - 'v*'

# Programado (ej: diario a las 2 AM UTC)
on:
  schedule:
    - cron: '0 2 * * *'
```


Para crear las credenciales de service account necesarias para esta acción de GitHub, necesitas seguir estos pasos:

## 1. Crear un Service Account en Google Cloud Console

1. Ve a [Google Cloud Console](https://console.cloud.google.com/)
2. Selecciona o crea un proyecto
3. Ve a **IAM & Admin** → **Service Accounts**
4. Haz clic en **"Create Service Account"**
5. Completa los campos:
   - **Service account name**: Un nombre descriptivo (ej: "github-drive-uploader")
   - **Service account ID**: Se genera automáticamente
   - **Description**: Opcional

## 2. Generar las credenciales JSON

1. Una vez creado el service account, haz clic en él
2. Ve a la pestaña **"Keys"**
3. Haz clic en **"Add Key"** → **"Create new key"**
4. Selecciona **JSON** como formato
5. Haz clic en **"Create"**
6. Se descargará automáticamente un archivo JSON con las credenciales

## 3. Habilitar la API de Google Drive

1. En Google Cloud Console, ve a **APIs & Services** → **Library**
2. Busca "Google Drive API"
3. Haz clic en **"Enable"**

## 4. Codificar las credenciales en Base64

Ejecuta este comando en tu terminal (reemplaza el nombre del archivo):

```bash
base64 mi_service_account_key.json > encoded.txt
```

## 5. Agregar el secreto en GitHub

1. Ve a tu repositorio en GitHub
2. Ve a **Settings** → **Secrets and variables** → **Actions**
3. Haz clic en **"New repository secret"**
4. Nombre: `GOOGLE_DRIVE_CREDENTIALS` (o el nombre que prefieras)
5. Valor: Pega el contenido del archivo `encoded.txt`

## 6. Compartir la carpeta de Drive

1. Ve a Google Drive y localiza la carpeta donde quieres subir archivos
2. Haz clic derecho → **"Share"**
3. Agrega el email del service account (lo encuentras en el archivo JSON descargado, campo `client_email`)
4. Dale permisos de **"Editor"**

## 7. Obtener el ID de la carpeta

El `parent_folder_id` es la parte final de la URL cuando navegas a tu carpeta en Google Drive:
```
https://drive.google.com/drive/folders/1ABC123DEF456GHI789
                                        ↑
                                   Este es el ID
```

¡Ya tienes todo listo para usar la acción en tu workflow de GitHub!
