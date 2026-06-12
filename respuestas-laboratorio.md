# Reporte y Respuestas — Laboratorio de Observabilidad

Este documento consolida las respuestas a las preguntas del cuestionario de la guía, detalla los cambios de compatibilidad aplicados para el entorno de desarrollo y provee las instrucciones necesarias para validar toda la solución implementada.

---

## 1. Respuestas al Cuestionario (Sección 13)

### Pregunta 1: ¿Por qué necesitamos Loki además de Prometheus si ya tenemos `/metrics`?
**Respuesta:** 
Prometheus y Loki cubren dos pilares complementarios pero distintos de la observabilidad:
* **Prometheus** está diseñado para recopilar y consultar series de tiempo numéricas (métricas). Nos indica el rendimiento cuantitativo del sistema (ej. tasas de peticiones, consumo de CPU) y permite disparar alarmas ante anomalías matemáticas.
* **Loki** se encarga de recolectar logs (datos de texto estructurados o planos). Cuando Prometheus nos indica un fallo numérico (por ejemplo, un pico de latencia), Loki nos permite ver el contenido real de los logs de la aplicación en ese mismo segundo para descifrar la causa raíz del error (ej. excepciones no controladas o fallos de conexión).

### Pregunta 2: ¿Qué ventaja aporta que las fuentes de datos de Grafana estén aprovisionadas como código y no creadas a mano?
**Respuesta:** 
El aprovisionamiento de datasources como código (IaC) en `datasources.yml` ofrece las siguientes ventajas:
* **Automatización y repetibilidad:** Los datasources están disponibles de inmediato al levantar el stack con `docker compose up`, sin necesidad de clics manuales en la interfaz.
* **Portabilidad y consistencia:** Elimina la posibilidad de errores de configuración humana (como apuntar a URLs o puertos incorrectos) entre diferentes desarrolladores que despliegan el laboratorio.
* **Control de versiones:** Las conexiones e integraciones de observabilidad quedan documentadas en Git, facilitando la auditoría de cambios en la infraestructura.

### Pregunta 3: El panel "CPU contenedor" y el panel "CPU host" pueden mostrar valores muy distintos. ¿Por qué? ¿Cuál usarías para alertar sobre una aplicación concreta?
**Respuesta (basada en la experiencia de Docker Desktop):**
Muestran valores distintos porque la CPU del host representa el consumo agregado de todo el hardware real (o la máquina virtual de WSL2 en Windows), mientras que la CPU del contenedor mide exclusivamente el consumo del proceso del backend. 

En entornos con Docker Desktop para Windows, las métricas de contenedor individual pueden presentar limitaciones por la arquitectura de cgroups de WSL2, por lo que es recomendable complementar con métricas de instrumentación directa a nivel de proceso.

**Cuál usar:** Para alertar sobre una aplicación concreta se debe usar siempre la CPU del contenedor o del proceso específico. La CPU del host contiene ruido de fondo de otras aplicaciones (como navegadores o el sistema operativo) y podría no alertar cuando nuestro servicio backend está saturado pero el host global sigue libre.

### Pregunta 4: ¿Qué diferencia hay entre el *evaluation interval* y el *pending period* de una alarma?
**Respuesta:** 
* **Evaluation interval (Intervalo de evaluación):** Define la frecuencia con la que Grafana ejecuta la consulta PromQL para revisar el umbral de la alarma (fijado a `10s` en nuestro laboratorio).
* **Pending period (Período pendiente):** Es el tiempo de gracia (fijado a `30s`) que la condición de alerta debe mantenerse en estado crítico de forma ininterrumpida antes de pasar a `Firing` y enviar notificaciones. Esto previene alertas innecesarias debido a picos de CPU cortos y normales durante el inicio o cargas puntuales del proceso.

---

## 2. Cambios Realizados al Repositorio (Fase 0)

Para asegurar la ejecución fluida del stack completo sobre **Windows con Docker Desktop** sin requerir WSL de forma directa, se realizaron las siguientes correcciones de infraestructura:

1. **Compatibilidad de `node-exporter`:**
   * **Cambio:** Se eliminó la configuración de namespaces de red compartidos (`pid: host`), la directiva de filesystem raíz (`--path.rootfs=/host`) y el montaje del volumen local `/:/host`.
   * **Justificación:** Windows no soporta el montaje del sistema de archivos raíz de Linux `/` ni el uso compartido del PID del host, lo cual impedía que el contenedor de métricas de hardware pudiera iniciarse.
2. **Compatibilidad de `cAdvisor`:**
   * **Cambio:** Se removió el montaje de disco `/dev/disk/:/dev/disk:ro` y el mapeo de dispositivos `devices: - /dev/kmsg`.
   * **Justificación:** Estas rutas y accesos de dispositivos específicos del kernel Linux no existen ni son mapeables directamente en la arquitectura de Docker Desktop sobre Windows.

---

## 3. Nota de Diseño sobre el Umbral de Alarma (80% vs 50%)

* **Decisión de diseño:** Se estableció el umbral de alerta en **80%** en lugar del 50% recomendado originalmente en la guía.
* **Justificación:** En entornos locales que ejecutan Docker Desktop dentro de máquinas virtuales de desarrollo, son frecuentes los picos de CPU aleatorios debido a la virtualización de discos y tareas secundarias del sistema operativo. Subir el umbral al **80%** previene falsos positivos en el entorno Windows local del estudiante, permitiendo que la alarma permanezca limpia en estado `Normal` y que la transición a `Firing` sea clara y controlada únicamente cuando se ejecuta la carga intensiva programada.

---

## Cómo validar el laboratorio

1. Clonar el repo y levantar el stack:
```bash
git clone 
cd iac-observabilidad
docker compose up -d --build
```

2. Verificar que todos los servicios estén arriba:
```bash
docker compose ps
```

3. Confirmar en el navegador:
   - Frontend: http://localhost:8080
   - Métricas del backend: http://localhost:3001/metrics
   - Prometheus targets (todos en UP): http://localhost:9090/targets
   - Grafana: http://localhost:3000 (admin/admin)

4. En Grafana, ir a Connections → Data sources y hacer Save & test en Prometheus y Loki. Ambos deben responder verde.

5. Abrir el dashboard **Observabilidad — Lab** y verificar los 4 paneles con datos.

6. Para probar la alarma y el ciclo completo:
```bash
curl "http://localhost:3001/load?seconds=60"
```
En Alerting → Alert rules la regla `CPU backend > 80%` debe pasar por Normal → Pending → Firing. Una vez en Firing, el panel de logs de aplicación debe mostrar:
`ERROR [backend] grafana_alert_received alert="CPU backend > 80%" status=firing count=1`
