# Análisis Funcional SmartTiming - Instituto Salmedina
## Análisis Post-Reunión con Jefe de Estudios - Centro Público Andaluz

---

## 📋 INFORMACIÓN DE CONTEXTO

**Fecha**: Análisis basado en reunión con Jefe de Estudios  
**Centro**: Instituto Público de Andalucía (Salmedina)  
**Participantes**: Jefe de Estudios, Equipo de Desarrollo SmartTiming  
**Objetivo**: Identificar problemática actual y requisitos para solución SaaS de gestión de horarios  

---

## 🎯 RESUMEN EJECUTIVO

El Instituto Salmedina enfrenta una problemática crítica en la gestión de horarios académicos, utilizando actualmente herramientas con **UX deficiente** que generan ineficiencias operativas significativas. La complejidad del proceso manual actual requiere **semanas de trabajo intensivo** para generar horarios que cumplan con la normativa andaluza y las necesidades pedagógicas del centro.

**Puntos Clave Identificados:**
- Proceso manual extremadamente complejo y propenso a errores
- Herramientas actuales con UX inadecuada para la complejidad del problema
- Necesidad de automatización inteligente con algoritmos de optimización
- Oportunidad de diferenciación en mercado de centros públicos andaluces

---

## 🔍 INSIGHTS PRINCIPALES

### Oportunidades Identificadas

| Oportunidad | Impacto Estimado | Prioridad |
|-------------|------------------|-----------|
| **Automatización completa del proceso** | Alto - Reducción 80% tiempo gestión | Crítica |
| **Integración con sistemas Junta Andalucía** | Alto - Eliminación duplicación datos | Alta |
| **Optimización algorítmica avanzada** | Medio-Alto - Mejora calidad horarios | Alta |
| **Interfaz intuitiva especializada** | Medio - Mejora experiencia usuario | Media |
| **Validación automática normativa** | Alto - Eliminación errores compliance | Alta |

### Problemas/Pain Points Identificados

| Problema | Severidad | Frecuencia | Impacto |
|----------|-----------|------------|---------|
| **Proceso manual extremadamente lento** | Crítica | Anual | Operativo |
| **Herramientas con UX deficiente** | Alta | Diaria | Productividad |
| **Conflictos y solapamientos frecuentes** | Alta | Semanal | Calidad |
| **Dificultad gestión restricciones complejas** | Alta | Anual | Operativo |
| **Falta integración con sistemas oficiales** | Media | Continua | Eficiencia |

---

## 📋 ANÁLISIS DE REQUISITOS

### Requisitos Funcionales Principales

#### RF001: Gestión Integral de Entidades Educativas
**Descripción**: El sistema debe gestionar todas las entidades del centro educativo
**Criterios de Aceptación**:
- Gestión completa de profesorado con cargas horarias
- Administración de grupos y subgrupos de alumnos
- Catálogo de asignaturas con sesiones semanales
- Gestión de aulas y recursos especializados

#### RF002: Motor de Optimización Algorítmica
**Descripción**: Algoritmo inteligente para generación automática de horarios
**Criterios de Aceptación**:
- Cumplimiento 100% restricciones duras (normativa)
- Optimización restricciones blandas (preferencias)
- Capacidad de manejar 30+ grupos simultáneamente
- Tiempo de procesamiento < 30 minutos para centro completo

#### RF003: Gestión Avanzada de Restricciones
**Descripción**: Sistema flexible para definir y aplicar restricciones complejas
**Criterios de Aceptación**:
- Restricciones duras (obligatorias) vs blandas (preferencias)
- Gestión de paralelismos y optativas cruzadas
- Restricciones por profesor, asignatura y aula
- Validación automática de conflictos

#### RF004: Interfaz de Usuario Especializada
**Descripción**: UX optimizada para gestores educativos
**Criterios de Aceptación**:
- Dashboard ejecutivo con métricas clave
- Vistas especializadas por rol (Jefe Estudios, Profesores)
- Edición visual drag-and-drop de horarios
- Alertas automáticas de conflictos

