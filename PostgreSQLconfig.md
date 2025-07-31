# 🐘 PostgreSQL Configuration for Xampo

This guide explains how to add PostgreSQL database support to your Xampo environment alongside or instead of MySQL.

## 🎯 Overview

Xampo comes with PostgreSQL extensions pre-installed (`pgsql`, `pdo_pgsql`), but PostgreSQL service needs to be configured manually. This guide shows you how to set it up.

## 📋 Prerequisites

- ✅ Xampo already installed and working
- ✅ Basic understanding of Docker Compose
- ✅ Text editor access to configuration files

## 🚀 Configuration Steps

### Step 1: Add PostgreSQL Service to docker-compose.yml

**Edit your `docker-compose.yml` file:**
```bash
nano docker-compose.yml
```

**Add the PostgreSQL service after the existing `db` service:**
```yaml
services:
  # ... existing web service ...
  
  # MySQL service (existing)
  db:
    # ... existing MySQL configuration ...
  
  # PostgreSQL service (NEW)
  postgres:
    image: postgres:15
    container_name: ${CONTAINER_POSTGRES_NAME}
    restart: always
    environment:
      POSTGRES_DB: ${POSTGRES_DATABASE}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      TZ: ${TZ}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      # Only accessible from localhost for security
      - "127.0.0.1:5432:5432"
    networks:
      - app-network

  # ... existing phpmyadmin service ...

volumes:
  db_data_avanzado:
  postgres_data:  # NEW volume for PostgreSQL data

networks:
  app-network:
    driver: bridge
```

### Step 2: Update Environment Variables

**Edit your `.env` file:**
```bash
nano .env
```

**Add PostgreSQL variables:**
```env
# Existing container names
CONTAINER_WEB_NAME=d7_web
CONTAINER_DB_NAME=d7_db
CONTAINER_PHPMYADMIN_NAME=d7_phpmyadmin
CONTAINER_POSTGRES_NAME=d7_postgres

# Existing MySQL configuration
TZ=America/Bogota
MYSQL_ROOT_PASSWORD=Hholamundo256@
MYSQL_DATABASE=mi_base_de_datos
MYSQL_USER=mi_usuario_db
MYSQL_PASSWORD=root

# NEW PostgreSQL configuration
POSTGRES_DATABASE=mi_base_postgres
POSTGRES_USER=postgres_user
POSTGRES_PASSWORD=postgres_password123
```

### Step 3: Add pgAdmin (Optional)

**For PostgreSQL database management, add pgAdmin service:**
```yaml
  # pgAdmin for PostgreSQL management (OPTIONAL)
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: ${CONTAINER_PGADMIN_NAME}
    restart: always
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin123
      PGADMIN_CONFIG_SERVER_MODE: 'False'
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    ports:
      - "127.0.0.1:5050:80"
    depends_on:
      - postgres
    networks:
      - app-network
```

**Add to volumes section:**
```yaml
volumes:
  db_data_avanzado:
  postgres_data:
  pgadmin_data:  # NEW volume for pgAdmin
```

**Add to .env:**
```env
CONTAINER_PGADMIN_NAME=d7_pgadmin
```

### Step 4: Start the Services

**Deploy the updated configuration:**
```bash
# Stop existing services
docker-compose down

# Build and start all services
docker-compose up -d --build

# Verify all containers are running
docker-compose ps
```

## 🔌 PHP Connection Examples

### Basic PDO Connection
```php
<?php
try {
    $host = 'postgres';  // Service name from docker-compose.yml
    $dbname = 'mi_base_postgres';  // From POSTGRES_DATABASE
    $username = 'postgres_user';   // From POSTGRES_USER
    $password = 'postgres_password123';  // From POSTGRES_PASSWORD
    
    $dsn = "pgsql:host=$host;dbname=$dbname;port=5432";
    $pdo = new PDO($dsn, $username, $password, [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
    ]);
    
    echo "✅ PostgreSQL connection successful!";
} catch (PDOException $e) {
    echo "❌ Connection failed: " . $e->getMessage();
}
?>
```

### Environment-Based Connection
```php
<?php
// config/database.php
return [
    'postgresql' => [
        'host' => getenv('POSTGRES_HOST') ?: 'postgres',
        'port' => getenv('POSTGRES_PORT') ?: '5432',
        'database' => getenv('POSTGRES_DATABASE'),
        'username' => getenv('POSTGRES_USER'),
        'password' => getenv('POSTGRES_PASSWORD'),
    ],
    'mysql' => [
        'host' => 'db',
        'database' => getenv('MYSQL_DATABASE'),
        'username' => getenv('MYSQL_USER'),
        'password' => getenv('MYSQL_PASSWORD'),
    ]
];
?>
```

