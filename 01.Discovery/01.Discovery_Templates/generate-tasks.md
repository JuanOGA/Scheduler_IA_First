# Instrucciones Operativas: Generar Tareas de Desarrollo

## Objetivo
Transformar un PRD (Product Requirements Document) aprobado en un conjunto estructurado y detallado de tareas de desarrollo, organizadas por sprints y priorizadas según dependencias técnicas y valor de negocio.

## Contexto del Proceso
Este documento forma parte del flujo de trabajo de AI Product Management, donde el CTO Expert AI Assistant colabora con el equipo de desarrollo para descomponer PRDs en tareas ejecutables, asegurando una implementación eficiente y de alta calidad.

## Roles Involucrados

### Rol Principal
- **CTO Expert AI Assistant**: Responsable de liderar la generación de tareas técnicas

### Roles de Apoyo
- **Senior Python Developer Agent**: Validación de tareas de desarrollo
- **QA Expert Agent**: Definición de tareas de testing
- **DevOps Expert Agent**: Tareas de infraestructura y deployment
- **CPO Expert AI Assistant**: Validación de prioridades de negocio

## Metodología de Generación de Tareas

### Fase 1: Análisis del PRD
```markdown
## Análisis del PRD

### 1.1 Extracción de Requisitos
- **Requisitos Funcionales**: [Lista de funcionalidades core]
- **Requisitos No Funcionales**: [Performance, seguridad, escalabilidad]
- **Dependencias Técnicas**: [Sistemas externos, APIs, servicios]
- **Restricciones**: [Tiempo, recursos, tecnología]

### 1.2 Identificación de Componentes
- **Frontend Components**: [Interfaces de usuario]
- **Backend Services**: [APIs, lógica de negocio]
- **Database Changes**: [Esquemas, migraciones]
- **Infrastructure**: [Deployment, monitoring]
- **Integration Points**: [APIs externas, webhooks]

### 1.3 Análisis de Complejidad
| Componente | Complejidad | Estimación | Dependencias |
|------------|-------------|------------|--------------|
| [Componente 1] | Alta/Media/Baja | X días | [Lista] |
| [Componente 2] | Alta/Media/Baja | X días | [Lista] |
```

### Fase 2: Descomposición en Épicas
```markdown
## Descomposición en Épicas

### Épica 1: [Nombre de la Épica]
**Objetivo**: [Descripción del objetivo de negocio]
**Valor**: [Valor que aporta al usuario final]
**Criterios de Aceptación**:
- [ ] [Criterio 1]
- [ ] [Criterio 2]
- [ ] [Criterio 3]

**Componentes Técnicos**:
- Frontend: [Componentes UI necesarios]
- Backend: [Servicios y APIs]
- Database: [Cambios en esquema]
- Testing: [Estrategia de testing]

**Estimación Total**: [X story points / días]
**Prioridad**: [Alta/Media/Baja]
**Dependencias**: [Otras épicas o sistemas]

### Épica 2: [Nombre de la Épica]
[Repetir estructura]
```

### Fase 3: Creación de User Stories
```markdown
## User Stories

### US-001: [Título de la User Story]
**Como** [tipo de usuario]
**Quiero** [funcionalidad deseada]
**Para** [beneficio o valor]

**Criterios de Aceptación**:
- **Dado que** [contexto inicial]
- **Cuando** [acción del usuario]
- **Entonces** [resultado esperado]

**Definición de Hecho (DoD)**:
- [ ] Código implementado y revisado
- [ ] Unit tests con >80% cobertura
- [ ] Integration tests pasando
- [ ] Documentación actualizada
- [ ] QA testing completado
- [ ] Performance dentro de límites
- [ ] Security review completado

**Tareas Técnicas**:
- [ ] [Tarea técnica 1]
- [ ] [Tarea técnica 2]
- [ ] [Tarea técnica 3]

**Estimación**: [X story points]
**Prioridad**: [Alta/Media/Baja]
**Sprint**: [Número de sprint]
```

## Estructura de Tareas Técnicas

### Categorías de Tareas

#### 1. Tareas de Backend
```markdown
### Backend Tasks

#### BACK-001: Crear API Endpoint para [Funcionalidad]
**Descripción**: Implementar endpoint REST para [descripción específica]

**Especificaciones Técnicas**:
- **Método HTTP**: GET/POST/PUT/DELETE
- **URL**: `/api/v1/[resource]`
- **Parámetros de Entrada**:
  ```json
  {
    "param1": "string",
    "param2": "integer",
    "param3": "boolean"
  }
  ```
- **Respuesta Esperada**:
  ```json
  {
    "status": "success",
    "data": {
      "id": "uuid",
      "created_at": "timestamp"
    }
  }
  ```

**Validaciones Requeridas**:
- [ ] Validación de parámetros de entrada
- [ ] Autenticación y autorización
- [ ] Rate limiting
- [ ] Input sanitization

**Testing**:
- [ ] Unit tests para lógica de negocio
- [ ] Integration tests para endpoint
- [ ] Error handling tests
- [ ] Performance tests

**Estimación**: [X horas]
**Dependencias**: [Lista de dependencias]
**Asignado a**: [Senior Python Developer Agent]
```

