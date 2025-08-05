# Manual de Uso: GitHub Actions Backup a MEGA

## 📋 Descripción

Esta GitHub Action automatiza el proceso de crear copias de seguridad de tu repositorio y subirlas a tu cuenta de MEGA. Comprime todo el contenido del repositorio en un archivo ZIP y lo almacena de forma segura en la nube.

## 🎯 Funcionalidades

- **Backup automático** del repositorio completo
- **Compresión ZIP** para optimizar el espacio de almacenamiento
- **Nomenclatura inteligente** con fecha y hora automática
- **Organización automática** en carpetas estructuradas en MEGA
- **Múltiples triggers** para diferentes escenarios de uso
- **Creación automática** de estructura de carpetas en MEGA

## ⚙️ Configuración Inicial

### 1. Configurar Credenciales de MEGA

1. Ve a tu repositorio en GitHub
2. Navega a **Settings** → **Secrets and variables** → **Actions**
3. Crea dos nuevos **Repository secrets**:

   - **Nombre**: `MEGA_USERNAME`
   - **Valor**: Tu email de cuenta MEGA

   - **Nombre**: `MEGA_PASSWORD`
   - **Valor**: Tu contraseña de MEGA

### 2. Instalar el Workflow

1. En tu repositorio, crea la estructura de carpetas: `.github/workflows/`
2. Crea un archivo llamado `backup-to-mega.yml`
3. Copia y pega el código del workflow
4. Haz commit y push de los cambios

## 🚀 Modos de Ejecución

### Ejecución Manual
- Ve a **Actions** en tu repositorio
- Selecciona **"Backup Repository to MEGA"**
- Haz clic en **"Run workflow"**
- Selecciona la rama y ejecuta

### Ejecución Automática

**Por Schedule (Programado):**
- Se ejecuta automáticamente cada domingo a las 2:00 AM UTC
- Puedes modificar la programación editando la línea `cron`

**Por Push:**
- Se ejecuta automáticamente cada vez que haces push a la rama `main`
- Útil para tener siempre la versión más reciente respaldada

## 📁 Estructura de Archivos en MEGA

### Organización Automática
```
MEGA/
└── backups/
    └── github/
        └── nombre-del-repositorio/
            ├── nombre-repo_backup_20250805_003231.zip
            ├── nombre-repo_backup_20250812_003215.zip
            └── nombre-repo_backup_20250819_003142.zip
```

### Nomenclatura de Archivos
- **Formato**: `nombre-repositorio_backup_YYYYMMDD_HHMMSS.zip`
- **Ejemplo**: `mi-proyecto_backup_20250805_143022.zip`
- **Información incluida**:
  - Nombre del repositorio
  - Fecha (Año-Mes-Día)
  - Hora (Hora-Minuto-Segundo)

## 📦 Contenido del Backup

### Incluido por Defecto
- Todos los archivos del repositorio
- Configuraciones y archivos ocultos (excepto .git)
- Estructura completa de carpetas
- README, licencias, documentación

### Excluido por Defecto
- Carpeta `.git` (historial de versiones)
- Archivos temporales del sistema

### Personalizar Contenido

**Para incluir el historial de Git:**
```yaml
# Cambiar esta línea:
zip -r ${{ steps.vars.outputs.zip_name }} . -x "*.git*"

# Por esta:
zip -r ${{ steps.vars.outputs.zip_name }} .
```

**Para excluir carpetas adicionales:**
```yaml
zip -r ${{ steps.vars.outputs.zip_name }} . -x "*.git*" "*node_modules*" "*dist*"
```

## 🔧 Personalización Avanzada

### Cambiar Frecuencia de Backup
```yaml
schedule:
  # Diario a las 3:00 AM
  - cron: '0 3 * * *'
  
  # Solo lunes a las 9:00 AM
  - cron: '0 9 * * 1'
  
  # Cada 6 horas
  - cron: '0 */6 * * *'
```

### Modificar Ruta de Destino
```yaml
# Estructura simple
args: put -c ${{ steps.vars.outputs.zip_name }} /backups/

# Por fecha
args: put -c ${{ steps.vars.outputs.zip_name }} /backups/$(date +%Y)/$(date +%m)/

# Por repositorio y año
args: put -c ${{ steps.vars.outputs.zip_name }} /backups/${{ steps.vars.outputs.repo_name }}/$(date +%Y)/
```

