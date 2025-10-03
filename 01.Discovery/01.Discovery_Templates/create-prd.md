# Instrucciones Operativas: Crear PRD (Product Requirements Document)

## Objetivo
Crear un Product Requirements Document (PRD) completo y detallado que sirva como guía definitiva para el desarrollo de productos SAAS B2B, asegurando alineación entre todos los stakeholders y proporcionando claridad técnica y de negocio.

## Contexto del Proceso
Este documento forma parte del flujo de trabajo de AI Product Management, donde el CPO Expert AI Assistant colabora con otros expertos para crear PRDs de alta calidad que cumplan con estándares técnicos, regulatorios y de negocio.

## Roles Involucrados

### Rol Principal
- **CPO Expert AI Assistant**: Responsable de liderar la creación del PRD

### Roles de Apoyo
- **CTO Expert AI Assistant**: Validación técnica y arquitectural
- **Regulatory Expert AI Assistant**: Compliance y aspectos regulatorios
- **AI Research Expert**: Validación de componentes de IA/ML
- **Stakeholders de Negocio**: Product Managers, Business Analysts

## Estructura del PRD

### 1. Executive Summary
```markdown
## Executive Summary

### Problema a Resolver
[Descripción clara del problema de negocio que se está resolviendo]

### Propuesta de Valor
[Valor único que el producto proporcionará a los usuarios]

### Métricas de Éxito
- Métrica 1: [Descripción y objetivo]
- Métrica 2: [Descripción y objetivo]
- Métrica 3: [Descripción y objetivo]

### Inversión Requerida
- Desarrollo: [X semanas/sprints]
- Recursos: [X desarrolladores, X diseñadores]
- Presupuesto: [Estimación si aplica]
```

### 2. Contexto y Justificación
```markdown
## Contexto y Justificación

### Análisis del Mercado
[Investigación de mercado relevante, competidores, oportunidades]

### Feedback de Usuarios
[Insights de usuarios actuales, encuestas, entrevistas]

### Alineación Estratégica
[Cómo se alinea con la estrategia de producto y objetivos de la empresa]

### Riesgos de No Hacer
[Consecuencias de no desarrollar esta funcionalidad]
```

### 3. Definición del Usuario
```markdown
## Definición del Usuario

### Personas Objetivo
#### Persona 1: [Nombre del Persona]
- **Rol**: [Título del trabajo]
- **Responsabilidades**: [Principales responsabilidades]
- **Pain Points**: [Problemas actuales]
- **Objetivos**: [Qué quiere lograr]
- **Comportamiento**: [Cómo interactúa con productos similares]

#### Persona 2: [Nombre del Persona]
[Repetir estructura]

### User Journey
[Mapeo del journey del usuario desde el descubrimiento hasta el éxito]

### Casos de Uso Principales
1. **Caso de Uso 1**: [Descripción detallada]
2. **Caso de Uso 2**: [Descripción detallada]
3. **Caso de Uso 3**: [Descripción detallada]
```

### 4. Requisitos Funcionales
```markdown
## Requisitos Funcionales

### Funcionalidades Core
#### F1: [Nombre de la Funcionalidad]
- **Descripción**: [Qué hace la funcionalidad]
- **Criterios de Aceptación**:
  - Dado que [contexto]
  - Cuando [acción]
  - Entonces [resultado esperado]
- **Prioridad**: [Alta/Media/Baja]
- **Complejidad**: [Alta/Media/Baja]
- **Dependencias**: [Otras funcionalidades o sistemas]

#### F2: [Nombre de la Funcionalidad]
[Repetir estructura]

### Funcionalidades Secundarias
[Funcionalidades importantes pero no críticas para el MVP]

### Funcionalidades Futuras
[Funcionalidades consideradas para versiones posteriores]
```