#### RF005: Integración con Sistemas Oficiales
**Descripción**: Conectividad con plataformas de la Junta de Andalucía
**Criterios de Aceptación**:
- Importación automática desde Séneca
- Exportación a formatos oficiales requeridos
- Sincronización bidireccional de datos
- Cumplimiento normativa protección datos

### Requisitos No Funcionales

#### RNF001: Rendimiento y Escalabilidad
- **Tiempo respuesta**: < 3 segundos para operaciones básicas
- **Capacidad**: Soporte hasta 3,000 alumnos por centro
- **Concurrencia**: 50+ usuarios simultáneos
- **Disponibilidad**: 99.5% uptime durante período lectivo

#### RNF002: Seguridad y Compliance
- **Protección datos**: Cumplimiento RGPD y normativa educativa
- **Autenticación**: Integración con sistemas corporativos Junta
- **Auditoría**: Trazabilidad completa de cambios
- **Backup**: Copias automáticas cada 4 horas

#### RNF003: Usabilidad y Accesibilidad
- **Responsive design**: Adaptación móvil y tablet
- **Accesibilidad**: Cumplimiento WCAG 2.1 AA
- **Multiidioma**: Soporte español e inglés
- **Curva aprendizaje**: < 2 horas formación básica

---

## 🗃️ ENTIDADES DE DATOS CRÍTICAS

### Entidades Principales (Input Obligatorio)

#### 1. **PROFESORADO**
```yaml
Profesor:
  - id_profesor: string (único)
  - nombre_completo: string
  - dni: string
  - departamento: string
  - especialidades: array[string]
  - tipo_contrato: enum[completo, parcial, interino]
  - horas_lectivas_asignadas: integer (18-21)
  - horas_complementarias: integer (7)
  - reducciones_especiales: object
    - jefe_departamento: boolean (3h reducción)
    - mayor_55: boolean (3h reducción)
    - tutor: boolean (1h tutoría)
    - otros_cargos: array[object]
  - disponibilidad_horaria: object
    - indisponibilidades_duras: array[slot_tiempo]
    - preferencias_medias: array[slot_tiempo]
    - preferencias_blandas: array[slot_tiempo]
  - guardias_asignadas: object
    - guardias_pasillo: integer
    - guardias_recreo: integer
    - guardias_convivencia: integer
```

#### 2. **ESTRUCTURA ACADÉMICA**
```yaml
Curso:
  - id_curso: string
  - nombre: string (ej: "3º ESO")
  - etapa_educativa: enum[ESO, BACHILLERATO, FP, ADULTOS]
  - grupos: array[Grupo]

Grupo:
  - id_grupo: string
  - nombre: string (ej: "3º ESO A")
  - numero_alumnos: integer
  - aula_referencia: string
  - programas_especiales: array[string] (diversificación, etc.)
  - tutor_asignado: id_profesor
```

#### 3. **ASIGNATURAS Y MATERIAS**
```yaml
Asignatura:
  - id_asignatura: string
  - codigo_oficial: string (Séneca)
  - nombre: string
  - departamento_responsable: string
  - etapa: enum[ESO, BACHILLERATO, FP]
  - curso: string
  - tipo: enum[troncal, específica, optativa, libre_configuración]
  - sesiones_semanales: integer
  - duracion_sesion: integer (minutos)
  - permite_sesiones_dobles: boolean
  - requiere_aula_especifica: boolean
  - aula_tipo_requerido: enum[general, laboratorio, taller, gimnasio]
  - preferencias_horarias: object
    - no_primera_hora: boolean
    - no_ultima_hora: boolean
    - no_post_recreo: boolean
    - no_pre_recreo: boolean
```

#### 4. **ASIGNACIÓN DOCENTE**
```yaml
AsignacionDocente:
  - id_asignacion: string
  - id_profesor: string
  - id_grupo: string
  - id_asignatura: string
  - sesiones_semanales: integer
  - aula_asignada: string
  - fecha_asignacion: date
  - estado: enum[activa, pendiente, cancelada]
```