#### 2. Tareas de Frontend
```markdown
### Frontend Tasks

#### FRONT-001: Crear Componente [Nombre del Componente]
**Descripción**: Desarrollar componente React para [funcionalidad específica]

**Especificaciones de UI**:
- **Tipo**: [Functional Component/Class Component]
- **Props**:
  ```typescript
  interface ComponentProps {
    prop1: string;
    prop2: number;
    onAction: (data: any) => void;
  }
  ```
- **Estado Local**: [Descripción del estado necesario]
- **Hooks Utilizados**: [useState, useEffect, custom hooks]

**Comportamiento**:
- [ ] Renderizado inicial correcto
- [ ] Manejo de estados de carga
- [ ] Manejo de errores
- [ ] Responsive design
- [ ] Accesibilidad (WCAG)

**Integración**:
- [ ] Conexión con APIs backend
- [ ] Manejo de estado global
- [ ] Routing si aplica
- [ ] Validación de formularios

**Testing**:
- [ ] Unit tests con React Testing Library
- [ ] Integration tests
- [ ] E2E tests con Playwright
- [ ] Visual regression tests

**Estimación**: [X horas]
**Dependencias**: [APIs backend, componentes padre]
**Asignado a**: [Senior Python Developer Agent]
```

#### 3. Tareas de Base de Datos
```markdown
### Database Tasks

#### DB-001: Crear Migración para [Tabla/Cambio]
**Descripción**: Implementar migración de base de datos para [cambio específico]

**Cambios en Esquema**:
```sql
-- Migración UP
CREATE TABLE new_table (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_new_table_email ON new_table(email);

-- Migración DOWN
DROP TABLE IF EXISTS new_table;
```

**Consideraciones**:
- [ ] Backward compatibility
- [ ] Data migration si es necesario
- [ ] Performance impact
- [ ] Rollback strategy

**Validación**:
- [ ] Testing en ambiente de desarrollo
- [ ] Performance testing con datos de prueba
- [ ] Rollback testing
- [ ] Documentation actualizada

**Estimación**: [X horas]
**Dependencias**: [Cambios en código, aprobación DBA]
**Asignado a**: [Senior Python Developer Agent]
```

#### 4. Tareas de Testing
```markdown
### Testing Tasks

#### TEST-001: Implementar Test Suite para [Funcionalidad]
**Descripción**: Crear suite completa de tests para [funcionalidad específica]

**Tipos de Tests**:
- **Unit Tests**:
  - [ ] Tests para lógica de negocio
  - [ ] Tests para utilidades
  - [ ] Tests para validaciones
  - **Cobertura Objetivo**: >80%

- **Integration Tests**:
  - [ ] Tests de API endpoints
  - [ ] Tests de base de datos
  - [ ] Tests de servicios externos
  - **Escenarios**: [Lista de escenarios críticos]

- **E2E Tests**:
  - [ ] Happy path completo
  - [ ] Error scenarios
  - [ ] Edge cases
  - **Herramienta**: Playwright

**Test Data**:
- [ ] Fixtures para datos de prueba
- [ ] Mocks para servicios externos
- [ ] Test database setup

**CI/CD Integration**:
- [ ] Tests ejecutándose en pipeline
- [ ] Reporting de cobertura
- [ ] Fail-fast en errores críticos

**Estimación**: [X horas]
**Dependencias**: [Código implementado]
**Asignado a**: [QA Expert Agent]
```

#### 5. Tareas de DevOps
```markdown
### DevOps Tasks

#### DEVOPS-001: Configurar Deployment para [Servicio]
**Descripción**: Configurar pipeline de deployment para [servicio específico]

**Infraestructura**:
- **Ambiente**: [Development/Staging/Production]
- **Plataforma**: [AWS/GCP/Azure]
- **Servicios**: [Lista de servicios cloud necesarios]

**Pipeline Configuration**:
```yaml
# .github/workflows/deploy.yml
name: Deploy Service
on:
  push:
    branches: [main]
    paths: ['src/service/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest
      - name: Deploy to staging
        run: ./scripts/deploy.sh staging
```

**Monitoring**:
- [ ] Health checks configurados
- [ ] Logging centralizado
- [ ] Métricas de performance
- [ ] Alertas automáticas

**Security**:
- [ ] Secrets management
- [ ] Network security
- [ ] Access controls
- [ ] Vulnerability scanning

**Estimación**: [X horas]
**Dependencias**: [Código de aplicación, configuración de infraestructura]
**Asignado a**: [DevOps Expert Agent]
```

## Planificación de Sprints

### Sprint Planning Template
```markdown
## Sprint [Número]: [Nombre del Sprint]

**Duración**: [2 semanas]
**Objetivo**: [Objetivo principal del sprint]
**Capacity**: [X story points / horas disponibles]

### Sprint Goal
[Descripción clara del objetivo que se quiere alcanzar en este sprint]

### User Stories Incluidas
| ID | Título | Story Points | Asignado | Prioridad |
|----|--------|--------------|----------|-----------|
| US-001 | [Título] | 5 | [Developer] | Alta |
| US-002 | [Título] | 3 | [Developer] | Media |
| US-003 | [Título] | 8 | [Developer] | Alta |

**Total Story Points**: [X]

### Tareas Técnicas
| ID | Descripción | Estimación | Asignado | Dependencias |
|----|-------------|------------|----------|--------------|
| BACK-001 | [Descripción] | 4h | [Developer] | [Lista] |
| FRONT-001 | [Descripción] | 6h | [Developer] | [Lista] |
| TEST-001 | [Descripción] | 3h | [QA] | [Lista] |

### Definition of Done para el Sprint
- [ ] Todas las user stories completadas
- [ ] Code review completado para todos los cambios
- [ ] Tests automatizados pasando
- [ ] Documentación actualizada
- [ ] Deploy a staging exitoso
- [ ] QA sign-off obtenido

### Riesgos y Mitigaciones
| Riesgo | Probabilidad | Impacto | Mitigación |
|--------|-------------|---------|------------|
| [Riesgo 1] | Alta | Alto | [Estrategia] |
| [Riesgo 2] | Media | Medio | [Estrategia] |
```

### Criterios de Priorización

#### 1. Valor de Negocio
```markdown
### Matriz de Valor de Negocio

| Criterio | Peso | Puntuación (1-5) | Total |
|----------|------|------------------|-------|
| Impacto en Revenue | 30% | [X] | [X * 0.3] |
| Satisfacción del Usuario | 25% | [X] | [X * 0.25] |
| Ventaja Competitiva | 20% | [X] | [X * 0.2] |
| Alineación Estratégica | 15% | [X] | [X * 0.15] |
| Urgencia del Mercado | 10% | [X] | [X * 0.1] |

**Puntuación Total**: [Suma total]
```

#### 2. Complejidad Técnica
```markdown
### Matriz de Complejidad Técnica

| Factor | Peso | Puntuación (1-5) | Total |
|--------|------|------------------|-------|
| Complejidad de Implementación | 35% | [X] | [X * 0.35] |
| Dependencias Externas | 25% | [X] | [X * 0.25] |
| Riesgo Técnico | 20% | [X] | [X * 0.2] |
| Esfuerzo de Testing | 10% | [X] | [X * 0.1] |
| Impacto en Arquitectura | 10% | [X] | [X * 0.1] |

**Puntuación Total**: [Suma total]
```

#### 3. Matriz de Priorización
```markdown
### Matriz Valor vs Complejidad

|                | Baja Complejidad | Media Complejidad | Alta Complejidad |
|----------------|------------------|-------------------|------------------|
| **Alto Valor** | 🟢 Hacer Primero | 🟡 Hacer Segundo  | 🟠 Evaluar       |
| **Medio Valor**| 🟡 Hacer Segundo | 🟠 Evaluar        | 🔴 Evitar        |
| **Bajo Valor** | 🟠 Evaluar       | 🔴 Evitar         | 🔴 Evitar        |

**Leyenda**:
- 🟢 Prioridad Alta (Sprint 1-2)
- 🟡 Prioridad Media (Sprint 3-4)
- 🟠 Prioridad Baja (Sprint 5+)
- 🔴 No Priorizar (Backlog)
```

## Estimación de Tareas

### Técnicas de Estimación

#### 1. Planning Poker
```markdown
### Planning Poker Session

**Participantes**:
- CTO Expert AI Assistant (Facilitador)
- Senior Python Developer Agent
- QA Expert Agent
- DevOps Expert Agent

**Escala de Fibonacci**: 1, 2, 3, 5, 8, 13, 21

**Proceso**:
1. Presentar user story
2. Discusión inicial (5 min)
3. Estimación individual
4. Revelar estimaciones
5. Discutir diferencias
6. Re-estimación si es necesario
7. Consenso final

**Criterios de Estimación**:
- **1 punto**: Tarea muy simple, <2 horas
- **2 puntos**: Tarea simple, 2-4 horas
- **3 puntos**: Tarea moderada, 4-8 horas
- **5 puntos**: Tarea compleja, 1-2 días
- **8 puntos**: Tarea muy compleja, 2-3 días
- **13 puntos**: Épica, necesita descomposición
```

#### 2. T-Shirt Sizing
```markdown
### T-Shirt Sizing para Épicas

| Tamaño | Descripción | Story Points | Duración |
|--------|-------------|--------------|----------|
| XS | Cambio mínimo | 1-2 | <1 día |
| S | Funcionalidad simple | 3-5 | 1-2 días |
| M | Funcionalidad estándar | 8-13 | 1 semana |
| L | Funcionalidad compleja | 21-34 | 2-3 semanas |
| XL | Épica grande | 55+ | 1+ mes |

**Nota**: Tareas XL deben descomponerse antes de implementación
```

### Factores de Ajuste
```markdown
### Factores de Ajuste de Estimaciones

#### Factores que Incrementan Estimación (+)
- [ ] **Tecnología Nueva** (+25%): Uso de tecnologías no familiares
- [ ] **Dependencias Externas** (+20%): APIs de terceros, servicios externos
- [ ] **Requisitos Ambiguos** (+30%): Falta de claridad en requisitos
- [ ] **Integración Compleja** (+15%): Múltiples sistemas involucrados
- [ ] **Performance Crítica** (+20%): Requisitos estrictos de performance

#### Factores que Reducen Estimación (-)
- [ ] **Código Reutilizable** (-15%): Componentes existentes utilizables
- [ ] **Experiencia del Equipo** (-10%): Equipo experto en la tecnología
- [ ] **Herramientas Maduras** (-10%): Uso de herramientas bien establecidas
- [ ] **Requisitos Claros** (-15%): Documentación detallada y clara

**Fórmula de Ajuste**:
```
Estimación Final = Estimación Base × (1 + Σ Factores Positivos - Σ Factores Negativos)
```
```

## Gestión de Dependencias

### Tipos de Dependencias
```markdown
### Matriz de Dependencias

#### Dependencias Técnicas
| Tarea | Depende de | Tipo | Impacto | Mitigación |
|-------|------------|------|---------|------------|
| FRONT-001 | BACK-001 | Bloqueante | Alto | Mock API para desarrollo paralelo |
| TEST-001 | FRONT-001, BACK-001 | Secuencial | Medio | Preparar test data con anticipación |

#### Dependencias de Recursos
| Tarea | Recurso Necesario | Disponibilidad | Alternativa |
|-------|-------------------|----------------|-------------|
| DEVOPS-001 | Acceso a AWS | Pendiente | Usar ambiente local |
| DB-001 | DBA Review | 2 días | Peer review interno |

#### Dependencias Externas
| Tarea | Dependencia Externa | SLA | Contacto | Plan B |
|-------|-------------------|-----|----------|--------|
| INT-001 | API de Terceros | 99.9% | [Contacto] | Cache + fallback |
| PAY-001 | Gateway de Pago | 99.95% | [Contacto] | Proveedor alternativo |
```

### Estrategias de Mitigación
```markdown
### Estrategias para Manejar Dependencias

#### 1. Desarrollo Paralelo
- **Mock Services**: Crear mocks de APIs no disponibles
- **Stub Implementation**: Implementaciones temporales
- **Feature Flags**: Activar/desactivar funcionalidades

#### 2. Desacoplamiento
- **Event-Driven Architecture**: Reducir dependencias directas
- **API Contracts**: Definir contratos antes de implementación
- **Microservices**: Independencia de servicios

#### 3. Contingencia
- **Plan B**: Alternativas para dependencias críticas
- **Timeboxing**: Límites de tiempo para dependencias
- **Escalation Path**: Proceso de escalación para bloqueos
```

## Tracking y Monitoreo

### Métricas de Desarrollo
```markdown
### Dashboard de Métricas de Sprint

#### Velocity Tracking
- **Velocity Promedio**: [X story points por sprint]
- **Velocity Actual**: [X story points]
- **Tendencia**: [Ascendente/Descendente/Estable]

#### Burndown Chart
```
Story Points Restantes
    |
 40 |●
    |  ●
 30 |    ●
    |      ●
 20 |        ●
    |          ●
 10 |            ●
    |              ●
  0 |________________●
    1  2  3  4  5  6  7  8  9  10
              Días del Sprint
```

#### Métricas de Calidad
- **Code Coverage**: [X%]
- **Bug Rate**: [X bugs por story point]
- **Cycle Time**: [X días promedio por tarea]
- **Lead Time**: [X días desde idea hasta producción]

#### Métricas de Equipo
- **Capacity Utilization**: [X%]
- **Blocked Tasks**: [X tareas]
- **WIP Limit Adherence**: [X%]
```

### Herramientas de Tracking
```markdown
### Stack de Herramientas

#### Project Management
- **Jira**: Tracking de user stories y tareas
- **Linear**: Gestión ágil de producto
- **Azure DevOps**: Integración con desarrollo

#### Development
- **GitHub**: Code repository y reviews
- **GitHub Actions**: CI/CD pipelines
- **SonarQube**: Code quality analysis

#### Communication
- **Slack**: Comunicación del equipo
- **Notion**: Documentación colaborativa
- **Miro**: Diagramas y planning

#### Monitoring
- **DataDog**: Performance monitoring
- **Sentry**: Error tracking
- **New Relic**: Application performance
```

## Templates y Checklists

### Template de User Story
```markdown
# User Story Template

## US-[ID]: [Título Descriptivo]

### Story Description
**Como** [tipo de usuario]
**Quiero** [funcionalidad deseada]  
**Para** [beneficio o valor obtenido]

### Acceptance Criteria
- **Dado que** [contexto inicial]
- **Cuando** [acción del usuario]
- **Entonces** [resultado esperado]

### Technical Requirements
- [ ] [Requisito técnico 1]
- [ ] [Requisito técnico 2]
- [ ] [Requisito técnico 3]

### Definition of Done
- [ ] Código implementado y revisado
- [ ] Unit tests con >80% cobertura
- [ ] Integration tests pasando
- [ ] Documentación actualizada
- [ ] QA testing completado
- [ ] Performance validada
- [ ] Security review completado
- [ ] Deployed to staging

### Estimation
- **Story Points**: [X]
- **Hours**: [X]
- **Complexity**: [Alta/Media/Baja]

### Dependencies
- [Lista de dependencias]

### Notes
[Notas adicionales, consideraciones especiales]
```

### Checklist de Generación de Tareas
```markdown
## Checklist: Generación de Tareas

### Pre-Planning
- [ ] PRD revisado y aprobado
- [ ] Stakeholders identificados
- [ ] Recursos disponibles confirmados
- [ ] Timeline establecido

### Análisis
- [ ] Requisitos funcionales extraídos
- [ ] Requisitos no funcionales identificados
- [ ] Dependencias técnicas mapeadas
- [ ] Riesgos técnicos evaluados

### Descomposición
- [ ] Épicas definidas
- [ ] User stories creadas
- [ ] Tareas técnicas detalladas
- [ ] Criterios de aceptación claros

### Estimación
- [ ] Planning poker completado
- [ ] Estimaciones validadas por equipo
- [ ] Factores de ajuste aplicados
- [ ] Capacity planning realizado

### Planificación
- [ ] Sprints planificados
- [ ] Dependencias resueltas
- [ ] Recursos asignados
- [ ] Timeline validado

### Documentación
- [ ] Tareas documentadas en herramienta
- [ ] Criterios de aceptación claros
- [ ] Definition of Done establecida
- [ ] Comunicación a stakeholders
```

## Mejores Prácticas

### Para la Generación de Tareas
1. **Granularidad Apropiada**: Tareas de 4-8 horas máximo
2. **Criterios Claros**: Cada tarea debe tener criterios de aceptación específicos
3. **Testabilidad**: Todas las tareas deben ser testeable
4. **Independencia**: Minimizar dependencias entre tareas

### Para la Estimación
1. **Estimación Relativa**: Comparar con tareas similares anteriores
2. **Incluir Testing**: Considerar tiempo de testing en estimaciones
3. **Buffer para Incertidumbre**: Agregar buffer para tareas complejas
4. **Revisión Regular**: Ajustar estimaciones basado en datos históricos

### Para la Priorización
1. **Valor Primero**: Priorizar por valor de negocio
2. **Dependencias**: Considerar dependencias técnicas
3. **Riesgo**: Abordar riesgos técnicos temprano
4. **Feedback Loops**: Crear oportunidades para feedback temprano

**Recuerda**: La generación efectiva de tareas es crucial para el éxito del desarrollo. Invierte tiempo en crear tareas claras, bien estimadas y apropiadamente priorizadas que permitan al equipo trabajar de manera eficiente y entregar valor consistentemente.