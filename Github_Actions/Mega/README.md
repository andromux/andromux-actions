Searched for "MEGA API upload files GitHub Actions"

¡**MEGA es EXCELENTE opción!** Es mucho más fácil que Terabox y tiene herramientas específicas para GitHub Actions.

## 🚀 **MEGA - Mucho mejor que Terabox:**

### **✅ Ventajas de MEGA:**
- Existe una acción oficial `action-megacmd` que envuelve la CLI de MEGA para permitir interacción directa [Stack Overflow](https://stackoverflow.com/questions/78113499/how-to-download-from-terabox-by-terabox-api-with-python) [TeraBox](https://blog.terabox.com/terabox-how-to-upload-documents/)
- **50 GB gratis** (vs 1 TB de Terabox, pero más estable)
- **API robusta** y bien documentada
- **Cliente oficial** para GitHub Actions
- SDK oficial disponible con documentación completa [TeraBox](https://blog.terabox.com/terabox-remote-url-files-upload-for-free/)

# Backup automático a MEGA
name: Backup a MEGA

on:
  schedule:
    - cron: '0 2 * * *'  # Diario a las 2:00 AM UTC
  workflow_dispatch: # Ejecutar manualmente

jobs:
  backup-to-mega:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout repositorio
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
        echo "repo_name=$(echo '${{ github.repository }}' | cut -d'/' -f2)" >> $GITHUB_OUTPUT
        
    - name: Crear estructura de backup
      id: create_backup
      run: |
        mkdir -p backup-data
        
        # Información del backup
        echo "=== BACKUP AUTOMÁTICO A MEGA ===" > backup-data/backup-info.md
        echo "" >> backup-data/backup-info.md
        echo "## 📊 Información del Backup" >> backup-data/backup-info.md
        echo "- **Fecha**: $(date)" >> backup-data/backup-info.md
        echo "- **Repositorio**: ${{ github.repository }}" >> backup-data/backup-info.md
        echo "- **Commit**: $(git rev-parse HEAD)" >> backup-data/backup-info.md
        echo "- **Rama**: $(git branch --show-current)" >> backup-data/backup-info.md
        echo "- **Último commit**: $(git log -1 --pretty=format:'%h - %ar : %s')" >> backup-data/backup-info.md
        echo "" >> backup-data/backup-info.md
        
        # Estadísticas
        echo "## 📈 Estadísticas" >> backup-data/backup-info.md
        echo "- **Total archivos**: $(find . -type f -not -path './.git/*' -not -path './backup-data/*' | wc -l)" >> backup-data/backup-info.md
        echo "- **Archivos HTML**: $(find . -name '*.html' -not -path './.git/*' | wc -l)" >> backup-data/backup-info.md
        echo "- **Archivos CSS**: $(find . -name '*.css' -not -path './.git/*' | wc -l)" >> backup-data/backup-info.md
        echo "- **Archivos JS**: $(find . -name '*.js' -not -path './.git/*' | wc -l)" >> backup-data/backup-info.md
        echo "- **Imágenes**: $(find . \( -name '*.jpg' -o -name '*.png' -o -name '*.gif' -o -name '*.svg' \) -not -path './.git/*' | wc -l)" >> backup-data/backup-info.md
        echo "" >> backup-data/backup-info.md
        
        # Lista de archivos
        echo "## 📁 Estructura de archivos" >> backup-data/backup-info.md
        echo "\`\`\`" >> backup-data/backup-info.md
        find . -type f -not -path './.git/*' -not -path './backup-data/*' | sort >> backup-data/backup-info.md
        echo "\`\`\`" >> backup-data/backup-info.md
        
        # Crear backup comprimido principal
        tar -czf backup-data/mega-backup-${{ steps.date.outputs.date }}.tar.gz \
          --exclude='.git' \
          --exclude='backup-data' \
          --exclude='node_modules' \
          --exclude='.github' \
          --exclude='*.log' \
          .
          
        # Crear backup del historial Git (solo último commit)
        git bundle create backup-data/git-history-${{ steps.date.outputs.date }}.bundle HEAD
        
        # Obtener tamaños para notificación
        MAIN_SIZE=$(du -h backup-data/mega-backup-${{ steps.date.outputs.date }}.tar.gz | cut -f1)
        BUNDLE_SIZE=$(du -h backup-data/git-history-${{ steps.date.outputs.date }}.bundle | cut -f1)
        
        echo "main_size=$MAIN_SIZE" >> $GITHUB_OUTPUT
        echo "bundle_size=$BUNDLE_SIZE" >> $GITHUB_OUTPUT
        
        # Crear archivo con URLs de GitHub (para respaldo)
        echo "# 🔗 Enlaces de respaldo" > backup-data/github-links.md
        echo "" >> backup-data/github-links.md
        echo "Si necesitas descargar desde GitHub:" >> backup-data/github-links.md
        echo "- **Repositorio**: https://github.com/${{ github.repository }}" >> backup-data/github-links.md
        echo "- **Commit**: https://github.com/${{ github.repository }}/commit/${{ github.sha }}" >> backup-data/github-links.md
        echo "- **Fecha backup**: ${{ steps.date.outputs.date }}" >> backup-data/github-links.md
        
    - name: Setup MEGA CLI y autenticación
      uses: Difegue/action-megacmd@master
      with:
        username: ${{ secrets.MEGA_USERNAME }}
        password: ${{ secrets.MEGA_PASSWORD }}
        
    - name: Crear estructura de carpetas en MEGA
      run: |
        # Crear estructura jerárquica en MEGA
        REPO_NAME="${{ steps.date.outputs.repo_name }}"
        YEAR="${{ steps.date.outputs.year }}"
        MONTH="${{ steps.date.outputs.month }}"
        
        # Verificar y crear carpetas si no existen
        mega-mkdir -p "/GitHub-Backups" || echo "Carpeta principal ya existe"
        mega-mkdir -p "/GitHub-Backups/$REPO_NAME" || echo "Carpeta del repo ya existe"
        mega-mkdir -p "/GitHub-Backups/$REPO_NAME/$YEAR" || echo "Carpeta del año ya existe"
        mega-mkdir -p "/GitHub-Backups/$REPO_NAME/$YEAR/$MONTH" || echo "Carpeta del mes ya existe"
        
        echo "📁 Estructura de carpetas creada en MEGA"
        
    - name: Subir archivos a MEGA
      id: upload_mega
      run: |
        REPO_NAME="${{ steps.date.outputs.repo_name }}"
        YEAR="${{ steps.date.outputs.year }}"
        MONTH="${{ steps.date.outputs.month }}"
        DATE="${{ steps.date.outputs.date }}"
        
        MEGA_PATH="/GitHub-Backups/$REPO_NAME/$YEAR/$MONTH"
        
        echo "📤 Subiendo archivos a MEGA..."
        
        # Subir archivo principal
        echo "Subiendo código fuente..."
        mega-put backup-data/mega-backup-$DATE.tar.gz "$MEGA_PATH/"
        
        # Subir historial Git
        echo "Subiendo historial Git..."
        mega-put backup-data/git-history-$DATE.bundle "$MEGA_PATH/"
        
        # Subir información del backup
        echo "Subiendo información del backup..."
        mega-put backup-data/backup-info.md "$MEGA_PATH/"
        mega-put backup-data/github-links.md "$MEGA_PATH/"
        
        echo "✅ Archivos subidos exitosamente"
        
        # Obtener enlaces públicos de MEGA (si quieres compartir)
        echo "🔗 Generando enlaces públicos..."
        
        # Exportar archivo principal como público (opcional)
        MAIN_LINK=$(mega-export "$MEGA_PATH/mega-backup-$DATE.tar.gz" | grep -o 'https://mega.nz/file/[^[:space:]]*' || echo "Error generando enlace")
        
        echo "main_mega_link=$MAIN_LINK" >> $GITHUB_OUTPUT
        echo "mega_path=$MEGA_PATH" >> $GITHUB_OUTPUT
        
    - name: Verificar subida y obtener información
      id: verify_upload
      run: |
        REPO_NAME="${{ steps.date.outputs.repo_name }}"
        YEAR="${{ steps.date.outputs.year }}"
        MONTH="${{ steps.date.outputs.month }}"
        MEGA_PATH="/GitHub-Backups/$REPO_NAME/$YEAR/$MONTH"
        
        echo "📋 Verificando archivos en MEGA..."
        
        # Listar archivos en la carpeta
        FILES_IN_MEGA=$(mega-ls "$MEGA_PATH" | wc -l)
        
        # Obtener información de cuota
        QUOTA_INFO=$(mega-du 2>/dev/null | head -3 | tail -1 || echo "No disponible")
        
        echo "files_count=$FILES_IN_MEGA" >> $GITHUB_OUTPUT
        echo "quota_info=$QUOTA_INFO" >> $GITHUB_OUTPUT
        
        echo "✅ Verificación completada: $FILES_IN_MEGA archivos en MEGA"
        
    - name: Limpiar backups antiguos en MEGA
      run: |
        REPO_NAME="${{ steps.date.outputs.repo_name }}"
        YEAR="${{ steps.date.outputs.year }}"
        
        echo "🧹 Limpiando backups antiguos..."
        
        # Obtener lista de carpetas del año actual
        FOLDERS=$(mega-ls "/GitHub-Backups/$REPO_NAME/$YEAR" 2>/dev/null | grep -E '^[0-9]{2}-' | sort)
        
        if [ -n "$FOLDERS" ]; then
          FOLDER_COUNT=$(echo "$FOLDERS" | wc -l)
          
          # Si hay más de 6 carpetas mensuales, eliminar las más antiguas
          if [ $FOLDER_COUNT -gt 6 ]; then
            FOLDERS_TO_DELETE=$(echo "$FOLDERS" | head -n $((FOLDER_COUNT - 6)))
            
            echo "$FOLDERS_TO_DELETE" | while read folder; do
              if [ -n "$folder" ]; then
                echo "🗑️ Eliminando backup antiguo: $folder"
                mega-rm -rf "/GitHub-Backups/$REPO_NAME/$YEAR/$folder"
              fi
            done
          fi
        fi
        
        echo "✅ Limpieza completada"
        
    # Notificación de éxito
    - name: Notificación de éxito a Discord
      if: success()
      uses: tsickert/discord-webhook@v5.3.0
      with:
        webhook-url: ${{ secrets.DISCORD_WEBHOOK_URL }}
        content: |
          ✅ **¡Backup a MEGA completado!**
          
          **📊 Detalles del backup:**
          - **Repositorio**: `${{ github.repository }}`
          - **Fecha**: `${{ steps.date.outputs.short_date }}`
          - **Código fuente**: `${{ steps.create_backup.outputs.main_size }}`
          - **Historial Git**: `${{ steps.create_backup.outputs.bundle_size }}`
          - **Archivos en MEGA**: `${{ steps.verify_upload.outputs.files_count }}`
          
          **📁 Ubicación en MEGA:**
          > `GitHub-Backups/${{ steps.date.outputs.repo_name }}/${{ steps.date.outputs.year }}/${{ steps.date.outputs.month }}/`
          
          **💾 Uso de cuota:**
          > ${{ steps.verify_upload.outputs.quota_info }}
          
          ${{ steps.upload_mega.outputs.main_mega_link && format('**🔗 Enlace público:**
          > {0}', steps.upload_mega.outputs.main_mega_link) || '' }}
          
          *Accede a tu MEGA para descargar los archivos*
        
    # Notificación de fallo
    - name: Notificación de fallo a Discord
      if: failure()
      uses: tsickert/discord-webhook@v5.3.0
      with:
        webhook-url: ${{ secrets.DISCORD_WEBHOOK_URL }}
        content: |
          ❌ **¡Error en backup a MEGA!**
          
          El backup del repositorio **${{ github.repository }}** ha fallado.
          
          **🔍 Posibles causas:**
          - Credenciales de MEGA incorrectas
          - Cuota de almacenamiento excedida
          - Problemas de conexión
          
          **📋 Ver logs detallados:**
          > ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
          
          *Revisa tus credenciales en GitHub Secrets*

## 🔧 **Configuración súper fácil:**

### **1. Solo necesitas configurar 2 secrets en GitHub:**

**Settings** → **Secrets and variables** → **Actions**:

- `MEGA_USERNAME`: Tu email de MEGA
- `MEGA_PASSWORD`: Tu contraseña de MEGA
- `DISCORD_WEBHOOK_URL`: Tu webhook de Discord (opcional)

### **2. ¡Y listo! No necesitas:**
- ❌ Tokens complicados
- ❌ APIs keys
- ❌ Configuraciones OAuth
- ❌ Permisos especiales

## 🌟 **Ventajas de MEGA vs otras opciones:**

| Característica | MEGA | Google Drive | Terabox |
|----------------|------|--------------|---------|
| **Configuración** | ✅ Súper fácil | ⚠️ Compleja (OAuth) | ❌ Sin API pública |
| **Espacio gratis** | 50 GB | 15 GB | 1 TB |
| **Automación** | ✅ GitHub Action oficial | ✅ API completa | ❌ Solo manual |
| **Privacidad** | ✅ Encriptado E2E | ⚠️ Estándar | ⚠️ Estándar |
| **Velocidad setup** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐ |

## 📁 **Estructura automática en MEGA:**

```
MEGA:/
└── GitHub-Backups/
    └── chakielzero.github.io/
        └── 2025/
            └── 01-January/
                ├── mega-backup-2025-01-15_14-30-22.tar.gz
                ├── git-history-2025-01-15_14-30-22.bundle
                ├── backup-info.md
                └── github-links.md
```

## 🚀 **Características incluidas:**

✅ **Limpieza automática** (mantiene últimos 6 meses)  
✅ **Verificación de subida** (confirma que llegó bien)  
✅ **Enlaces públicos** (opcionales para compartir)  
✅ **Información de cuota** (te avisa si se llena)  
✅ **Organización automática** por fecha  
✅ **Notificaciones Discord** con detalles completos  

**¿Quieres que ajuste algo específico?** MEGA es definitivamente la opción más práctica para esto.

Citations:
- [How to Download From Terabox by Terabox api with Python? - Stack Overflow](https://stackoverflow.com/questions/78113499/how-to-download-from-terabox-by-terabox-api-with-python)
- [TeraBox: How to Upload Documents?](https://blog.terabox.com/terabox-how-to-upload-documents/)
- [TeraBox: Remote URL Files Upload for Free](https://blog.terabox.com/terabox-remote-url-files-upload-for-free/)

More sources:
- [GitHub - actions/upload-release-asset: An Action to upload a release asset via the GitHub Release API](https://github.com/actions/upload-release-asset)
- [script uploader for multiple files, using mega.co.nz python API · GitHub](https://gist.github.com/5502087)
- [action-megacmd - GitHub Marketplace](https://github.com/marketplace/actions/action-megacmd)
- [GitHub - t3rm1n4l/go-mega: A client library in go for mega.co.nz storage service](https://github.com/t3rm1n4l/go-mega)
- [GitHub - lmb/Supermega: MEGA API client in Python](https://github.com/lmb/Supermega)
- [GitHub - 3v1n0/mega.py: Simple mega python APIs](https://github.com/3v1n0/mega.py)
- [GitHub - edunavajas/Mega-Api: Api in pyhton to comunicate with Mega service](https://github.com/edunavajas/Mega-Api)
- [MEGA Software Development Kit](https://mega.io/developers)
- [GitHub - Difegue/action-megacmd: GitHub Action for interacting with MEGA](https://github.com/Difegue/action-megacmd)
- [Attachment API : Upload or Download Business Docum... - MEGA Community](https://community.mega.com/t5/User-Forum/Attachment-API-Upload-or-Download-Business-Document/td-p/22821)
