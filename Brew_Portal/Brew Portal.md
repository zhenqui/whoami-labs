---
tipo: writeup
plataforma: whoami-labs
maquina: Brew Portal
ip: 172.17.0.2
dificultad: Fácil
estado: finalizada
---
# Informe de Seguridad:
Brew Portal

## 1. Resumen Ejecutivo
**Puntuación de Riesgo:** Crítico
**Descripción del Impacto:**
Se ha identificado y explotado con éxito una vulnerabilidad de **Inclusión de Archivos Locales (LFI)** en el parámetro `page` del servidor web principal. Esta brecha permitió el acceso no autorizado a archivos sensibles del sistema operativo fuera de la raíz del servidor web (`/var/www/html`), culminando en la exfiltración de información confidencial (flag del sistema).

El impacto es crítico, ya que este vector de ataque no solo permite la lectura de archivos de configuración y credenciales (como `/etc/passwd` o archivos de entorno de Apache), sino que establece una base directa para ataques de **Log Poisoning**, lo que permitiría al atacante ejecutar comandos remotos (**RCE**) y obtener el control total del servidor.

## 2. Resumen Técnico (Matriz de Hallazgos)
| ID  | Vulnerabilidad     | Severidad | Estado       |
| --- | ------------------ | --------- | ------------ |
| 01  | LFI                | Crítica   | Explotado    |
| 02  | PACKETSTORM:176334 | Crítica   | No explotado |
| 03  | CVE-2023-25690     | Crítica   | No explotado |

---
## 3. Fase de Reconocimiento
### Enumeración de Puertos (Nmap)
```
sudo nmap -p- --open -sS -sC -sV --min-rate 5000 -vvvv -n -Pn 172.17.0.2
```
- **Puertos abiertos:** 80.
- **Evidencias:** > [!info] Resultado Nmap > ![[image20260419112255.png]]
### Enumeración Web
```
dirsearch -u http://172.17.0.2 -w /usr/share/wordlists/dirb/common.txt -e all
```

- **Tecnologías:** [[Apache HTTP Server]], [[PHP]].
	- ![[image20260419113121.png]]
- **Directorias hallados:** `/server-status.
	- ![[image20260419112746.png]]
	
---
## 4. Análisis y Explotación
### Vector de entrada: [[http://172.17.0.2/?page=../flag.txt]]
- Se buscan vulnerabilidades con nmap
```
sudo nmap $TARGET -p 80 -sV --script vuln
```
- El análisis muestra un servidor Apache 2.4.54 en Debian con múltiples vulnerabilidades críticas (CVSS 9.8), incluyendo RCE (CVE-2024-38476, CVE-2024-38474) y Request Smuggling (CVE-2023-25690). El vector de entrada principal es la explotación de estos fallos conocidos mediante exploits públicos listados en el reporte.
![[image20260419114933.png]]
- Se observa posible vulnerabilidad LFI en la ruta
	- ![[image20260419121409.png]]
	- http://172.17.0.2/?page=../../../../etc/passwd
	- ![[image20260419122022.png]]
	- http://172.17.0.2/?page=../../../../etc/apache2/apache2.conf
	- ![[image20260419123241.png]]
	
	- **Permiso en `/var/www/`**: Aquí es donde vive la web. El archivo dice que en esa carpeta **sí** se permite ver contenido (`Require all granted`).
	
- **Paylod utilizado:** [[LFI]]

### Post-Explotación
- Técnica utilizada [[LFI]] 
- http://172.17.0.2/?page=../flag.txt
- ![[image20260419123856.png]]


### Captura de Flag
`FLAG{lfi_to_rce_brew_master}`

---
## 5. Recomendaciones y Conclusiones
### Remediación Técnica
1. **Actualizar el servicio Apache:** Migrar a la última versión estable (2.4.62 o superior) para parchear las vulnerabilidades críticas de _Request Smuggling_ y ejecución de código detectadas en la versión 2.4.54.
2. **Sanitización de parámetros:** Configurar el código de la aplicación para que no acepte caracteres de control ni secuencias de ruta (`../`) en variables de navegación
3. **Endurecimiento del sistema (Hardening):** Restringir los permisos del usuario `www-data` para que no tenga capacidad de lectura sobre archivos en la raíz (`/`) o directorios sensibles fuera de `/var/www/html`.

### Conclusión Final
**Recomendación:** Implementar una lista blanca (allowlist) de archivos permitidos para el parámetro `page` y sanitizar cualquier entrada que contenga secuencias de salto de directorio (`../`).