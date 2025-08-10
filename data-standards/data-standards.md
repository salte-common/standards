# Data Standards

## Overview

This document outlines comprehensive data standards for applications following the [Twelve-Factor App methodology](https://12factor.net/). These standards ensure proper data management, governance, and protection while supporting scalable, reliable, and secure data operations.

## Core Data Principles

### 1. Data as a Service
- **Backing Services**: Treat databases as attached resources
- **Configuration-Based**: Connect to data stores via configuration
- **Service Discovery**: Use service discovery for data store location
- **Fallback Strategies**: Implement graceful degradation for data services

### 2. Data Governance
- **Data Classification**: Classify data by sensitivity and importance
- **Data Lineage**: Track data flow and transformations
- **Data Quality**: Ensure data accuracy, completeness, and consistency
- **Data Lifecycle**: Manage data from creation to deletion

### 3. Data Protection
- **Encryption**: Encrypt data at rest and in transit
- **Access Control**: Implement role-based data access
- **Audit Trails**: Comprehensive logging of data access
- **Compliance**: Meet regulatory data protection requirements

## Twelve-Factor Data Standards

### Factor IV: Backing Services - Treat backing services as attached resources

#### Database as a Service
- **Configuration-Based**: Connect to databases via environment variables
- **Service Abstraction**: Treat databases as external services
- **Failover Support**: Implement database failover strategies
- **Connection Pooling**: Use connection pooling for efficiency

#### Implementation Standards
```javascript
// Database connection configuration
const databaseConfig = {
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  database: process.env.DB_NAME,
  username: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  ssl: process.env.NODE_ENV === 'production',
  pool: {
    min: 2,
    max: 10,
    acquireTimeoutMillis: 30000,
    createTimeoutMillis: 30000,
    destroyTimeoutMillis: 5000,
    idleTimeoutMillis: 30000,
    reapIntervalMillis: 1000,
    createRetryIntervalMillis: 100
  }
};

// Database connection with health check
const createDatabaseConnection = async () => {
  try {
    const connection = await db.connect(databaseConfig);
    
    // Health check
    await connection.query('SELECT 1');
    
    logger.info('Database connection established');
    return connection;
  } catch (error) {
    logger.error('Database connection failed', { error: error.message });
    throw error;
  }
};
```

### Factor VI: Processes - Execute the app as one or more stateless processes

#### Stateless Data Access
- **No Local State**: Processes should not store data locally
- **External Storage**: Store all data in external data stores
- **Session Management**: Store sessions in external services
- **Cache Management**: Use external caching services

#### Implementation Examples
```javascript
// Stateless data access patterns
const dataAccess = {
  // User session storage in Redis
  storeSession: async (sessionId, userData) => {
    await redis.setex(`session:${sessionId}`, 3600, JSON.stringify(userData));
  },
  
  // User data storage in database
  storeUserData: async (userId, data) => {
    await db.query(
      'INSERT INTO users (id, data, created_at) VALUES (?, ?, NOW())',
      [userId, JSON.stringify(data)]
    );
  },
  
  // File storage in cloud storage
  storeFile: async (fileId, fileData) => {
    await s3.upload({
      Bucket: process.env.S3_BUCKET,
      Key: `files/${fileId}`,
      Body: fileData
    }).promise();
  }
};
```

### Factor XI: Logs - Treat logs as event streams

#### Data Event Logging
> **Note**: For comprehensive logging standards and implementation, see [Operations Standards](./../operations-standards/operations-standards.md).

- **Structured Logging**: Log data events in structured format
- **Audit Trails**: Comprehensive audit trails for data access
- **Data Lineage**: Track data transformations and movements
- **Compliance Logging**: Log events for regulatory compliance

#### Data Event Logging Implementation
```javascript
// Data event logging
const logDataEvent = (event) => {
  logger.info('Data event', {
    event: event.type,
    userId: event.userId,
    dataId: event.dataId,
    action: event.action,
    timestamp: new Date().toISOString(),
    correlationId: event.correlationId,
    metadata: event.metadata
  });
};

// Data access logging middleware
const logDataAccess = (req, res, next) => {
  const originalSend = res.send;
  
  res.send = function(data) {
    logDataEvent({
      type: 'data_access',
      userId: req.user?.id,
      dataId: req.params.id,
      action: req.method,
      correlationId: req.correlationId,
      metadata: {
        endpoint: req.path,
        userAgent: req.get('User-Agent'),
        ip: req.ip
      }
    });
    
    originalSend.call(this, data);
  };
  
  next();
};
```

## Data Classification Standards

### Data Sensitivity Levels
- **Public**: Information that can be freely shared
- **Internal**: Information for internal use only
- **Confidential**: Sensitive business information
- **Restricted**: Highly sensitive information (PII, PHI, etc.)

### Data Classification Implementation
```javascript
// Data classification system
const dataClassification = {
  public: {
    level: 1,
    encryption: false,
    access: 'public',
    retention: 'indefinite',
    audit: false
  },
  internal: {
    level: 2,
    encryption: true,
    access: 'authenticated',
    retention: '5 years',
    audit: true
  },
  confidential: {
    level: 3,
    encryption: true,
    access: 'authorized',
    retention: '10 years',
    audit: true,
    masking: true
  },
  restricted: {
    level: 4,
    encryption: true,
    access: 'need-to-know',
    retention: '7 years',
    audit: true,
    masking: true,
    anonymization: true
  }
};

// Data classification helper
const classifyData = (data, context) => {
  // PII detection
  if (containsPII(data)) {
    return 'restricted';
  }
  
  // Business data classification
  if (isBusinessCritical(data)) {
    return 'confidential';
  }
  
  // Internal data
  if (isInternalOnly(data)) {
    return 'internal';
  }
  
  return 'public';
};
```

## Data Quality Standards

### Data Quality Dimensions
- **Accuracy**: Data correctly represents real-world entities
- **Completeness**: All required data fields are present
- **Consistency**: Data is consistent across systems
- **Timeliness**: Data is current and up-to-date
- **Validity**: Data conforms to defined formats and rules

### Data Quality Implementation
```javascript
// Data validation framework
const dataValidation = {
  // Schema validation
  validateSchema: (data, schema) => {
    const validation = schema.validate(data);
    if (validation.error) {
      throw new Error(`Schema validation failed: ${validation.error.message}`);
    }
    return validation.value;
  },
  
  // Data quality checks
  checkDataQuality: (data) => {
    const issues = [];
    
    // Check for required fields
    const requiredFields = ['id', 'name', 'email'];
    requiredFields.forEach(field => {
      if (!data[field]) {
        issues.push(`Missing required field: ${field}`);
      }
    });
    
    // Check email format
    if (data.email && !isValidEmail(data.email)) {
      issues.push('Invalid email format');
    }
    
    // Check data types
    if (data.age && typeof data.age !== 'number') {
      issues.push('Age must be a number');
    }
    
    return issues;
  },
  
  // Data cleansing
  cleanseData: (data) => {
    return {
      ...data,
      name: data.name?.trim(),
      email: data.email?.toLowerCase(),
      phone: data.phone?.replace(/\D/g, '')
    };
  }
};
```

## Data Security Standards

### Data Encryption
- **At Rest**: Encrypt all sensitive data in storage
- **In Transit**: Encrypt data during transmission
- **Key Management**: Secure key generation and storage
- **Encryption Algorithms**: Use industry-standard algorithms

### Data Access Control
```javascript
// Data access control implementation
const dataAccessControl = {
  // Role-based access control
  checkAccess: (user, resource, action) => {
    const userRoles = user.roles || [];
    const requiredPermission = `${resource}:${action}`;
    
    return userRoles.some(role => 
      role.permissions.includes(requiredPermission)
    );
  },
  
  // Data masking for sensitive information
  maskData: (data, userLevel) => {
    if (userLevel < data.classification.level) {
      return {
        ...data,
        ssn: '***-**-' + data.ssn.slice(-4),
        creditCard: '****-****-****-' + data.creditCard.slice(-4),
        email: data.email.replace(/(.{2}).+(@.+)/, '$1***$2')
      };
    }
    return data;
  },
  
  // Data anonymization
  anonymizeData: (data) => {
    return {
      ...data,
      id: hash(data.id),
      name: 'Anonymous',
      email: hash(data.email),
      phone: hash(data.phone)
    };
  }
};
```

## Database Standards

### Database Design Principles
- **Normalization**: Follow database normalization principles
- **Indexing**: Proper indexing for performance
- **Constraints**: Use database constraints for data integrity
- **Migrations**: Version-controlled database schema changes

### Database Implementation Standards
```javascript
// Database migration system
// This is the primary source for database migration standards
const databaseMigrations = {
  // Migration runner
  runMigration: async (migrationName) => {
    try {
      logger.info('Starting database migration', { migration: migrationName });
      
      // Run migration
      await db.migrate.up();
      
      // Verify migration
      const migrations = await db.migrate.list();
      logger.info('Migration completed', { 
        migration: migrationName,
        appliedMigrations: migrations 
      });
      
      return true;
    } catch (error) {
      logger.error('Migration failed', { 
        migration: migrationName,
        error: error.message 
      });
      throw error;
    }
  },
  
  // Schema validation
  validateSchema: async () => {
    const schema = await db.getSchema();
    const issues = [];
    
    // Check for required tables
    const requiredTables = ['users', 'sessions', 'audit_logs'];
    requiredTables.forEach(table => {
      if (!schema.tables.includes(table)) {
        issues.push(`Missing required table: ${table}`);
      }
    });
    
    // Check for required columns
    const requiredColumns = {
      users: ['id', 'email', 'created_at'],
      sessions: ['id', 'user_id', 'expires_at']
    };
    
    Object.entries(requiredColumns).forEach(([table, columns]) => {
      columns.forEach(column => {
        if (!schema.tables[table]?.columns.includes(column)) {
          issues.push(`Missing required column: ${table}.${column}`);
        }
      });
    });
    
    return issues;
  }
};
```

## Data Backup and Recovery Standards

### Backup Strategy
- **Full Backups**: Daily full database backups
- **Incremental Backups**: Hourly incremental backups
- **Point-in-Time Recovery**: Support for point-in-time recovery
- **Backup Testing**: Regular backup restoration testing

### Recovery Procedures
```javascript
// Data recovery procedures
const dataRecovery = {
  // Backup verification
  verifyBackup: async (backupId) => {
    try {
      const backup = await backupService.getBackup(backupId);
      const integrity = await backupService.verifyIntegrity(backup);
      
      logger.info('Backup verification completed', {
        backupId,
        integrity: integrity.valid,
        size: backup.size,
        timestamp: backup.timestamp
      });
      
      return integrity.valid;
    } catch (error) {
      logger.error('Backup verification failed', { backupId, error: error.message });
      return false;
    }
  },
  
  // Data restoration
  restoreData: async (backupId, targetEnvironment) => {
    try {
      logger.info('Starting data restoration', { backupId, targetEnvironment });
      
      // Stop application
      await stopApplication();
      
      // Restore database
      await backupService.restore(backupId, targetEnvironment);
      
      // Verify restoration
      await verifyDataIntegrity();
      
      // Start application
      await startApplication();
      
      logger.info('Data restoration completed', { backupId, targetEnvironment });
      return true;
    } catch (error) {
      logger.error('Data restoration failed', { 
        backupId, 
        targetEnvironment, 
        error: error.message 
      });
      throw error;
    }
  }
};
```

## Data Compliance Standards

### Regulatory Compliance
- **GDPR**: General Data Protection Regulation compliance
- **CCPA**: California Consumer Privacy Act compliance
- **HIPAA**: Health Insurance Portability and Accountability Act
- **SOX**: Sarbanes-Oxley Act compliance

### Compliance Implementation
```javascript
// GDPR compliance helpers
const gdprCompliance = {
  // Data subject rights
  dataSubjectRights: {
    // Right to access
    access: async (userId) => {
      const userData = await getUserData(userId);
      return userData;
    },
    
    // Right to rectification
    rectification: async (userId, data) => {
      await updateUserData(userId, data);
      logDataEvent({
        type: 'data_rectification',
        userId,
        action: 'rectification',
        metadata: { updatedFields: Object.keys(data) }
      });
    },
    
    // Right to erasure
    erasure: async (userId) => {
      await deleteUserData(userId);
      logDataEvent({
        type: 'data_erasure',
        userId,
        action: 'erasure'
      });
    },
    
    // Right to data portability
    portability: async (userId) => {
      const userData = await getUserData(userId);
      return exportUserData(userData);
    }
  },
  
  // Consent management
  consentManagement: {
    record: async (userId, consent) => {
      await recordConsent(userId, consent);
      logDataEvent({
        type: 'consent_recorded',
        userId,
        action: 'consent',
        metadata: { consent }
      });
    },
    
    withdraw: async (userId) => {
      await withdrawConsent(userId);
      logDataEvent({
        type: 'consent_withdrawn',
        userId,
        action: 'consent_withdrawal'
      });
    }
  }
};
```

## Data Performance Standards

### Performance Optimization
- **Query Optimization**: Optimize database queries for performance
- **Indexing Strategy**: Proper indexing for common queries
- **Caching**: Implement caching for frequently accessed data
- **Connection Pooling**: Use connection pooling for efficiency

### Performance Monitoring
```javascript
// Data performance monitoring
const dataPerformance = {
  // Query performance tracking
  trackQueryPerformance: (query, duration) => {
    logger.info('Query performance', {
      query: query.substring(0, 100),
      duration,
      timestamp: new Date().toISOString()
    });
    
    // Alert for slow queries
    if (duration > 1000) {
      logger.warn('Slow query detected', { query, duration });
    }
  },
  
  // Connection pool monitoring
  monitorConnectionPool: (pool) => {
    const stats = {
      total: pool.totalCount,
      idle: pool.idleCount,
      waiting: pool.waitingCount
    };
    
    logger.info('Connection pool status', stats);
    
    // Alert for pool exhaustion
    if (stats.waiting > 5) {
      logger.warn('Connection pool under pressure', stats);
    }
  }
};
```

## Data Standards Checklist

### Development Standards
- [ ] Data validation implemented
- [ ] Data classification applied
- [ ] Access controls configured
- [ ] Encryption implemented
- [ ] Audit logging enabled
- [ ] Backup strategy defined
- [ ] Recovery procedures tested
- [ ] Compliance requirements met

### Operational Standards
- [ ] Data monitoring active
- [ ] Performance monitoring configured
- [ ] Backup verification scheduled
- [ ] Data quality checks automated
- [ ] Compliance audits scheduled
- [ ] Data retention policies enforced
- [ ] Disaster recovery tested
- [ ] Security controls verified

---

## Conclusion

These data standards ensure proper data management, governance, and protection while supporting scalable and reliable data operations. Regular review and updates ensure these standards remain effective and compliant with evolving requirements. 