### 5. Requisitos No Funcionales
```markdown
## Requisitos No Funcionales

### Performance
- **Tiempo de Respuesta**: [Máximo X segundos para operaciones críticas]
- **Throughput**: [X transacciones por segundo]
- **Disponibilidad**: [99.9% uptime]
- **Escalabilidad**: [Soportar X usuarios concurrentes]

### Seguridad
- **Autenticación**: [Métodos requeridos]
- **Autorización**: [Niveles de acceso]
- **Encriptación**: [Datos en tránsito y reposo]
- **Compliance**: [Regulaciones aplicables]

### Usabilidad
- **Accesibilidad**: [Estándares WCAG]
- **Compatibilidad**: [Navegadores y dispositivos soportados]
- **Internacionalización**: [Idiomas soportados]

### Integración
- **APIs Externas**: [Servicios de terceros requeridos]
- **Sistemas Internos**: [Integraciones con sistemas existentes]
- **Formatos de Datos**: [Estándares de intercambio]
```

### 6. Especificaciones Técnicas
```markdown
## Especificaciones Técnicas

### Arquitectura Propuesta
```
[Diagrama de arquitectura de alto nivel]
```

### Stack Tecnológico
- **Frontend**: [Tecnologías específicas]
- **Backend**: [Framework y lenguajes]
- **Base de Datos**: [Tipo y tecnología]
- **Infraestructura**: [Cloud provider y servicios]
- **Herramientas**: [Monitoring, logging, etc.]

### Modelo de Datos
```sql
-- Esquema principal de base de datos
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Tablas adicionales según necesidad
```

### APIs y Endpoints
#### Endpoint 1: [Nombre]
- **Método**: GET/POST/PUT/DELETE
- **URL**: `/api/v1/resource`
- **Parámetros**: [Descripción de parámetros]
- **Respuesta**: [Formato de respuesta]
- **Códigos de Estado**: [200, 400, 401, etc.]

### Consideraciones de Seguridad
- [Medidas específicas de seguridad]
- [Validaciones requeridas]
- [Manejo de datos sensibles]
```

### 7. Diseño y UX
```markdown
## Diseño y UX

### Principios de Diseño
- [Principio 1: Descripción]
- [Principio 2: Descripción]
- [Principio 3: Descripción]

### Wireframes y Mockups
[Enlaces o referencias a diseños]

### Flujos de Usuario
#### Flujo 1: [Nombre del Flujo]
1. Usuario accede a [pantalla]
2. Usuario realiza [acción]
3. Sistema muestra [resultado]
4. Usuario puede [siguiente acción]

### Componentes de UI
- **Componente 1**: [Descripción y comportamiento]
- **Componente 2**: [Descripción y comportamiento]

### Responsive Design
- **Desktop**: [Consideraciones específicas]
- **Tablet**: [Adaptaciones necesarias]
- **Mobile**: [Optimizaciones móviles]
```

### 8. Plan de Implementación
```markdown
## Plan de Implementación

### Fases de Desarrollo
#### Fase 1: MVP (Semanas 1-4)
- **Objetivos**: [Funcionalidades mínimas viables]
- **Entregables**:
  - [Entregable 1]
  - [Entregable 2]
- **Criterios de Éxito**: [Métricas específicas]

#### Fase 2: Mejoras (Semanas 5-8)
- **Objetivos**: [Funcionalidades adicionales]
- **Entregables**:
  - [Entregable 1]
  - [Entregable 2]

#### Fase 3: Optimización (Semanas 9-12)
- **Objetivos**: [Performance y UX]
- **Entregables**:
  - [Entregable 1]
  - [Entregable 2]

### Dependencias Críticas
- **Dependencia 1**: [Descripción y impacto]
- **Dependencia 2**: [Descripción y impacto]

### Riesgos y Mitigaciones
| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|------------|
| [Riesgo 1] | Alta/Media/Baja | Alto/Medio/Bajo | [Estrategia de mitigación] |
| [Riesgo 2] | Alta/Media/Baja | Alto/Medio/Bajo | [Estrategia de mitigación] |
```