#### 5. **INFRAESTRUCTURA Y RECURSOS**
```yaml
Aula:
  - id_aula: string
  - nombre: string
  - tipo: enum[general, laboratorio, taller, gimnasio, biblioteca]
  - capacidad_maxima: integer
  - equipamiento: array[string]
  - disponibilidad: object
    - indisponibilidades: array[slot_tiempo]
    - mantenimiento_programado: array[slot_tiempo]
  - permite_uso_compartido: boolean
  - edificio: string
  - planta: integer

RecursoEspecial:
  - id_recurso: string
  - nombre: string
  - tipo: enum[proyector, laboratorio_movil, equipo_musica]
  - disponibilidad: array[slot_tiempo]
  - requiere_reserva: boolean
```

#### 6. **ESTRUCTURA TEMPORAL**
```yaml
SlotTiempo:
  - dia_semana: enum[lunes, martes, miercoles, jueves, viernes]
  - periodo: integer (1-6)
  - hora_inicio: time
  - hora_fin: time
  - es_recreo: boolean

JornadaEscolar:
  - turno: enum[mañana, tarde]
  - slots_disponibles: array[SlotTiempo]
  - recreo_slot: SlotTiempo
  - duracion_sesion: integer (minutos)
```

#### 7. **RESTRICCIONES Y PARALELISMOS**
```yaml
BloqueParalelo:
  - id_bloque: string
  - nombre: string
  - tipo: enum[optativas_cruzadas, desdobles, religion_valores]
  - asignaturas_incluidas: array[object]
    - id_asignacion: string
    - sesiones_simultaneas: integer
  - restricciones_especiales: object
    - mismo_profesor_prohibido: boolean
    - misma_aula_prohibida: boolean

RestriccionHoraria:
  - id_restriccion: string
  - tipo: enum[dura, media, blanda]
  - entidad_afectada: enum[profesor, asignatura, grupo, aula]
  - id_entidad: string
  - slots_afectados: array[SlotTiempo]
  - descripcion: string
  - peso_penalizacion: float
```

#### 8. **GUARDIAS Y SERVICIOS**
```yaml
TipoGuardia:
  - id_tipo: string
  - nombre: string (ej: "Guardia Pasillo")
  - descripcion: string
  - minimo_profesores_requerido: integer
  - slots_aplicables: array[SlotTiempo]

AsignacionGuardia:
  - id_asignacion: string
  - id_profesor: string
  - tipo_guardia: string
  - slot_tiempo: SlotTiempo
  - es_fija: boolean
```

#### 9. **REUNIONES Y ACTIVIDADES**
```yaml
Reunion:
  - id_reunion: string
  - nombre: string
  - tipo: enum[tutores, departamento, claustro, evaluacion]
  - participantes_requeridos: array[id_profesor]
  - duracion: integer (minutos)
  - frecuencia: enum[semanal, quincenal, mensual, puntual]
  - preferencias_horarias: array[SlotTiempo]
  - es_obligatoria: boolean
```

### Entidades de Configuración y Parámetros

#### 10. **CONFIGURACIÓN DEL CENTRO**
```yaml
ConfiguracionCentro:
  - id_centro: string
  - codigo_centro: string (oficial Junta)
  - nombre_centro: string
  - tipo_centro: enum[IES, CEIP, CEPER, CIFP]
  - parametros_horarios: object
    - horas_lectivas_minimas: integer (18)
    - horas_lectivas_maximas: integer (21)
    - horas_complementarias: integer (7)
    - max_horas_diarias_profesor: integer (5)
    - min_horas_diarias_profesor: integer (2)
  - parametros_optimizacion: object
    - peso_huecos_profesor: float
    - peso_preferencias_duras: float
    - peso_preferencias_medias: float
    - peso_preferencias_blandas: float
    - max_tiempo_calculo: integer (minutos)
```

---

## 🎯 ANÁLISIS DE VALOR