### Laravel Configuration
```php
// .env (Laravel)
DB_CONNECTION=pgsql
DB_HOST=postgres
DB_PORT=5432
DB_DATABASE=mi_base_postgres
DB_USERNAME=postgres_user
DB_PASSWORD=postgres_password123
```

## 🔧 Database Management

### Access pgAdmin
- **URL:** http://localhost:5050
- **Email:** admin@example.com
- **Password:** admin123

### Connect to PostgreSQL from pgAdmin
1. Right-click "Servers" → "Create" → "Server"
2. **General tab:**
   - Name: `Xampo PostgreSQL`
3. **Connection tab:**
   - Host: `postgres`
   - Port: `5432`
   - Database: `mi_base_postgres`
   - Username: `postgres_user`
   - Password: `postgres_password123`

### Command Line Access
```bash
# Access PostgreSQL container
docker-compose exec postgres bash

# Connect to PostgreSQL
psql -U postgres_user -d mi_base_postgres

# List databases
\l

# List tables
\dt

# Exit
\q
```

## 🔄 Migration from MySQL

### Export MySQL Data
```bash
# From MySQL container
docker-compose exec db mysqldump -u root -p mi_base_de_datos > backup.sql
```

### Convert and Import to PostgreSQL
```bash
# Install conversion tools (on host)
sudo apt install mysql2pgsql

# Convert MySQL dump to PostgreSQL format
mysql2pgsql backup.sql backup_postgres.sql

# Import to PostgreSQL
docker-compose exec postgres psql -U postgres_user -d mi_base_postgres -f backup_postgres.sql
```

## 🚀 Performance Optimizations

### PostgreSQL Configuration
**Create custom postgresql.conf (optional):**
```bash
# Create config directory
mkdir -p ./postgres/config

# Create custom configuration
cat > ./postgres/config/postgresql.conf << EOF
# Memory settings
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 4MB

# Connection settings
max_connections = 100
listen_addresses = '*'

# Performance settings
random_page_cost = 1.1
effective_io_concurrency = 200
EOF
```

**Mount in docker-compose.yml:**
```yaml
postgres:
  # ... existing configuration ...
  volumes:
    - postgres_data:/var/lib/postgresql/data
    - ./postgres/config/postgresql.conf:/etc/postgresql/postgresql.conf
  command: postgres -c config_file=/etc/postgresql/postgresql.conf
```

## 🔍 Troubleshooting

### Common Issues

**1. PostgreSQL won't start:**
```bash
# Check logs
docker-compose logs postgres

# Check if port is available
netstat -tulpn | grep 5432

# Restart service
docker-compose restart postgres
```

**2. Connection refused:**
```bash
# Verify service is running
docker-compose ps

# Check network connectivity
docker-compose exec web ping postgres

# Test connection from PHP container
docker-compose exec web php -r "
$pdo = new PDO('pgsql:host=postgres;dbname=mi_base_postgres', 'postgres_user', 'postgres_password123');
echo 'Connected successfully';
"
```

**3. Authentication failed:**
- Verify username/password in `.env`
- Check PostgreSQL logs: `docker-compose logs postgres`
- Ensure environment variables are loaded: `docker-compose config`

## 📊 URLs and Access Points

After configuration, your services will be available at:

- **Main Application:** http://localhost:8080
- **phpMyAdmin (MySQL):** http://localhost:8081
- **pgAdmin (PostgreSQL):** http://localhost:5050
- **Direct PostgreSQL:** localhost:5432

## 🎯 Best Practices

### Security
- ✅ Use strong passwords in production
- ✅ Bind ports to localhost only
- ✅ Regular backups of both databases
- ✅ Keep PostgreSQL updated

### Performance
- ✅ Use connection pooling for high-traffic apps
- ✅ Index frequently queried columns
- ✅ Monitor query performance
- ✅ Optimize PostgreSQL configuration

### Development
- ✅ Use different databases for different projects
- ✅ Environment-specific configurations
- ✅ Version control your schema changes
- ✅ Document your database structure

---

**🎉 Congratulations! PostgreSQL is now integrated with your Xampo environment!**

*You can now build applications using both MySQL and PostgreSQL databases with the same PHP environment.* 🚀 