### 9. Testing y QA
```markdown
## Testing y QA

### Estrategia de Testing
- **Unit Tests**: [Cobertura objetivo: X%]
- **Integration Tests**: [Componentes críticos]
- **E2E Tests**: [Flujos principales]
- **Performance Tests**: [Escenarios de carga]
- **Security Tests**: [Vulnerabilidades comunes]

### Criterios de Aceptación
#### Funcionales
- [ ] Todas las funcionalidades core implementadas
- [ ] Casos de uso principales funcionando
- [ ] Validaciones de entrada implementadas

#### No Funcionales
- [ ] Performance dentro de límites establecidos
- [ ] Seguridad validada
- [ ] Accesibilidad cumple estándares

### Plan de Testing
#### Pre-Development
- [ ] Revisión de PRD con stakeholders
- [ ] Validación técnica con CTO
- [ ] Aprobación de diseños

#### Durante Development
- [ ] Testing continuo en cada sprint
- [ ] Code reviews obligatorios
- [ ] Testing de integración

#### Pre-Launch
- [ ] UAT con usuarios beta
- [ ] Performance testing
- [ ] Security audit
- [ ] Accessibility audit
```

### 10. Métricas y Monitoreo
```markdown
## Métricas y Monitoreo

### KPIs de Producto
- **Adopción**: [Métrica específica]
- **Engagement**: [Métrica específica]
- **Retención**: [Métrica específica]
- **Satisfacción**: [Métrica específica]

### Métricas Técnicas
- **Performance**: [Tiempo de respuesta, throughput]
- **Disponibilidad**: [Uptime, error rates]
- **Uso de Recursos**: [CPU, memoria, storage]

### Herramientas de Monitoreo
- **Analytics**: [Google Analytics, Mixpanel, etc.]
- **Performance**: [New Relic, DataDog, etc.]
- **Errors**: [Sentry, Rollbar, etc.]
- **User Feedback**: [Hotjar, FullStory, etc.]

### Dashboards
- **Business Dashboard**: [Métricas de negocio]
- **Technical Dashboard**: [Métricas técnicas]
- **User Experience Dashboard**: [Métricas de UX]
```

### 11. Go-to-Market
```markdown
## Go-to-Market

### Estrategia de Lanzamiento
- **Soft Launch**: [Usuarios beta limitados]
- **Gradual Rollout**: [Incremento progresivo]
- **Full Launch**: [Disponibilidad general]

### Comunicación
#### Interna
- [ ] Entrenamiento al equipo de soporte
- [ ] Documentación para ventas
- [ ] Actualización de materiales de marketing

#### Externa
- [ ] Comunicación a usuarios existentes
- [ ] Actualización de website
- [ ] Campañas de marketing
- [ ] PR y comunicados

### Soporte Post-Launch
- **Documentación**: [Guías de usuario, FAQs]
- **Entrenamiento**: [Webinars, tutoriales]
- **Soporte**: [Canales de comunicación]
```

## Proceso de Creación del PRD

### Paso 1: Investigación y Análisis
1. **Recopilar Información**
   - Analizar feedback de usuarios
   - Investigar competidores
   - Revisar métricas actuales
   - Consultar con stakeholders

2. **Definir Problema**
   - Articular el problema claramente
   - Cuantificar el impacto
   - Identificar usuarios afectados

### Paso 2: Ideación y Validación
1. **Generar Soluciones**
   - Brainstorming con equipo
   - Evaluar alternativas
   - Considerar restricciones técnicas

2. **Validar Propuesta**
   - Consultar con CTO Expert para viabilidad técnica
   - Revisar con Regulatory Expert si aplica
   - Validar con AI Research Expert para componentes de IA

### Paso 3: Documentación Detallada
1. **Escribir PRD**
   - Seguir estructura establecida
   - Incluir todos los elementos requeridos
   - Mantener claridad y precisión

2. **Revisión Iterativa**
   - Revisar con stakeholders
   - Incorporar feedback
   - Refinar detalles

### Paso 4: Aprobación y Comunicación
1. **Obtener Aprobaciones**
   - Aprobación técnica del CTO
   - Aprobación de negocio del CPO
   - Aprobación regulatoria si aplica

2. **Comunicar PRD**
   - Presentar a equipos de desarrollo
   - Distribuir a stakeholders
   - Crear resumen ejecutivo