### Impacto de Negocio Cuantificado

#### Beneficios Directos:
- **Reducción tiempo gestión**: 80% (de 40h a 8h por horario)
- **Eliminación errores**: 95% reducción conflictos horarios
- **Mejora satisfacción profesorado**: Optimización preferencias individuales
- **Cumplimiento normativo**: 100% adherencia regulaciones Junta Andalucía

#### Beneficios Indirectos:
- **Mejora clima laboral**: Horarios más equilibrados
- **Optimización recursos**: Mejor uso aulas y espacios
- **Reducción estrés directivo**: Automatización proceso crítico
- **Escalabilidad**: Capacidad crecimiento sin incremento recursos

### Esfuerzo Estimado

#### Desarrollo MVP (6 meses):
- **Equipo técnico**: 4 desarrolladores + 1 arquitecto
- **Equipo funcional**: 1 analista + 1 especialista educativo
- **Inversión estimada**: 180,000€ - 220,000€

#### Desarrollo Completo (12 meses):
- **Funcionalidades avanzadas**: IA predictiva, analytics
- **Integraciones completas**: Séneca, otros sistemas
- **Inversión total estimada**: 350,000€ - 450,000€

### Matriz de Priorización (Impacto vs Esfuerzo)

| Funcionalidad | Impacto | Esfuerzo | Prioridad |
|---------------|---------|----------|-----------|
| Motor optimización básico | Alto | Alto | P1 - Crítica |
| Gestión entidades core | Alto | Medio | P1 - Crítica |
| Interfaz básica UX | Medio | Medio | P2 - Alta |
| Integración Séneca | Alto | Alto | P2 - Alta |
| Analytics avanzados | Medio | Alto | P3 - Media |
| IA predictiva | Bajo | Alto | P4 - Baja |

---

## ⚡ ACCIONES INMEDIATAS

| Acción | Responsable | Fecha Límite | Prioridad |
|--------|-------------|--------------|-----------|
| **Validación técnica algoritmos optimización** | Arquitecto Técnico | 15 días | Crítica |
| **Análisis detallado integración Séneca** | Analista Sistemas | 20 días | Alta |
| **Prototipo UX/UI especializado** | Diseñador UX | 25 días | Alta |
| **Estudio viabilidad técnica IA** | Data Scientist | 30 días | Media |
| **Plan piloto con 3 centros** | Project Manager | 45 días | Alta |
| **Análisis competencia detallado** | Business Analyst | 15 días | Media |

---

## ⚠️ RIESGOS IDENTIFICADOS

### Riesgos Técnicos

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| **Complejidad algorítmica subestimada** | Media | Alto | Prototipo temprano + consultoría especializada |
| **Problemas integración Séneca** | Alta | Alto | Análisis técnico previo + plan B manual |
| **Rendimiento insuficiente** | Media | Medio | Arquitectura escalable + testing carga |
| **Dificultades UX complejidad dominio** | Alta | Medio | Co-diseño con usuarios finales |

### Riesgos de Negocio

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| **Resistencia cambio usuarios** | Alta | Alto | Plan formación + change management |
| **Competencia con soluciones existentes** | Media | Medio | Diferenciación sector público |
| **Cambios normativos Junta** | Media | Alto | Arquitectura flexible + monitoreo regulatorio |
| **Presupuestos limitados centros** | Alta | Alto | Modelo SaaS escalable + financiación |

### Riesgos Regulatorios

| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|--------------|---------|------------|
| **Incumplimiento RGPD** | Baja | Crítico | Auditoría legal + privacy by design |
| **No conformidad normativa educativa** | Media | Alto | Validación continua + asesoría legal |
| **Problemas certificación Junta** | Media | Alto | Proceso certificación temprano |

---

## 🚀 PRÓXIMOS PASOS

### Fase 1: Validación y Diseño (Mes 1-2)
1. **Validación técnica algoritmos** con expertos optimización
2. **Análisis profundo integración** sistemas Junta Andalucía
3. **Diseño arquitectura** escalable y modular
4. **Prototipo UX** con feedback usuarios reales
5. **Plan piloto** con 3 centros representativos

