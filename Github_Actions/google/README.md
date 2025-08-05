Para hacer backup a **Google Drive** en lugar de un repositorio, necesitas usar la API de Google Drive. Te muestro cómo adaptarlo:

# Nombre de la acción
name: Backup a Google Drive

on:
  schedule:
    # Ejecuta cada día a las 2:00 AM UTC
    - cron: '0 2 * * *'
  workflow_dispatch: # Permite ejecutar manualmente

jobs:
  backup-to-gdrive:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout repositorio actual
      uses: actions/checkout@v4
      with:
        fetch-depth: 1
        
    - name: Configurar fecha y variables
      id: date
      run: |
        echo "date=$(date +'%Y-%m-%d_%H-%M-%S')" >> $GITHUB_OUTPUT
        echo "short_date=$(date +'%Y-%m-%d')" >> $GITHUB_OUTPUT
        echo "year=$(date +'%Y')" >> $GITHUB_OUTPUT
        echo "month=$(date +'%m-%B')" >> $GITHUB_OUTPUT
        
    - name: Crear estructura de backup
      id: create_backup
      run: |
        mkdir -p backup-data
        
        # Información del repositorio
        echo "=== BACKUP DEL REPOSITORIO ${{ github.repository }} ===" > backup-data/backup-info.md
        echo "" >> backup-data/backup-info.md
        echo "## 📊 Información del Backup" >> backup-data/backup-info.md
        echo "- **Fecha**: $(date)" >> backup-data/backup-info.md
        echo "- **Repositorio origen**: ${{ github.repository }}" >> backup-data/backup-info.md
        echo "- **Commit actual**: $(git rev-parse HEAD)" >> backup-data/backup-info.md
        echo "- **Rama**: $(git branch --show-current)" >> backup-data/backup-info.md
        echo "- **Último commit**: $(git log -1 --pretty=format:'%h - %ar : %s')" >> backup-data/backup-info.md
        echo "" >> backup-data/backup-info.md
        
        # Estadísticas del repositorio
        echo "## 📈 Estadísticas" >> backup-data/backup-info.md
        echo "- **Total de archivos**: $(find . -type f -not -path './.git/*' -not -path './backup-data/*' | wc -l)" >> backup-data/backup-info.md
        echo "- **Archivos HTML**: $(find . -name '*.html' -not -path './.git/*' | wc -l)" >> backup-data/backup-info.md
        echo "- **Archivos CSS**: $(find . -name '*.css' -not -path './.git/*' | wc -l)" >> backup-data/backup-info.md
        echo "- **Archivos JS**: $(find . -name '*.js' -not -path './.git/*' | wc -l)" >> backup-data/backup-info.md
        echo "- **Imágenes**: $(find . \( -name '*.jpg' -o -name '*.png' -o -name '*.gif' -o -name '*.svg' \) -not -path './.git/*' | wc -l)" >> backup-data/backup-info.md
        
        # Lista de archivos
        echo "## 📁 Archivos incluidos" >> backup-data/backup-info.md
        echo "\`\`\`" >> backup-data/backup-info.md
        find . -type f -not -path './.git/*' -not -path './backup-data/*' | sort >> backup-data/backup-info.md
        echo "\`\`\`" >> backup-data/backup-info.md
        
        # Crear archivo comprimido del código fuente
        tar -czf backup-data/source-code-${{ steps.date.outputs.date }}.tar.gz \
          --exclude='.git' \
          --exclude='backup-data' \
          --exclude='node_modules' \
          --exclude='.github' \
          .
          
        # Crear backup del historial git
        git bundle create backup-data/git-history-${{ steps.date.outputs.date }}.bundle HEAD
        
        # Obtener tamaño para la notificación
        BACKUP_SIZE=$(du -h backup-data/source-code-${{ steps.date.outputs.date }}.tar.gz | cut -f1)
        echo "backup_size=$BACKUP_SIZE" >> $GITHUB_OUTPUT
        
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
        
    - name: Install Google Drive dependencies
      run: |
        pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib
        
    - name: Create Google Drive upload script
      run: |
        cat > upload_to_gdrive.py << 'EOF'
        import os
        import json
        from google.oauth2.credentials import Credentials
        from googleapiclient.discovery import build
        from googleapiclient.http import MediaFileUpload
        from google.auth.transport.requests import Request
        
        def upload_to_drive():
            # Configurar credenciales desde secrets
            creds_info = json.loads(os.environ['GOOGLE_CREDENTIALS'])
            creds = Credentials.from_authorized_user_info(creds_info)
            
            # Si las credenciales están caducadas, refrescarlas
            if creds.expired and creds.refresh_token:
                creds.refresh(Request())
        
            # Construir el servicio de Drive
            service = build('drive', 'v3', credentials=creds)
            
            # Obtener variables de entorno
            date = os.environ['BACKUP_DATE']
            year = os.environ['BACKUP_YEAR']
            month = os.environ['BACKUP_MONTH']
            repo_name = os.environ['GITHUB_REPOSITORY'].split('/')[-1]
            
            # Buscar o crear carpeta principal "GitHub Backups"
            main_folder_name = 'GitHub Backups'
            main_folder_id = find_or_create_folder(service, main_folder_name)
            
            # Buscar o crear carpeta del repositorio
            repo_folder_id = find_or_create_folder(service, repo_name, main_folder_id)
            
            # Buscar o crear carpeta del año
            year_folder_id = find_or_create_folder(service, year, repo_folder_id)
            
            # Buscar o crear carpeta del mes
            month_folder_id = find_or_create_folder(service, month, year_folder_id)
            
            uploaded_files = []
            
            # Subir todos los archivos de backup
            for filename in os.listdir('backup-data'):
                filepath = os.path.join('backup-data', filename)
                
                if os.path.isfile(filepath):
                    # Crear nombre con fecha para evitar sobrescribir
                    drive_filename = f"{date}_{filename}"
                    
                    # Detectar tipo MIME
                    if filename.endswith('.tar.gz'):
                        mime_type = 'application/gzip'
                    elif filename.endswith('.bundle'):
                        mime_type = 'application/octet-stream'
                    elif filename.endswith('.md'):
                        mime_type = 'text/markdown'
                    else:
                        mime_type = 'application/octet-stream'
                    
                    # Subir archivo
                    media = MediaFileUpload(filepath, mimetype=mime_type, resumable=True)
                    
                    file_metadata = {
                        'name': drive_filename,
                        'parents': [month_folder_id],
                        'description': f'Backup automático de {repo_name} - {date}'
                    }
                    
                    print(f"Subiendo {filename}...")
                    file = service.files().create(
                        body=file_metadata,
                        media_body=media,
                        fields='id,name,size,webViewLink'
                    ).execute()
                    
                    uploaded_files.append({
                        'name': file.get('name'),
                        'id': file.get('id'),
                        'size': file.get('size'),
                        'link': file.get('webViewLink')
                    })
                    
                    print(f"✅ {filename} subido con ID: {file.get('id')}")
            
            # Limpiar backups antiguos (mantener últimos 10)
            cleanup_old_backups(service, month_folder_id, 10)
            
            # Guardar información para la notificación
            with open('upload_results.json', 'w') as f:
                json.dump(uploaded_files, f)
            
            print(f"✅ Backup completado. {len(uploaded_files)} archivos subidos.")
            return uploaded_files
        
        def find_or_create_folder(service, folder_name, parent_id='root'):
            """Busca una carpeta o la crea si no existe"""
            # Buscar carpeta existente
            query = f"name='{folder_name}' and mimeType='application/vnd.google-apps.folder' and '{parent_id}' in parents and trashed=false"
            results = service.files().list(q=query, fields="files(id, name)").execute()
            folders = results.get('files', [])
            
            if folders:
                return folders[0]['id']
            else:
                # Crear nueva carpeta
                folder_metadata = {
                    'name': folder_name,
                    'mimeType': 'application/vnd.google-apps.folder',
                    'parents': [parent_id]
                }
                folder = service.files().create(body=folder_metadata, fields='id').execute()
                print(f"📁 Carpeta creada: {folder_name}")
                return folder.get('id')
        
        def cleanup_old_backups(service, folder_id, keep_count):
            """Elimina backups antiguos, manteniendo solo los más recientes"""
            query = f"'{folder_id}' in parents and trashed=false"
            results = service.files().list(
                q=query,
                orderBy='createdTime desc',
                fields="files(id, name, createdTime)"
            ).execute()
            
            files = results.get('files', [])
            
            # Eliminar archivos antiguos (mantener solo keep_count más recientes)
            if len(files) > keep_count:
                files_to_delete = files[keep_count:]
                for file in files_to_delete:
                    try:
                        service.files().delete(fileId=file['id']).execute()
                        print(f"🗑️ Eliminado backup antiguo: {file['name']}")
                    except Exception as e:
                        print(f"⚠️ Error eliminando {file['name']}: {e}")
        
        if __name__ == "__main__":
            upload_to_drive()
        EOF
        
    - name: Upload to Google Drive
      env:
        GOOGLE_CREDENTIALS: ${{ secrets.GOOGLE_CREDENTIALS }}
        BACKUP_DATE: ${{ steps.date.outputs.date }}
        BACKUP_YEAR: ${{ steps.date.outputs.year }}
        BACKUP_MONTH: ${{ steps.date.outputs.month }}
        GITHUB_REPOSITORY: ${{ github.repository }}
      run: |
        python upload_to_gdrive.py
        
    - name: Prepare notification data
      id: notification_data
      run: |
        if [ -f upload_results.json ]; then
          UPLOAD_COUNT=$(jq length upload_results.json)
          MAIN_FILE=$(jq -r '.[0].name' upload_results.json)
          MAIN_LINK=$(jq -r '.[0].link' upload_results.json)
          
          echo "upload_count=$UPLOAD_COUNT" >> $GITHUB_OUTPUT
          echo "main_file=$MAIN_FILE" >> $GITHUB_OUTPUT
          echo "main_link=$MAIN_LINK" >> $GITHUB_OUTPUT
        else
          echo "upload_count=0" >> $GITHUB_OUTPUT
          echo "main_file=Error" >> $GITHUB_OUTPUT
          echo "main_link=" >> $GITHUB_OUTPUT
        fi
        
    # --- Notificación de éxito a Discord ---
    - name: Notificación de éxito en Discord
      if: success()
      uses: tsickert/discord-webhook@v5.3.0
      with:
        webhook-url: ${{ secrets.DISCORD_WEBHOOK_URL }}
        content: |
          ✅ ¡Backup a Google Drive completado con éxito!
          
          **📊 Detalles del backup:**
          - **Repositorio**: `${{ github.repository }}`
          - **Fecha**: `${{ steps.date.outputs.short_date }}`
          - **Tamaño**: `${{ steps.create_backup.outputs.backup_size }}`
          - **Archivos subidos**: `${{ steps.notification_data.outputs.upload_count }}`
          
          **📁 Ubicación en Google Drive:**
          > `GitHub Backups/${{ github.event.repository.name }}/${{ steps.date.outputs.year }}/${{ steps.date.outputs.month }}/`
          
          **🔗 Archivo principal:**
          > ${{ steps.notification_data.outputs.main_link }}
          
          *Los backups se mantienen automáticamente (últimos 10)*
        
    # --- Notificación de fallo a Discord ---
    - name: Notificación de fallo en Discord
      if: failure()
      uses: tsickert/discord-webhook@v5.3.0
      with:
        webhook-url: ${{ secrets.DISCORD_WEBHOOK_URL }}
        content: |
          ❌ **¡Error en backup a Google Drive!**
          
          El backup del repositorio **${{ github.repository }}** ha fallado.
          
          🔍 **Revisar logs**: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          
          *Posibles causas: credenciales expiradas, cuota de Drive excedida, o problemas de red.*