## Criterios de Calidad

### Completitud
- [ ] Todos los elementos de la estructura están presentes
- [ ] Información suficiente para comenzar desarrollo
- [ ] Criterios de aceptación claros y medibles

### Claridad
- [ ] Lenguaje claro y sin ambigüedades
- [ ] Diagramas y ejemplos cuando sea necesario
- [ ] Terminología consistente

### Viabilidad
- [ ] Técnicamente factible con recursos disponibles
- [ ] Alineado con capacidades del equipo
- [ ] Realista en términos de tiempo y presupuesto

### Alineación
- [ ] Alineado con estrategia de producto
- [ ] Consideraciones de compliance incluidas
- [ ] Validado por expertos técnicos

## Templates y Herramientas

### Template de PRD
```markdown
# PRD: [Nombre del Producto/Feature]

**Versión**: 1.0
**Fecha**: [Fecha]
**Autor**: [CPO Expert AI Assistant]
**Revisores**: [Lista de revisores]
**Estado**: [Draft/Review/Approved]

## 1. Executive Summary
[Contenido según estructura]

## 2. Contexto y Justificación
[Contenido según estructura]

[... continuar con todas las secciones]
```

### Checklist de Revisión
```markdown
## Checklist de Revisión de PRD

### Contenido
- [ ] Executive summary claro y conciso
- [ ] Problema bien definido y cuantificado
- [ ] Usuarios objetivo claramente identificados
- [ ] Requisitos funcionales completos
- [ ] Requisitos no funcionales especificados
- [ ] Especificaciones técnicas detalladas
- [ ] Plan de implementación realista
- [ ] Estrategia de testing definida
- [ ] Métricas de éxito establecidas

### Calidad
- [ ] Información precisa y actualizada
- [ ] Lenguaje claro y sin ambigüedades
- [ ] Diagramas y ejemplos incluidos
- [ ] Referencias y fuentes citadas

### Validación
- [ ] Revisado por CTO Expert
- [ ] Validado por Regulatory Expert (si aplica)
- [ ] Aprobado por stakeholders de negocio
- [ ] Feedback incorporado
```

## Mejores Prácticas

### Durante la Creación
1. **Involucrar Stakeholders Temprano**: Obtener input desde el inicio
2. **Iterar Frecuentemente**: Revisar y refinar continuamente
3. **Mantener Foco**: Evitar scope creep durante la documentación
4. **Documentar Decisiones**: Registrar el razonamiento detrás de decisiones importantes

### Para la Calidad
1. **Usar Datos**: Basar decisiones en datos cuantitativos cuando sea posible
2. **Ser Específico**: Evitar generalidades, proporcionar detalles concretos
3. **Considerar Edge Cases**: Pensar en escenarios no típicos
4. **Planificar para Escalabilidad**: Considerar crecimiento futuro

### Para la Colaboración
1. **Comunicación Clara**: Mantener a todos los stakeholders informados
2. **Documentar Cambios**: Registrar modificaciones y razones
3. **Facilitar Revisiones**: Hacer el PRD fácil de revisar y comentar
4. **Seguimiento Post-Aprobación**: Monitorear implementación vs. PRD

## Herramientas Recomendadas

### Documentación
- **Notion**: Para PRDs colaborativos
- **Confluence**: Para documentación empresarial
- **Google Docs**: Para colaboración en tiempo real
- **Markdown**: Para documentación técnica

### Diagramas
- **Miro/Mural**: Para wireframes y flujos
- **Lucidchart**: Para diagramas técnicos
- **Figma**: Para mockups y prototipos
- **Draw.io**: Para diagramas de arquitectura

### Gestión
- **Jira**: Para tracking de requisitos
- **Linear**: Para gestión de producto
- **Asana**: Para coordinación de tareas
- **Slack**: Para comunicación del equipo

**Recuerda**: Un PRD de calidad es la base para el éxito del desarrollo de producto. Invierte tiempo en crear un documento completo, claro y bien validado que sirva como guía definitiva para todo el equipo.