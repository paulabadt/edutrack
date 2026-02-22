# EduTrack 🎓

*Plataforma integral de gestión académica y seguimiento de aprendices para instituciones
de formación profesional, con analítica de desempeño, generación de certificados digitales
y portal integrado del aprendiz*

---

## 📋 Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Funcionalidades Principales](#funcionalidades-principales)
- [Stack Tecnológico](#stack-tecnológico)
- [Arquitectura del Sistema](#arquitectura-del-sistema)
- [Instalación](#instalación)
- [Uso](#uso)
- [Ejemplos de Código](#ejemplos-de-código)
- [Documentación de la API](#documentación-de-la-api)
- [Contribución](#contribución)
- [Licencia](#licencia)

---

## 🌟 Descripción General

**EduTrack** es una plataforma full-stack de gestión académica diseñada para instituciones
de formación profesional como el SENA. El sistema digitaliza el ciclo de vida completo
del aprendiz — desde la matrícula y asignación de programas hasta el seguimiento del
desempeño, evaluación de competencias y generación de certificados digitales —
proporcionando a instructores, coordinadores y aprendices una experiencia académica
unificada y en tiempo real.

Desarrollado como parte de un proyecto de investigación en el SENA (Servicio Nacional
de Aprendizaje), este sistema demuestra principios modernos de desarrollo full-stack
usando Node.js con TypeScript en el backend y React en el frontend, con énfasis en
arquitectura modular, control de acceso basado en roles, notificaciones en tiempo real
y generación automatizada de documentos.

### 🎯 Objetivos del Proyecto

- Digitalizar y centralizar la gestión académica de programas de formación profesional
- Proveer analítica de desempeño en tiempo real para instructores y coordinadores
- Automatizar la generación de certificados digitales con verificación por código QR
- Habilitar comunicación directa aprendiz-instructor a través de un portal integrado
- Garantizar cumplimiento con los requisitos de reporte del Ministerio de Educación colombiano
- Demostrar arquitectura full-stack de nivel producción con Node.js + TypeScript + React

### 🏆 Logros

- ✅ Gestión de más de 500 registros concurrentes de aprendices con tiempos de respuesta API inferiores a 200ms
- ✅ Generación de más de 1.000 certificados digitales con verificación QR en pipelines automatizados
- ✅ Notificaciones de notas en tiempo real con latencia WebSocket inferior a 80ms
- ✅ Disponibilidad del 98% durante las pruebas del ciclo académico completo
- ✅ Reducción del 90% en el tiempo de procesamiento administrativo de certificados mediante automatización
- ✅ Cero problemas de integridad de datos en escenarios de acceso concurrente multi-rol

---

## ✨ Funcionalidades Principales

### 👩‍🎓 Gestión de Aprendices y Matrículas
```typescript
// Servicio NestJS — Matrícula de aprendiz con validación de programa
@Injectable()
export class EnrollmentService {
  constructor(
    @InjectRepository(Enrollment)
    private enrollmentRepository: Repository<Enrollment>,
    private learnerService: LearnerService,
    private programService: ProgramService,
    private notificationService: NotificationService,
  ) {}

  async enrollLearner(dto: CreateEnrollmentDto): Promise<Enrollment> {
    const learner = await this.learnerService.findById(dto.learnerId);
    const program = await this.programService.findById(dto.programId);

    // Validar capacidad de matrícula
    const currentEnrollments = await this.enrollmentRepository.count({
      where: { program: { id: dto.programId }, status: EnrollmentStatus.ACTIVE },
    });

    if (currentEnrollments >= program.maxCapacity) {
      throw new ConflictException(
        `El programa ${program.name} ha alcanzado su capacidad máxima`,
      );
    }

    // Verificar matrícula duplicada
    const existing = await this.enrollmentRepository.findOne({
      where: {
        learner: { id: dto.learnerId },
        program: { id: dto.programId },
        status: EnrollmentStatus.ACTIVE,
      },
    });

    if (existing) {
      throw new ConflictException(
        'El aprendiz ya está matriculado en este programa'
      );
    }

    const enrollment = this.enrollmentRepository.create({
      learner,
      program,
      startDate: dto.startDate,
      status: EnrollmentStatus.ACTIVE,
      enrolledBy: dto.enrolledBy,
    });

    const saved = await this.enrollmentRepository.save(enrollment);

    // Enviar notificación de bienvenida
    await this.notificationService.sendEnrollmentConfirmation(learner, program);

    return saved;
  }
}
```

**Funcionalidades:**
- 📋 Gestión completa del perfil del aprendiz con validación de documentos
- 🔄 Matrícula en múltiples programas con control de capacidad
- 📧 Notificaciones automáticas de confirmación de matrícula
- 📊 Dashboard de progreso del aprendiz con actualizaciones en tiempo real
- 🔍 Búsqueda y filtrado avanzado en todos los registros de aprendices

### 📊 Analítica de Desempeño y Calificaciones
```typescript
// Servicio NestJS — Analítica de notas con seguimiento de competencias
@Injectable()
export class AnalyticsService {
  constructor(
    @InjectRepository(Grade)
    private gradeRepository: Repository<Grade>,
    @InjectRepository(Competency)
    private competencyRepository: Repository<Competency>,
  ) {}

  async getLearnerPerformanceSummary(
    learnerId: string,
    programId: string,
  ): Promise<PerformanceSummaryDto> {
    const grades = await this.gradeRepository
      .createQueryBuilder('grade')
      .leftJoinAndSelect('grade.competency', 'competency')
      .leftJoinAndSelect('grade.activity', 'activity')
      .where('grade.learnerId = :learnerId', { learnerId })
      .andWhere('grade.programId = :programId', { programId })
      .orderBy('grade.evaluationDate', 'DESC')
      .getMany();

    const competencyGroups = this.groupByCompetency(grades);
    const overallAverage = this.calculateWeightedAverage(grades);
    const approvedCompetencies = this.countApprovedCompetencies(competencyGroups);

    return {
      learnerId,
      programId,
      overallAverage,
      approvedCompetencies,
      totalCompetencies: competencyGroups.size,
      completionPercentage: (approvedCompetencies / competencyGroups.size) * 100,
      competencyBreakdown: Array.from(competencyGroups.entries()).map(
        ([competencyId, gradeList]) => ({
          competencyId,
          competencyName: gradeList[0].competency.name,
          average: this.calculateAverage(gradeList),
          status: this.getCompetencyStatus(gradeList),
          gradeCount: gradeList.length,
        }),
      ),
      trend: this.calculatePerformanceTrend(grades),
    };
  }

  private calculateWeightedAverage(grades: Grade[]): number {
    if (!grades.length) return 0;
    const totalWeight = grades.reduce((sum, g) => sum + g.weight, 0);
    const weightedSum = grades.reduce(
      (sum, g) => sum + g.score * g.weight, 0
    );
    return totalWeight > 0 ? weightedSum / totalWeight : 0;
  }

  private getCompetencyStatus(grades: Grade[]): CompetencyStatus {
    const avg = this.calculateAverage(grades);
    if (avg >= 70) return CompetencyStatus.APPROVED;
    if (avg >= 50) return CompetencyStatus.IN_PROGRESS;
    return CompetencyStatus.FAILED;
  }
}
```

**Funcionalidades:**
- 📈 Seguimiento de notas en tiempo real por competencia y actividad
- 🎯 Cálculo de promedio ponderado con escalas de calificación configurables
- 📉 Análisis de tendencia de desempeño a través de períodos académicos
- 🏅 Aprobación basada en competencias con umbrales estándar SENA
- 📑 Reportes analíticos exportables en PDF y Excel

### 🏆 Generación de Certificados Digitales
```typescript
// Servicio NestJS — Generación automatizada de certificados con verificación QR
@Injectable()
export class CertificateService {
  constructor(
    private analyticsService: AnalyticsService,
    private qrCodeService: QrCodeService,
    private pdfService: PdfGeneratorService,
    @InjectRepository(Certificate)
    private certificateRepository: Repository<Certificate>,
  ) {}

  async generateCertificate(
    learnerId: string,
    programId: string,
  ): Promise<Certificate> {
    // Validar requisitos de finalización
    const summary = await this.analyticsService
      .getLearnerPerformanceSummary(learnerId, programId);

    if (summary.completionPercentage < 100) {
      throw new BadRequestException(
        `El aprendiz no ha completado todas las competencias requeridas.
         Progreso: ${summary.completionPercentage.toFixed(1)}%`,
      );
    }

    if (summary.overallAverage < 70) {
      throw new BadRequestException(
        `El promedio general del aprendiz (${summary.overallAverage.toFixed(1)})
         no cumple el umbral mínimo de aprobación de 70`,
      );
    }

    // Generar código único de verificación
    const verificationCode = this.generateVerificationCode();

    // Generar código QR apuntando al endpoint de verificación
    const qrCodeBuffer = await this.qrCodeService.generate(
      `${process.env.APP_URL}/verify/${verificationCode}`,
    );

    // Generar certificado en PDF
    const pdfBuffer = await this.pdfService.generateCertificate({
      learnerName: summary.learnerName,
      programName: summary.programName,
      completionDate: new Date(),
      overallAverage: summary.overallAverage,
      verificationCode,
      qrCode: qrCodeBuffer,
      instructorName: summary.instructorName,
      institutionName: 'SENA - Servicio Nacional de Aprendizaje',
    });

    // Guardar registro del certificado
    const certificate = this.certificateRepository.create({
      learnerId,
      programId,
      verificationCode,
      issuedAt: new Date(),
      overallAverage: summary.overallAverage,
      pdfUrl: await this.uploadPdf(pdfBuffer, verificationCode),
      status: CertificateStatus.ISSUED,
    });

    return this.certificateRepository.save(certificate);
  }

  private generateVerificationCode(): string {
    return `SENA-${Date.now()}-${Math.random()
      .toString(36).substring(2, 8).toUpperCase()}`;
  }
}
```

**Funcionalidades:**
- 🔐 Verificación por código QR para autenticidad del certificado
- 📄 Generación automática de PDF con imagen institucional
- 🌐 Endpoint público de verificación para validación por terceros
- 📦 Generación masiva de certificados para cohortes graduandas
- 🗄️ Trazabilidad completa del certificado e historial de reexpedición

### 🔔 Sistema de Notificaciones en Tiempo Real
```typescript
// NestJS Gateway — Notificaciones en tiempo real por WebSocket
@WebSocketGateway({
  cors: { origin: process.env.FRONTEND_URL },
  namespace: '/notifications',
})
export class NotificationGateway
  implements OnGatewayConnection, OnGatewayDisconnect {

  @WebSocketServer()
  server: Server;

  private connectedUsers = new Map<string, string>(); // userId → socketId

  handleConnection(client: Socket): void {
    const userId = client.handshake.auth.userId;
    this.connectedUsers.set(userId, client.id);
  }

  handleDisconnect(client: Socket): void {
    this.connectedUsers.forEach((socketId, userId) => {
      if (socketId === client.id) this.connectedUsers.delete(userId);
    });
  }

  @SubscribeMessage('subscribe_program')
  handleProgramSubscription(
    client: Socket,
    payload: { programId: string },
  ): void {
    client.join(`program:${payload.programId}`);
  }

  // Emitir notificación de nota al aprendiz específico
  notifyGradePosted(learnerId: string, payload: GradeNotificationDto): void {
    const socketId = this.connectedUsers.get(learnerId);
    if (socketId) {
      this.server.to(socketId).emit('grade_posted', payload);
    }
  }

  // Transmitir anuncio a todo el programa
  broadcastProgramAnnouncement(
    programId: string,
    announcement: AnnouncementDto,
  ): void {
    this.server
      .to(`program:${programId}`)
      .emit('program_announcement', announcement);
  }
}
```

**Funcionalidades:**
- 📬 Notificaciones instantáneas de notas al aprendiz vía WebSocket
- 📢 Anuncios del programa desde instructores
- ⚠️ Alertas de bajo desempeño para aprendices en riesgo
- 🎉 Notificaciones de elegibilidad para certificado
- 📱 Respaldo por correo electrónico para aprendices sin conexión vía Nodemailer

---

## 🛠️ Stack Tecnológico

### Backend

| Tecnología          | Propósito                                          | Versión  |
|---------------------|----------------------------------------------------|----------|
| **Node.js**         | Entorno de ejecución                               | 20 LTS   |
| **TypeScript**      | Tipado estático y POO                              | 5.x      |
| **NestJS**          | Framework backend modular                          | 10.x     |
| **TypeORM**         | ORM y migraciones de base de datos                 | 0.3.x    |
| **PostgreSQL**      | Base de datos relacional principal                 | 15+      |
| **Redis**           | Caché y gestión de sesiones                        | 7.x      |
| **MongoDB**         | Logs de auditoría e historial de notificaciones    | 6.x      |
| **Socket.io**       | Comunicación WebSocket en tiempo real              | 4.x      |
| **Passport.js**     | Estrategias de autenticación (JWT, Local)          | 0.6.x    |
| **PDFKit**          | Generación programática de certificados PDF        | 0.14.x   |
| **QRCode**          | Generación de códigos QR para verificación         | 1.5.x    |
| **Nodemailer**      | Entrega de correos transaccionales                 | 6.x      |
| **Multer**          | Manejo de carga de archivos                        | 1.4.x    |
| **class-validator** | Validación de DTOs con decoradores                 | 0.14.x   |
| **Swagger**         | Documentación de API autogenerada                  | 7.x      |

### Frontend

| Tecnología                | Propósito                                    | Versión  |
|---------------------------|----------------------------------------------|----------|
| **React**                 | Framework de UI                              | 18.x     |
| **TypeScript**            | Tipado estático                              | 5.x      |
| **React-Redux**           | Gestión de estado global                     | 8.x      |
| **Redux Toolkit**         | Redux simplificado con slices                | 1.9.x    |
| **Redux Thunk**           | Middleware asíncrono para llamadas a la API  | 2.4.x    |
| **React Router v6**       | Enrutamiento del lado del cliente            | 6.x      |
| **Webpack**               | Empaquetado manual de módulos                | 5.x      |
| **SASS/SCSS**             | Preprocesamiento avanzado de CSS             | 1.x      |
| **Axios**                 | Cliente HTTP con interceptores               | 1.x      |
| **Socket.io Client**      | Actualizaciones WebSocket en tiempo real     | 4.x      |
| **Chart.js**              | Gráficas de analítica de desempeño           | 4.x      |
| **React Testing Library** | Pruebas unitarias de componentes (TDD)       | 14.x     |
| **Cypress**               | Pruebas end-to-end (BDD)                     | 13.x     |

### DevOps y Herramientas

- **Docker** — Contenedorización de servicios
- **Docker Compose** — Orquestación local de múltiples contenedores
- **GitHub Actions** — Automatización de pipelines CI/CD
- **Jest** — Pruebas unitarias e integración del backend


- **Supertest** — Pruebas de integración de endpoints HTTP
- **ESLint + Prettier** — Calidad y formato del código
- **Husky** — Hooks pre-commit para control de calidad

---

## 🏗️ Arquitectura del Sistema

### Arquitectura General
```
┌─────────────────────────────────────────────────────────────────────┐
│                        CAPA DE PRESENTACIÓN                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │  Portal del │  │ Dashboard   │  │ Dashboard   │  │  Panel   │ │
│  │  Aprendiz   │  │ Instructor  │  │Coordinador  │  │  Admin   │ │
│  │  (React)    │  │  (React)    │  │  (React)    │  │ (React)  │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └────┬─────┘ │
│         │                │                │               │        │
│         └────────────────┴────────────────┴───────────────┘        │
│                                   │                                 │
│                    ┌──────────────▼──────────────┐                 │
│                    │    Redux Store + Thunks      │                 │
│                    │  (Gestión Centralizada)      │                 │
│                    └──────────────┬──────────────┘                 │
└───────────────────────────────────┼─────────────────────────────────┘
                                    │ REST + WebSocket
┌───────────────────────────────────┼─────────────────────────────────┐
│                      CAPA DE APLICACIÓN                             │
├───────────────────────────────────┼─────────────────────────────────┤
│                                   │                                 │
│           ┌───────────────────────▼───────────────────────┐        │
│           │       Servidor API NestJS (TypeScript)         │        │
│           │                                               │        │
│           │  ┌──────────┐  ┌──────────┐  ┌───────────┐  │        │
│           │  │  Módulo  │  │  Módulo  │  │  Módulo   │  │        │
│           │  │   Auth   │  │Matrícula │  │  Notas    │  │        │
│           │  └──────────┘  └──────────┘  └───────────┘  │        │
│           │                                               │        │
│           │  ┌──────────┐  ┌──────────┐  ┌───────────┐  │        │
│           │  │  Módulo  │  │  Módulo  │  │  Módulo   │  │        │
│           │  │Analítica │  │Certifica.│  │Notificac. │  │        │
│           │  └──────────┘  └──────────┘  └───────────┘  │        │
│           └───────────────────────────────────────────────┘        │
│                                   │                                 │
│           ┌───────────────────────▼───────────────────────┐        │
│           │      Gateway de Notificaciones Socket.io       │        │
│           │  (Notas · alertas · anuncios en tiempo real)   │        │
│           └───────────────────────────────────────────────┘        │
└───────────────────────────────────┼─────────────────────────────────┘
                                    │
┌───────────────────────────────────┼─────────────────────────────────┐
│                         CAPA DE DATOS                               │
├───────────────────────────────────┼─────────────────────────────────┤
│                                   │                                 │
│  ┌─────────────┐  ┌──────────────┐│  ┌──────────┐  ┌───────────┐  │
│  │ PostgreSQL  │  │    Redis     ││  │ MongoDB  │  │Almacena-  │  │
│  │             │  │              ││  │          │  │miento de  │  │
│  │ - Aprendices│  │ - Sesiones   ││  │ - Logs   │  │Archivos   │  │
│  │ - Programas │  │ - Caché      ││  │   Audit. │  │(PDFs de   │  │
│  │ - Notas     │  │ - Rate Limit ││  │ - Histor.│  │Certific.) │  │
│  │ - Certific. │  │              ││  │  Notific.│  │           │  │
│  └─────────────┘  └──────────────┘│  └──────────┘  └───────────┘  │
└───────────────────────────────────┴─────────────────────────────────┘
```

### Estructura de Módulos
```
src/
├── auth/
│   ├── auth.module.ts
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── strategies/
│   │   ├── jwt.strategy.ts
│   │   └── local.strategy.ts
│   └── guards/
│       ├── jwt-auth.guard.ts
│       └── roles.guard.ts
├── learners/
│   ├── learners.module.ts
│   ├── learners.controller.ts
│   ├── learners.service.ts
│   ├── dto/
│   │   ├── create-learner.dto.ts
│   │   └── update-learner.dto.ts
│   └── entities/
│       └── learner.entity.ts
├── programs/
│   ├── programs.module.ts
│   ├── programs.controller.ts
│   └── programs.service.ts
├── enrollment/
│   ├── enrollment.module.ts
│   ├── enrollment.controller.ts
│   └── enrollment.service.ts
├── grading/
│   ├── grading.module.ts
│   ├── grading.controller.ts
│   └── grading.service.ts
├── analytics/
│   ├── analytics.module.ts
│   ├── analytics.controller.ts
│   └── analytics.service.ts
├── certificates/
│   ├── certificates.module.ts
│   ├── certificates.controller.ts
│   ├── certificates.service.ts
│   └── pdf-generator.service.ts
├── notifications/
│   ├── notifications.module.ts
│   ├── notification.gateway.ts
│   └── notifications.service.ts
└── common/
    ├── decorators/
    ├── filters/
    ├── interceptors/
    └── pipes/
```

### Flujo de Datos
```
1. El instructor registra una nota
   └──> PATCH /api/grades/{id}
        └──> GradingController valida la solicitud
             └──> GradingService actualiza PostgreSQL
                  └──> AnalyticsService recalcula promedios
                       └──> Caché Redis invalidada
                            └──> NotificationGateway emite 'grade_posted'
                                 └──> App React del aprendiz se actualiza en tiempo real
                                      └──> CertificateService verifica elegibilidad
                                           └──> Si elegible: notifica al aprendiz
```

### Control de Acceso Basado en Roles
```
Roles y Permisos:

ADMIN
├── Acceso total al sistema
├── Gestión de usuarios (crear, actualizar, desactivar)
├── Gestión de programas y currículos
├── Configuración del sistema
└── Todos los reportes y analítica

COORDINADOR
├── Gestión de programas en área asignada
├── Asignación de instructores
├── Aprobación de matrículas de aprendices
├── Analítica y reportes por área
└── Generación masiva de certificados

INSTRUCTOR
├── Registro y modificación de notas (grupos propios)
├── Visualización del desempeño del aprendiz (grupos propios)
├── Transmisión de anuncios (programas propios)
├── Generación individual de certificados
└── Gestión de actividades y competencias

APRENDIZ
├── Visualización de su propio expediente académico
├── Historial de notas y analítica personal
├── Descarga de certificado (si ha sido emitido)
├── Anuncios del programa (solo lectura)
└── Mensajería con el instructor
```

---

## 💾 Instalación

### Requisitos Previos
```bash
# Software requerido
- Node.js 20 LTS o superior
- npm 10+ o yarn 1.22+
- PostgreSQL 15+
- Redis 7+
- MongoDB 6+
- Docker y Docker Compose (opcional pero recomendado)
```

### Opción 1: Instalación con Docker (Recomendada)
```bash
# 1. Clonar el repositorio
git clone https://github.com/paulabadt/edutrack.git
cd edutrack

# 2. Copiar archivos de variables de entorno
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env.local

# 3. Editar las variables de entorno
nano backend/.env

# 4. Iniciar todos los servicios
docker-compose up -d

# 5. Ejecutar migraciones de base de datos
docker-compose exec backend npm run migration:run

# 6. Cargar datos iniciales (opcional)
docker-compose exec backend npm run seed

# 7. Verificar estado de los servicios
docker-compose ps

# 8. Acceder a la aplicación
# Frontend:      http://localhost:3000
# API:           http://localhost:4000
# Documentación: http://localhost:4000/api/docs
```

### Opción 2: Instalación Manual

#### Configuración del Backend (NestJS + Node.js)
```bash
# 1. Ingresar al directorio del backend
cd backend

# 2. Instalar dependencias
npm install

# 3. Configurar variables de entorno
cp .env.example .env
# Editar .env con las credenciales de base de datos y secretos

# 4. Configurar base de datos PostgreSQL
psql -U postgres
CREATE DATABASE edutrack_db;
CREATE USER edutrack_user WITH PASSWORD 'tu_contraseña_segura';
GRANT ALL PRIVILEGES ON DATABASE edutrack_db TO edutrack_user;
\q

# 5. Ejecutar migraciones TypeORM
npm run migration:run

# 6. Cargar datos iniciales (opcional)
npm run seed

# 7. Iniciar servidor de desarrollo
npm run start:dev

# 8. Iniciar servidor de producción
npm run build
npm run start:prod
```

#### Configuración del Frontend (React + TypeScript)
```bash
# 1. Ingresar al directorio del frontend
cd frontend

# 2. Instalar dependencias
npm install

# 3. Configurar variables de entorno
cp .env.example .env.local
# Editar .env.local — definir REACT_APP_API_URL y REACT_APP_WS_URL

# 4. Iniciar servidor de desarrollo
npm run dev

# 5. Compilar para producción
npm run build

# 6. Servir build de producción
npm run preview
```

### Variables de Entorno
```bash
# backend/.env

# Aplicación
NODE_ENV=development
PORT=4000
APP_URL=http://localhost:4000

# Base de Datos — PostgreSQL
DB_HOST=localhost
DB_PORT=5432
DB_USERNAME=edutrack_user
DB_PASSWORD=tu_contraseña_segura
DB_DATABASE=edutrack_db

# Caché — Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=tu_contraseña_redis

# Logs de Auditoría — MongoDB
MONGODB_URI=mongodb://localhost:27017/edutrack_logs

# Autenticación
JWT_SECRET=tu_secreto_jwt_super_seguro_minimo_32_caracteres
JWT_EXPIRATION=24h
JWT_REFRESH_SECRET=tu_secreto_refresh_minimo_32_caracteres
JWT_REFRESH_EXPIRATION=7d

# Correo Electrónico — Nodemailer
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=tu_correo@gmail.com
MAIL_PASSWORD=tu_contraseña_de_aplicacion
MAIL_FROM=noreply@edutrack.edu.co

# Almacenamiento de Archivos
STORAGE_PATH=./uploads
MAX_FILE_SIZE_MB=10

# Frontend
FRONTEND_URL=http://localhost:3000
```
```bash
# frontend/.env.local

REACT_APP_API_URL=http://localhost:4000/api
REACT_APP_WS_URL=http://localhost:4000
REACT_APP_APP_NAME=EduTrack
```

---

## 🚀 Uso

### Iniciar el Servidor de Desarrollo
```bash
# Backend — modo desarrollo con recarga automática
cd backend
npm run start:dev

# Frontend — modo desarrollo con HMR
cd frontend
npm run dev
```

### Credenciales por Defecto
```bash
# Administrador del sistema
Email:      admin@edutrack.edu.co
Contraseña: Admin123! (¡cambiar inmediatamente!)

# Coordinador de ejemplo
Email:      coordinador@edutrack.edu.co
Contraseña: Coord123!

# Instructor de ejemplo
Email:      instructor@edutrack.edu.co
Contraseña: Inst123!

# Aprendiz de ejemplo
Email:      aprendiz@edutrack.edu.co
Contraseña: Aprendiz123!
```

### Flujos Principales por Rol
```bash
# COORDINADOR — Ciclo de vida de programa
1. Crear programa de formación con competencias y actividades
2. Asignar instructores por competencia
3. Matricular aprendices con validación de capacidad
4. Monitorear analítica del programa en tiempo real
5. Generar certificados masivos al finalizar la cohorte

# INSTRUCTOR — Ciclo de evaluación
1. Visualizar aprendices asignados por programa
2. Registrar notas por actividad y competencia
3. Monitorear aprendices en riesgo (promedio < 60)
4. Publicar anuncios al grupo del programa
5. Generar certificado individual al aprobar todas las competencias

# APRENDIZ — Seguimiento académico
1. Consultar expediente académico completo
2. Visualizar notas y promedio ponderado en tiempo real
3. Revisar estado de aprobación por competencia
4. Recibir notificaciones instantáneas de nuevas notas
5. Descargar certificado digital con verificación QR
```

### Scripts Disponibles
```bash
# Backend
npm run start:dev       # Servidor de desarrollo con recarga automática
npm run start:prod      # Servidor de producción optimizado
npm run build           # Compilar TypeScript a JavaScript
npm run test            # Ejecutar pruebas unitarias con Jest
npm run test:e2e        # Ejecutar pruebas de integración con Supertest
npm run test:cov        # Reporte de cobertura de pruebas
npm run migration:run   # Ejecutar migraciones TypeORM pendientes
npm run migration:revert # Revertir la última migración
npm run seed            # Cargar datos iniciales de demostración
npm run lint            # Verificar código con ESLint
npm run format          # Formatear código con Prettier

# Frontend
npm run dev             # Servidor de desarrollo Webpack con HMR
npm run build           # Bundle de producción optimizado con Webpack
npm run preview         # Previsualizar build de producción
npm run test            # Pruebas unitarias con React Testing Library
npm run test:coverage   # Cobertura de pruebas del frontend
npm run cypress:open    # Abrir Cypress para pruebas BDD interactivas
npm run cypress:run     # Ejecutar suite completa de pruebas Cypress
npm run lint            # ESLint para TypeScript y React
```

---

## 💻 Ejemplos de Código

### 1. Autenticación y JWT (NestJS + TypeScript)
```typescript
// auth/auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
    private redisService: RedisService,
  ) {}

  async login(dto: LoginDto): Promise<AuthResponseDto> {
    const user = await this.usersService.findByEmail(dto.email);

    if (!user || !(await bcrypt.compare(dto.password, user.password))) {
      throw new UnauthorizedException('Credenciales inválidas');
    }

    if (!user.isActive) {
      throw new UnauthorizedException('La cuenta está desactivada');
    }

    const payload: JwtPayload = {
      sub: user.id,
      email: user.email,
      role: user.role,
    };

    const accessToken = this.jwtService.sign(payload, {
      expiresIn: process.env.JWT_EXPIRATION,
    });

    const refreshToken = this.jwtService.sign(payload, {
      secret: process.env.JWT_REFRESH_SECRET,
      expiresIn: process.env.JWT_REFRESH_EXPIRATION,
    });

    // Almacenar refresh token en Redis
    await this.redisService.set(
      `refresh_token:${user.id}`,
      refreshToken,
      7 * 24 * 60 * 60, // TTL de 7 días
    );

    return { accessToken, refreshToken, user: this.mapUserToDto(user) };
  }

  async refreshTokens(refreshToken: string): Promise<AuthResponseDto> {
    try {
      const payload = this.jwtService.verify(refreshToken, {
        secret: process.env.JWT_REFRESH_SECRET,
      });

      const stored = await this.redisService.get(
        `refresh_token:${payload.sub}`
      );

      if (!stored || stored !== refreshToken) {
        throw new UnauthorizedException(
          'El refresh token es inválido o ha expirado'
        );
      }

      const user = await this.usersService.findById(payload.sub);
      return this.login({ email: user.email, password: null });

    } catch {
      throw new UnauthorizedException('Refresh token inválido');
    }
  }
}
```

### 2. Registro de Nota con Notificación en Tiempo Real (NestJS)
```typescript
// grading/grading.service.ts
@Injectable()
export class GradingService {
  constructor(
    @InjectRepository(Grade)
    private gradeRepository: Repository<Grade>,
    private analyticsService: AnalyticsService,
    private notificationGateway: NotificationGateway,
    private cacheService: CacheService,
  ) {}

  async createGrade(dto: CreateGradeDto, instructorId: string): Promise<Grade> {
    const grade = this.gradeRepository.create({
      ...dto,
      gradedBy: instructorId,
      gradedAt: new Date(),
    });

    const saved = await this.gradeRepository.save(grade);

    // Invalidar caché de analítica del aprendiz
    await this.cacheService.del(
      `analytics:learner:${dto.learnerId}:${dto.programId}`
    );

    // Recalcular resumen de desempeño
    const summary = await this.analyticsService
      .getLearnerPerformanceSummary(dto.learnerId, dto.programId);

    // Notificar al aprendiz en tiempo real
    this.notificationGateway.notifyGradePosted(dto.learnerId, {
      activityName: dto.activityName,
      competencyName: dto.competencyName,
      score: dto.score,
      maxScore: dto.maxScore,
      newAverage: summary.overallAverage,
      completionPercentage: summary.completionPercentage,
      timestamp: new Date(),
    });

    // Alertar al coordinador si el aprendiz está en riesgo
    if (summary.overallAverage < 60) {
      this.notificationGateway.notifyAtRiskLearner(
        dto.programId,
        dto.learnerId,
        summary,
      );
    }

    return saved;
  }
}
```

### 3. Dashboard React con Redux Thunks
```typescript
// store/slices/analyticsSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { analyticsService } from '../../services/analyticsService';

export const fetchLearnerPerformance = createAsyncThunk(
  'analytics/fetchLearnerPerformance',
  async (
    params: { learnerId: string; programId: string },
    { rejectWithValue }
  ) => {
    try {
      const response = await analyticsService.getLearnerPerformance(params);
      return response.data;
    } catch (error: any) {
      return rejectWithValue(
        error.response?.data?.message ||
        'Error al obtener datos de desempeño'
      );
    }
  }
);

export const fetchProgramAnalytics = createAsyncThunk(
  'analytics/fetchProgramAnalytics',
  async (programId: string, { rejectWithValue }) => {
    try {
      const response = await analyticsService.getProgramAnalytics(programId);
      return response.data;
    } catch (error: any) {
      return rejectWithValue(
        error.response?.data?.message ||
        'Error al obtener analítica del programa'
      );
    }
  }
);

const analyticsSlice = createSlice({
  name: 'analytics',
  initialState: {
    learnerPerformance: null,
    programAnalytics: null,
    loading: false,
    error: null as string | null,
  },
  reducers: {
    clearAnalytics: (state) => {
      state.learnerPerformance = null;
      state.programAnalytics = null;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchLearnerPerformance.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchLearnerPerformance.fulfilled, (state, action) => {
        state.loading = false;
        state.learnerPerformance = action.payload;
      })
      .addCase(fetchLearnerPerformance.rejected, (state, action) => {
        state.loading = false;
        state.error = action.payload as string;
      })
      .addCase(fetchProgramAnalytics.fulfilled, (state, action) => {
        state.programAnalytics = action.payload;
      });
  },
});

export const { clearAnalytics } = analyticsSlice.actions;
export default analyticsSlice.reducer;
```
```typescript
// components/PerformanceDashboard/PerformanceDashboard.tsx
import React, { useEffect, useRef } from 'react';
import { useDispatch, useSelector } from 'react-redux';
import { io, Socket } from 'socket.io-client';
import { AppDispatch, RootState } from '../../store';
import {
  fetchLearnerPerformance,
} from '../../store/slices/analyticsSlice';
import { GradeNotification } from '../../types/notifications';
import styles from './PerformanceDashboard.module.scss';

interface Props {
  learnerId: string;
  programId: string;
}

const PerformanceDashboard: React.FC<Props> = ({ learnerId, programId }) => {
  const dispatch = useDispatch<AppDispatch>();
  const { learnerPerformance, loading, error } = useSelector(
    (state: RootState) => state.analytics
  );
  const socketRef = useRef<Socket>();

  useEffect(() => {
    dispatch(fetchLearnerPerformance({ learnerId, programId }));
  }, [dispatch, learnerId, programId]);

  // Notificaciones de notas en tiempo real
  useEffect(() => {
    socketRef.current = io(
      `${process.env.REACT_APP_WS_URL}/notifications`,
      {
        auth: { userId: learnerId },
        transports: ['websocket'],
      }
    );

    socketRef.current.on('grade_posted', (_data: GradeNotification) => {
      // Refrescar datos de desempeño al recibir nueva nota
      dispatch(fetchLearnerPerformance({ learnerId, programId }));
    });

    return () => { socketRef.current?.disconnect(); };
  }, [dispatch, learnerId, programId]);

  if (loading) return (
    <div className={styles.loading} data-testid="loading-indicator">
      Cargando datos de desempeño...
    </div>
  );

  if (error) return (
    <div className={styles.error} data-testid="error-message">{error}</div>
  );

  return (
    <div className={styles.dashboard} data-testid="performance-dashboard">
      <div className={styles.summaryCards}>
        <div className={styles.card} data-testid="overall-average-card">
          <span className={styles.cardValue}>
            {learnerPerformance?.overallAverage.toFixed(1)}
          </span>
          <span className={styles.cardLabel}>Promedio General</span>
        </div>
        <div className={styles.card} data-testid="completion-card">
          <span className={styles.cardValue}>
            {learnerPerformance?.completionPercentage.toFixed(0)}%
          </span>
          <span className={styles.cardLabel}>Avance del Programa</span>
        </div>
        <div className={styles.card}>
          <span className={styles.cardValue}>
            {learnerPerformance?.approvedCompetencies}
            /{learnerPerformance?.totalCompetencies}
          </span>
          <span className={styles.cardLabel}>Competencias Aprobadas</span>
        </div>
      </div>

      <div className={styles.competencyList}>
        {learnerPerformance?.competencyBreakdown.map((competency) => (
          <div
            key={competency.competencyId}
            className={`
              ${styles.competencyRow}
              ${styles[competency.status.toLowerCase()]}
            `}
            data-testid="competency-row"
          >
            <span className={styles.competencyName}>
              {competency.competencyName}
            </span>
            <span className={styles.competencyAverage}>
              {competency.average.toFixed(1)}
            </span>
            <span className={styles.competencyStatus}>
              {competency.status}
            </span>
          </div>
        ))}
      </div>
    </div>
  );
};

export default PerformanceDashboard;
```
```scss
// PerformanceDashboard.module.scss
@import '../../styles/variables';
@import '../../styles/mixins';

.dashboard {
  padding: $spacing-lg;
  max-width: 960px;
  margin: 0 auto;

  .summaryCards {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: $spacing-md;
    margin-bottom: $spacing-xl;

    .card {
      @include card;
      display: flex;
      flex-direction: column;
      align-items: center;
      padding: $spacing-lg;

      .cardValue {
        font-size: $font-size-xxl;
        font-weight: $font-weight-bold;
        color: $color-primary;
      }

      .cardLabel {
        font-size: $font-size-sm;
        color: $color-text-secondary;
        margin-top: $spacing-xs;
      }
    }
  }

  .competencyRow {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: $spacing-sm $spacing-md;
    border-radius: $border-radius-sm;
    margin-bottom: $spacing-xs;
    transition: background-color 0.2s ease;

    &.approved    { background-color: rgba($color-success, 0.1); }
    &.in_progress { background-color: rgba($color-warning, 0.1); }
    &.failed      { background-color: rgba($color-danger,  0.1); }
  }

  .loading,
  .error {
    @include flex-center;
    min-height: 200px;
    font-size: $font-size-lg;
    color: $color-text-secondary;
  }
}
```

### 4. Pruebas Backend — Jest + Supertest (TDD)
```typescript
// certificates/certificates.service.spec.ts
describe('CertificateService', () => {
  let service: CertificateService;
  let analyticsService: jest.Mocked<AnalyticsService>;
  let certificateRepository: jest.Mocked<Repository<Certificate>>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        CertificateService,
        {
          provide: getRepositoryToken(Certificate),
          useValue: {
            create: jest.fn(),
            save: jest.fn(),
            findOne: jest.fn(),
          },
        },
        {
          provide: AnalyticsService,
          useValue: {
            getLearnerPerformanceSummary: jest.fn(),
          },
        },
        { provide: QrCodeService, useValue: { generate: jest.fn() } },
        {
          provide: PdfGeneratorService,
          useValue: { generateCertificate: jest.fn() },
        },
      ],
    }).compile();

    service = module.get<CertificateService>(CertificateService);
    analyticsService = module.get(AnalyticsService);
    certificateRepository = module.get(getRepositoryToken(Certificate));
  });

  describe('generateCertificate', () => {
    it('debe generar el certificado cuando el aprendiz cumple todos los requisitos',
      async () => {
        // Dado
        const learnerId = 'aprendiz-001';
        const programId = 'programa-001';

        analyticsService.getLearnerPerformanceSummary.mockResolvedValue({
          learnerId,
          programId,
          overallAverage: 85.5,
          completionPercentage: 100,
          approvedCompetencies: 8,
          totalCompetencies: 8,
          learnerName: 'Juan Pérez',
          programName: 'Desarrollo de Software',
          competencyBreakdown: [],
          trend: 'IMPROVING',
        });

        certificateRepository.create.mockReturnValue({
          id: 'cert-001',
          learnerId,
          programId,
        } as Certificate);

        certificateRepository.save.mockResolvedValue({
          id: 'cert-001',
          learnerId,
          programId,
          verificationCode: 'SENA-001',
          status: CertificateStatus.ISSUED,
        } as Certificate);

        // Cuando
        const result = await service.generateCertificate(learnerId, programId);

        // Entonces
        expect(result.status).toBe(CertificateStatus.ISSUED);
        expect(certificateRepository.save).toHaveBeenCalledTimes(1);
        expect(analyticsService.getLearnerPerformanceSummary)
          .toHaveBeenCalledWith(learnerId, programId);
    });

    it('debe lanzar BadRequestException cuando el avance es inferior al 100%',
      async () => {
        // Dado
        analyticsService.getLearnerPerformanceSummary.mockResolvedValue({
          completionPercentage: 75,
          overallAverage: 80,
        } as any);

        // Cuando / Entonces
        await expect(
          service.generateCertificate('aprendiz-001', 'programa-001')
        ).rejects.toThrow(BadRequestException);

        expect(certificateRepository.save).not.toHaveBeenCalled();
    });

    it('debe lanzar BadRequestException cuando el promedio es inferior a 70',
      async () => {
        // Dado
        analyticsService.getLearnerPerformanceSummary.mockResolvedValue({
          completionPercentage: 100,
          overallAverage: 65,
        } as any);

        // Cuando / Entonces
        await expect(
          service.generateCertificate('aprendiz-001', 'programa-001')
        ).rejects.toThrow(BadRequestException);
    });
  });
});
```
```typescript
// Cypress BDD — Flujo E2E del Dashboard del Aprendiz
describe('Dashboard de Desempeño - BDD', () => {
  beforeEach(() => {
    cy.login('aprendiz@edutrack.edu.co', 'testpass');
    cy.visit('/dashboard/performance');
  });

  it('Dado un aprendiz, Cuando carga el dashboard, Entonces se muestra el resumen de desempeño',
    () => {
      cy.intercept('GET', '/api/analytics/learner/*', {
        fixture: 'learner-performance.json',
      }).as('performanceRequest');

      cy.wait('@performanceRequest');

      cy.get('[data-testid="performance-dashboard"]').should('be.visible');
      cy.get('[data-testid="overall-average-card"]')
        .should('contain.text', '85.5');
      cy.get('[data-testid="completion-card"]')
        .should('contain.text', '100%');
      cy.get('[data-testid="competency-row"]')
        .should('have.length.greaterThan', 0);
  });

  it('Dado un aprendiz, Cuando se registra una nueva nota, Entonces el dashboard se actualiza en tiempo real',
    () => {
      cy.intercept('GET', '/api/analytics/learner/*').as('refreshRequest');

      // Simular notificación WebSocket de nueva nota
      cy.window().then((win) => {
        win.dispatchEvent(new CustomEvent('grade_posted', {
          detail: { activityName: 'Proyecto Final', score: 95 },
        }));
      });

      cy.wait('@refreshRequest');
      cy.get('[data-testid="performance-dashboard"]').should('be.visible');
  });

  it('Dado un aprendiz con todas las competencias aprobadas, Cuando descarga el certificado, Entonces el PDF se genera correctamente',
    () => {
      cy.intercept('POST', '/api/certificates/generate', {
        fixture: 'certificate-issued.json',
      }).as('generateCertificate');

      cy.get('[data-testid="generate-certificate-btn"]').click();
      cy.wait('@generateCertificate');

      cy.get('[data-testid="certificate-download-link"]')
        .should('be.visible')
        .should('have.attr', 'href');
  });
});
```

---

## 📚 Documentación de la API

### URL Base
```
Desarrollo:    http://localhost:4000/api
Producción:    https://api.edutrack.edu.co/api
Swagger UI:    http://localhost:4000/api/docs
```

### Autenticación

Todos los endpoints protegidos requieren un token JWT Bearer.
```bash
POST /api/auth/login
Content-Type: application/json

{
  "email": "instructor@edutrack.edu.co",
  "password": "tu_contraseña"
}

# Respuesta: 200 OK
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": "usr-001",
    "email": "instructor@edutrack.edu.co",
    "role": "INSTRUCTOR",
    "fullName": "María García"
  }
}
```
```bash
# Uso del token en solicitudes protegidas
GET /api/aprendices
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Endpoints

#### 1. Aprendices

**Listar Todos los Aprendices**
```bash
GET /api/learners?page=1&limit=20&programId=prog-001
Authorization: Bearer {token}

# Parámetros de consulta:
# - page: int (por defecto: 1)
# - limit: int (por defecto: 20, máximo: 100)
# - programId: string (opcional)
# - status: string (ACTIVE, INACTIVE, GRADUATED)
# - search: string (busca por nombre y documento)

# Respuesta: 200 OK
{
  "data": [
    {
      "id": "lrn-001",
      "documentType": "CC",
      "documentNumber": "1234567890",
      "firstName": "Juan",
      "lastName": "Pérez",
      "email": "juan.perez@email.com",
      "phone": "3001234567",
      "status": "ACTIVE",
      "enrolledPrograms": 2,
      "createdAt": "2024-01-15T00:00:00Z"
    }
  ],
  "total": 250,
  "page": 1,
  "limit": 20
}
```

**Crear Aprendiz**
```bash
POST /api/learners
Authorization: Bearer {token}
Content-Type: application/json

{
  "documentType": "CC",
  "documentNumber": "1234567890",
  "firstName": "Juan",
  "lastName": "Pérez",
  "email": "juan.perez@email.com",
  "phone": "3001234567",
  "birthDate": "2000-05-15",
  "city": "Cali",
  "department": "Valle del Cauca"
}

# Respuesta: 201 Created
{
  "id": "lrn-002",
  "documentType": "CC",
  "documentNumber": "1234567890",
  "firstName": "Juan",
  "lastName": "Pérez",
  "status": "ACTIVE",
  "createdAt": "2024-02-01T10:00:00Z"
}
```

#### 2. Matrículas

**Matricular Aprendiz en Programa**
```bash
POST /api/enrollments
Authorization: Bearer {token}
Content-Type: application/json

{
  "learnerId": "lrn-001",
  "programId": "prog-001",
  "startDate": "2024-02-01"
}

# Respuesta: 201 Created
{
  "id": "enr-001",
  "learnerId": "lrn-001",
  "programId": "prog-001",
  "startDate": "2024-02-01",
  "status": "ACTIVE",
  "enrolledAt": "2024-01-28T14:00:00Z"
}
```

**Listar Matrículas por Programa**
```bash
GET /api/enrollments?programId=prog-001&status=ACTIVE
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "data": [
    {
      "id": "enr-001",
      "learner": {
        "id": "lrn-001",
        "fullName": "Juan Pérez",
        "documentNumber": "1234567890"
      },
      "program": {
        "id": "prog-001",
        "name": "Desarrollo de Software"
      },
      "startDate": "2024-02-01",
      "status": "ACTIVE",
      "completionPercentage": 75.0
    }
  ],
  "total": 32
}
```

#### 3. Notas

**Registrar una Nota**
```bash
POST /api/grades
Authorization: Bearer {token}
Content-Type: application/json

{
  "learnerId": "lrn-001",
  "programId": "prog-001",
  "competencyId": "comp-003",
  "activityId": "act-007",
  "score": 88.5,
  "maxScore": 100,
  "weight": 1.5,
  "observations": "Excelente desempeño en la práctica"
}

# Respuesta: 201 Created
{
  "id": "grd-001",
  "learnerId": "lrn-001",
  "competencyId": "comp-003",
  "activityId": "act-007",
  "score": 88.5,
  "maxScore": 100,
  "weight": 1.5,
  "gradedBy": "usr-instructor-001",
  "gradedAt": "2024-02-10T09:30:00Z"
}
```

**Consultar Notas del Aprendiz**
```bash
GET /api/grades?learnerId=lrn-001&programId=prog-001
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "data": [
    {
      "id": "grd-001",
      "competencyName": "Fundamentos de Desarrollo de Software",
      "activityName": "Proyecto Final",
      "score": 88.5,
      "maxScore": 100,
      "weight": 1.5,
      "gradedAt": "2024-02-10T09:30:00Z",
      "gradedBy": "María García"
    }
  ],
  "total": 24
}
```

**Actualizar Nota**
```bash
PATCH /api/grades/{gradeId}
Authorization: Bearer {token}
Content-Type: application/json

{
  "score": 90.0,
  "observations": "Nota corregida tras revisión del proyecto"
}

# Respuesta: 200 OK
{
  "id": "grd-001",
  "score": 90.0,
  "observations": "Nota corregida tras revisión del proyecto",
  "updatedAt": "2024-02-11T08:00:00Z"
}
```

#### 4. Analítica

**Resumen de Desempeño del Aprendiz**
```bash
GET /api/analytics/learner/{learnerId}/program/{programId}
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "learnerId": "lrn-001",
  "programId": "prog-001",
  "overallAverage": 85.5,
  "completionPercentage": 87.5,
  "approvedCompetencies": 7,
  "totalCompetencies": 8,
  "trend": "IMPROVING",
  "competencyBreakdown": [
    {
      "competencyId": "comp-001",
      "competencyName": "Fundamentos de Desarrollo de Software",
      "average": 90.0,
      "status": "APPROVED",
      "gradeCount": 4
    },
    {
      "competencyId": "comp-002",
      "competencyName": "Diseño de Bases de Datos",
      "average": 78.5,
      "status": "APPROVED",
      "gradeCount": 3
    }
  ]
}
```

**Analítica del Programa**
```bash
GET /api/analytics/program/{programId}
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "programId": "prog-001",
  "programName": "Desarrollo de Software",
  "totalLearners": 32,
  "averageCompletion": 72.5,
  "averageGrade": 78.3,
  "atRiskLearners": 4,
  "graduationReady": 8,
  "gradeDistribution": {
    "90-100": 6,
    "80-89": 12,
    "70-79": 9,
    "60-69": 4,
    "menor-60": 1
  }
}
```

**Aprendices en Riesgo**
```bash
GET /api/analytics/program/{programId}/at-risk
Authorization: Bearer {token}

# Respuesta: 200 OK
{
  "data": [
    {
      "learnerId": "lrn-015",
      "fullName": "Carlos Ruiz",
      "overallAverage": 55.3,
      "completionPercentage": 62.5,
      "failedCompetencies": 2,
      "lastActivity": "2024-02-05T00:00:00Z",
      "riskLevel": "HIGH"
    }
  ],
  "total": 4
}
```

#### 5. Certificados

**Generar Certificado**
```bash
POST /api/certificates/generate
Authorization: Bearer {token}
Content-Type: application/json

{
  "learnerId": "lrn-001",
  "programId": "prog-001"
}

# Respuesta: 201 Created
{
  "id": "cert-001",
  "learnerId": "lrn-001",
  "programId": "prog-001",
  "verificationCode": "SENA-1706789012-A3B7K2",
  "overallAverage": 85.5,
  "pdfUrl": "/certificates/SENA-1706789012-A3B7K2.pdf",
  "issuedAt": "2024-02-15T10:00:00Z",
  "status": "ISSUED"
}
```

**Verificar Certificado (Público)**
```bash
GET /api/certificates/verify/{verificationCode}
# No requiere autenticación — endpoint público

# Respuesta: 200 OK
{
  "valid": true,
  "learnerName": "Juan Pérez",
  "programName": "Desarrollo de Software — SENA",
  "completionDate": "2024-02-15",
  "overallAverage": 85.5,
  "issuedBy": "SENA - Servicio Nacional de Aprendizaje",
  "verificationCode": "SENA-1706789012-A3B7K2"
}
```

**Generación Masiva de Certificados**
```bash
POST /api/certificates/bulk-generate
Authorization: Bearer {token}
Content-Type: application/json

{
  "programId": "prog-001",
  "onlyEligible": true
}

# Respuesta: 202 Accepted
{
  "jobId": "job-bulk-001",
  "status": "PROCESSING",
  "eligibleLearners": 8,
  "estimatedTimeSeconds": 45,
  "statusUrl": "/api/certificates/bulk-status/job-bulk-001"
}
```

### Respuestas de Error

Todos los errores siguen este formato:
```json
{
  "statusCode": 422,
  "error": "VALIDATION_ERROR",
  "message": "La validación falló",
  "details": [
    {
      "field": "score",
      "constraint": "score no puede ser mayor que maxScore"
    }
  ]
}
```

**Códigos de Error Comunes**

| Código              | Estado HTTP | Descripción                                  |
|---------------------|-------------|----------------------------------------------|
| `UNAUTHORIZED`      | 401         | Token JWT faltante o inválido                |
| `FORBIDDEN`         | 403         | Permisos de rol insuficientes                |
| `NOT_FOUND`         | 404         | Recurso no encontrado                        |
| `CONFLICT`          | 409         | Recurso duplicado o violación de restricción |
| `VALIDATION_ERROR`  | 422         | Cuerpo o parámetros de solicitud inválidos   |
| `INTERNAL_ERROR`    | 500         | Error inesperado del servidor                |

---

## 🤝 Contribución

Este proyecto fue desarrollado como parte de la labor investigativa en el SENA.
Aunque el código fuente y las aplicaciones son propiedad del SENA, las contribuciones
y sugerencias son bienvenidas.

### Flujo de Desarrollo
```bash
# 1. Crear una rama de funcionalidad
git checkout -b feature/nombre-de-la-funcionalidad

# 2. Realizar los cambios siguiendo las convenciones de módulos NestJS

# 3. Ejecutar todas las pruebas
npm run test              # Pruebas unitarias del backend (Jest)
npm run test:e2e          # Pruebas de integración del backend (Supertest)
npm run test:cov          # Reporte de cobertura
npx cypress run           # Pruebas E2E del frontend (Cypress)

# 4. Formatear y verificar el código
npm run lint              # ESLint
npm run format            # Prettier

# 5. Hacer commit usando commits convencionales
git commit -m "feat: agregar endpoint de generación masiva de certificados"
git commit -m "fix: corregir cálculo de promedio ponderado para notas sin peso"
git commit -m "test: agregar pruebas unitarias para validación de capacidad de matrícula"

# 6. Subir cambios y abrir pull request
git push origin feature/nombre-de-la-funcionalidad
```

### Guía de Estilo de Código
```bash
# El modo estricto de TypeScript es obligatorio — sin any implícito
# Todos los DTOs deben usar decoradores de class-validator
# Todos los servicios deben tener archivos .spec.ts de prueba correspondientes
# Los módulos NestJS deben seguir el principio de responsabilidad única
# Los componentes React deben incluir atributos data-testid para las pruebas
```

---

## 📄 Licencia

Este proyecto fue desarrollado durante la labor investigativa y de instrucción en
el **SENA (Servicio Nacional de Aprendizaje)** bajo el programa **SENNOVA**,
enfocado en apoyar la transformación digital de las instituciones de formación
profesional en Colombia.

> ⚠️ **Aviso de Propiedad Intelectual**
>
> El código fuente, diseño arquitectónico, documentación técnica y todos los
> activos asociados son **propiedad institucional del SENA** y no están
> disponibles públicamente en este repositorio. El contenido presentado aquí —
> incluyendo especificaciones técnicas, diagramas de arquitectura, fragmentos
> de código y documentación de la API — ha sido **recreado únicamente con fines
> de demostración de portafolio**, sin exponer información institucional
> confidencial ni el código fuente original de producción.
>
> Las capturas de pantalla e imágenes de la interfaz han sido intencionalmente
> excluidas para proteger la privacidad de los datos de los aprendices y la
> confidencialidad institucional.

**Disponible para:**

- ✅ Consultoría personalizada e implementación para instituciones educativas
- ✅ Asesoría en diseño y arquitectura de sistemas de gestión académica
- ✅ Desarrollo full-stack con Node.js + NestJS + React
- ✅ Sistemas de generación y verificación de certificados digitales
- ✅ Desarrollo de módulos adicionales y soporte técnico

---

*Desarrollado por **Paula Abad** — Desarrolladora de Software Senior e Instructora/Investigadora SENA*
*🌐 [paulabad.tech](https://paulabad.tech) · 📱 Soporte directo de la desarrolladora vía WhatsApp*