## 🔧 **Configuración necesaria**

### **1. Crear credenciales de Google Drive:**

1. Ve a [Google Cloud Console](https://console.cloud.google.com/)
2. Crea un nuevo proyecto o selecciona uno existente
3. Habilita la **Google Drive API**
4. Ve a "Credenciales" → "Crear credenciales" → "ID de cliente OAuth 2.0"
5. Configura como "Aplicación de escritorio"

### **2. Obtener token de autorización:**

Ejecuta este script Python en tu computadora para obtener el token:

```python
# get_google_token.py
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
import json

SCOPES = ['https://www.googleapis.com/auth/drive.file']

def main():
    creds = None
    
    # El archivo credentials.json lo descargas de Google Cloud Console
    flow = InstalledAppFlow.from_client_secrets_file(
        'credentials.json', SCOPES)
    creds = flow.run_local_server(port=0)
    
    # Guardar credenciales
    creds_data = {
        'token': creds.token,
        'refresh_token': creds.refresh_token,
        'token_uri': creds.token_uri,
        'client_id': creds.client_id,
        'client_secret': creds.client_secret,
        'scopes': creds.scopes
    }
    
    print("Copia este JSON en tu GitHub Secret GOOGLE_CREDENTIALS:")
    print(json.dumps(creds_data, indent=2))

if __name__ == '__main__':
    main()
```

### **3. Configurar GitHub Secrets:**

En tu repositorio de GitHub, ve a **Settings** → **Secrets and variables** → **Actions** y añade:

- `GOOGLE_CREDENTIALS`: El JSON completo que te dio el script anterior
- `DISCORD_WEBHOOK_URL`: Tu webhook de Discord (opcional)

## 🌟 **Ventajas del backup a Google Drive:**

✅ **15 GB gratis** (vs límites de GitHub)  
✅ **Acceso desde cualquier lugar** con interfaz web  
✅ **Versionado automático** (Google Drive mantiene historial)  
✅ **Fácil descarga** con un clic  
✅ **Organización automática** por carpetas (año/mes)  
✅ **Limpieza automática** (mantiene últimos 10 backups)  

## 📁 **Estructura resultante en Google Drive:**

```
Google Drive/
└── GitHub Backups/
    └── chakielzero.github.io/
        └── 2025/
            └── 01-January/
                ├── 2025-01-15_14-30-22_source-code-2025-01-15_14-30-22.tar.gz
                ├── 2025-01-15_14-30-22_git-history-2025-01-15_14-30-22.bundle
                └── 2025-01-15_14-30-22_backup-info.md
```