### Cambiar Triggers
```yaml
on:
  # Solo manual
  workflow_dispatch:
  
  # Solo en releases
  release:
    types: [published]
  
  # En pull requests a main
  pull_request:
    branches: [main]
```

## 📊 Monitoreo y Verificación

### Ver Estado de Backups
1. Ve a la pestaña **Actions** de tu repositorio
2. Busca **"Backup Repository to MEGA"**
3. Revisa el estado de las ejecuciones (✅ exitosa, ❌ fallida)

### Verificar en MEGA
1. Inicia sesión en tu cuenta MEGA
2. Navega a la carpeta `/backups/github/nombre-repositorio/`
3. Verifica que los archivos ZIP estén presentes y tengan el tamaño esperado

### Logs de Depuración
En caso de errores, revisa los logs detallados:
1. Haz clic en la ejecución fallida
2. Expande cada paso para ver los detalles
3. Busca mensajes de error específicos

## ⚠️ Consideraciones Importantes

### Límites de MEGA
- **Cuenta gratuita**: 15 GB de almacenamiento
- **Límites de transferencia**: Pueden aplicar según tu plan
- **Tamaño de archivo**: Sin límite específico para archivos individuales

### Seguridad
- Las credenciales se almacenan de forma segura como GitHub Secrets
- No se exponen en logs ni en el código del workflow
- Se transmiten encriptadas durante el proceso

### Rendimiento
- El tiempo de ejecución depende del tamaño del repositorio
- Repositorios grandes pueden tomar varios minutos
- La velocidad de subida depende de tu conexión y la de GitHub

## 🛠️ Solución de Problemas

### Error: "Couln't find destination folder"
**Solución**: Agregar la opción `-c` en los argumentos de MEGAcmd
```yaml
args: put -c archivo.zip /ruta/destino/
```

### Error: "Authentication failed"
**Causas posibles**:
- Credenciales incorrectas en los Secrets
- Cuenta MEGA suspendida o con problemas
- Límites de API alcanzados

**Solución**: Verificar credenciales y estado de la cuenta

### Backup muy grande
**Soluciones**:
- Excluir carpetas innecesarias (node_modules, dist, etc.)
- Usar compresión adicional
- Dividir en múltiples archivos

### Falla intermitente
**Causas**:
- Problemas de red temporales
- Límites de API de MEGA
- Sobrecarga del servidor

**Solución**: El workflow se puede re-ejecutar manualmente

## 📚 Comandos MEGAcmd Útiles

### Listar archivos
```yaml
args: ls /backups/github/mi-repo/
```

### Descargar backup
```yaml
args: get /backups/github/mi-repo/archivo.zip ./
```

### Eliminar backups antiguos
```yaml
args: rm /backups/github/mi-repo/archivo-viejo.zip
```

## 🔄 Mantenimiento

### Limpieza Periódica
Considera implementar una limpieza automática de backups antiguos para no saturar tu cuenta de MEGA:

```yaml
- name: Clean old backups (optional)
  uses: Difegue/action-megacmd@master
  with:
    args: find /backups/github/${{ steps.vars.outputs.repo_name }}/ -type f -mtime +30 -delete
  env:
    USERNAME: ${{ secrets.MEGA_USERNAME }}
    PASSWORD: ${{ secrets.MEGA_PASSWORD }}
```

### Actualización del Workflow
- Revisa periódicamente si hay nuevas versiones de las actions utilizadas
- Actualiza la referencia `@master` por versiones específicas para mayor estabilidad
- Mantén actualizadas las credenciales si cambias tu contraseña de MEGA

---

## 💡 Consejos Adicionales

1. **Prueba inicial**: Ejecuta manualmente el workflow antes de confiar en la automatización
2. **Documentación**: Mantén este manual actualizado si modificas el workflow
3. **Respaldo de respaldo**: Considera usar múltiples servicios de almacenamiento
4. **Monitoreo**: Configura notificaciones para fallos de backup críticos
5. **Versionado**: Usa tags de Git para marcar versiones importantes antes del backup

¡Tu sistema de backup automático a MEGA está listo para usar! 🎉