### Fase 2: Desarrollo MVP (Mes 3-6)
1. **Desarrollo motor optimización** básico
2. **Implementación gestión entidades** core
3. **Interfaz usuario** especializada
4. **Integración básica** con sistemas existentes
5. **Testing exhaustivo** con datos reales

### Fase 3: Piloto y Refinamiento (Mes 7-9)
1. **Despliegue piloto** en centros seleccionados
2. **Recopilación feedback** y métricas uso
3. **Refinamiento funcionalidades** basado en uso real
4. **Optimización rendimiento** y escalabilidad
5. **Preparación lanzamiento** comercial

### Fase 4: Lanzamiento y Escalado (Mes 10-12)
1. **Lanzamiento comercial** en Andalucía
2. **Plan marketing** dirigido centros públicos
3. **Programa formación** usuarios
4. **Soporte técnico** especializado
5. **Roadmap evolución** funcionalidades avanzadas

---

## 📊 MÉTRICAS DE ÉXITO

### KPIs Técnicos
- **Tiempo generación horario**: < 30 minutos (vs 40+ horas manual)
- **Tasa éxito optimización**: > 95% horarios sin conflictos críticos
- **Satisfacción preferencias**: > 80% preferencias profesorado cumplidas
- **Disponibilidad sistema**: > 99.5% durante período lectivo
- **Tiempo respuesta**: < 3 segundos operaciones básicas

### KPIs de Negocio
- **Adopción**: 50+ centros primer año
- **Retención**: > 90% renovación anual
- **Satisfacción usuario**: NPS > 50
- **ROI cliente**: > 300% primer año uso
- **Time-to-value**: < 1 semana implementación

### KPIs de Impacto
- **Reducción tiempo gestión**: 80% vs métodos actuales
- **Mejora calidad horarios**: Métricas equilibrio carga profesorado
- **Reducción conflictos**: 95% menos incidencias post-implementación
- **Satisfacción profesorado**: Encuestas clima laboral

---

## 🎯 CONCLUSIONES Y RECOMENDACIONES

### Conclusiones Principales

1. **Oportunidad de Mercado Validada**: Existe una necesidad real y urgente en el sector educativo público andaluz para una solución especializada de gestión de horarios.

2. **Complejidad Técnica Manejable**: Aunque el problema es algorítmicamente complejo, existen enfoques técnicos probados que pueden proporcionar soluciones efectivas.

3. **Diferenciación Clara**: La especialización en centros públicos andaluces y la integración con sistemas oficiales proporciona una ventaja competitiva sostenible.

4. **Viabilidad Económica**: El modelo SaaS es viable con una base de clientes objetivo clara y necesidades validadas.

### Recomendaciones Estratégicas

#### Recomendación 1: Enfoque Iterativo y Validación Temprana
- Desarrollar MVP con funcionalidades core en 6 meses
- Validar con pilotos reales antes de inversión completa
- Iterar basado en feedback usuarios reales

#### Recomendación 2: Especialización Sector Público
- Mantener foco en centros públicos andaluces
- Desarrollar expertise profundo en normativa específica
- Construir relaciones con Consejería Educación

#### Recomendación 3: Excelencia Técnica en Optimización
- Invertir en algoritmos de optimización avanzados
- Considerar técnicas IA/ML para mejora continua
- Asegurar escalabilidad desde diseño inicial

#### Recomendación 4: UX Especializada
- Co-diseñar con jefes de estudios y directores
- Priorizar simplicidad sobre funcionalidades avanzadas
- Invertir en formación y change management

---

**Analista**: Analista Funcional Expert SmartTiming | **Fecha Análisis**: Diciembre 2024

---

*Este análisis funcional proporciona la base completa para el desarrollo de SmartTiming, asegurando que la solución aborde todas las necesidades identificadas en el Instituto Salmedina y sea escalable a otros centros públicos andaluces.*