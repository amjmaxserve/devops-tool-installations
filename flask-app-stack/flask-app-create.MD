# Flask Application DevOps Journey

## Overview

This document provides a complete step-by-step guide to build, containerize, deploy, and observe a Flask application backed by PostgreSQL.

The workflow includes:

- Flask application development
- PostgreSQL database setup
- Local development with Python virtualenv
- Docker containerization
- Docker Compose orchestration
- Kubernetes deployment using KIND
- Helm packaging and deployment
- Secret management
- Observability support with Prometheus, Grafana, and related exporters

## Prerequisites

Install the following on your Ubuntu system:

- Python 3.12
- Git
- Docker
- Docker Compose
- kubectl
- KIND
- Helm

## Phase 1 — Create the Flask + PostgreSQL Application Locally

### 1. Create the project directory

```bash
mkdir -p ~/flask-postgres-app
cd ~/flask-postgres-app
```

### 2. Create and activate a Python virtual environment

```bash
python3 -m venv venv
source venv/bin/activate
```

### 3. Install Python dependencies

```bash
pip install flask flask-sqlalchemy psycopg2-binary python-dotenv
pip freeze > requirements.txt
```

### 4. Create environment variables file

Create a file named `.env` with the following content:

```env
POSTGRES_USER=flaskuser
POSTGRES_PASSWORD=flaskpass
POSTGRES_HOST=localhost
POSTGRES_DB=flaskdb
```

This file stores database connection values for local development.

### 5. Create the Flask app

Create `app.py` with the following content:

```python
from flask import Flask, render_template, request, redirect
from flask_sqlalchemy import SQLAlchemy
from dotenv import load_dotenv
import os

load_dotenv()

app = Flask(__name__)

DB_USER = os.getenv("POSTGRES_USER")
DB_PASSWORD = os.getenv("POSTGRES_PASSWORD")
DB_HOST = os.getenv("POSTGRES_HOST")
DB_NAME = os.getenv("POSTGRES_DB")

app.config['SQLALCHEMY_DATABASE_URI'] = (
    f'postgresql://{DB_USER}:{DB_PASSWORD}@{DB_HOST}/{DB_NAME}'
)
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False

db = SQLAlchemy(app)

class Message(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    content = db.Column(db.String(200), nullable=False)

    def __repr__(self):
        return f"<Message {self.content}>"

@app.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        content = request.form['content']
        if content.strip():
            msg = Message(content=content)
            db.session.add(msg)
            db.session.commit()
        return redirect('/')

    messages = Message.query.all()
    return render_template('index.html', messages=messages)

@app.before_request
def create_tables():
    db.create_all()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

### 6. Add the HTML template

Create the folder and file:

```bash
mkdir -p templates
cat > templates/index.html <<'HTML'
<!DOCTYPE html>
<html>
  <head>
    <meta charset="UTF-8" />
    <title>Flask PostgreSQL App</title>
  </head>
  <body>
    <h1>Flask + PostgreSQL App</h1>
    <form method="POST">
      <input type="text" name="content" placeholder="Enter message" required />
      <button type="submit">Save</button>
    </form>
    <hr />
    <h2>Messages</h2>
    <ul>
      {% for message in messages %}
      <li>{{ message.content }}</li>
      {% endfor %}
    </ul>
  </body>
</html>
HTML
```

### 7. Initialize the PostgreSQL database locally

Use a local PostgreSQL container for development:

```bash
mkdir -p ~/docker-vols/postgres_vol
docker run -d --name postgres_db -p 5432:5432 \
  -v ~/docker-vols/postgres_vol:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD='pg123' postgres:12
```

### 8. Create database and user

Connect inside the PostgreSQL container and create the database:

```bash
docker exec -ti postgres_db bash
psql -U postgres
```

In the `psql` shell:

```sql
CREATE DATABASE flaskdb;
CREATE USER flaskuser WITH LOGIN ENCRYPTED PASSWORD 'flaskpass';
GRANT ALL PRIVILEGES ON DATABASE flaskdb TO flaskuser;
\q
exit
```

### 9. Run the Flask application locally

Start the app from the project directory:

```bash
source venv/bin/activate
python app.py
```

Visit `http://localhost:5000` to confirm the app is running.

## Phase 2 — Dockerize the Flask Application

### 1. Create a Dockerfile

Create `Dockerfile` with the following content:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

This Dockerfile builds a lightweight image for the Flask app.

### 2. Create Docker Compose configuration

Create `docker-compose.yml`:

```yaml
version: '3.9'
services:
  web:
    build: .
    container_name: flask-app
    ports:
      - "5000:5000"
    env_file:
      - .env
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16
    container_name: postgres-db
    env_file:
      - .env
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres_data:
```

### 3. Start the stack with Docker Compose

```bash
docker-compose up -d --build
```

### 4. Verify the containers

```bash
docker ps
```

### 5. Confirm application access

Open `http://localhost:5000` in your browser.

## Conclusion

This document covers the local application creation and Docker Compose deployment phases for the Flask + PostgreSQL project.

- Phase 1 includes application development, local database setup, and running the Flask app in a virtual environment.
- Phase 2 includes containerizing the app with Docker and running it with Docker Compose.
- Phase 3 is now available in a separate document: `flask-app-phase3.MD`.

Use this file for local development and testing, and use `flask-app-phase3.MD` for Kubernetes deployment.

