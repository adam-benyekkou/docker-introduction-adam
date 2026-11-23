# Image de base Python légère
FROM python:3.12-slim

# Répertoire de travail de l'application
WORKDIR /app

# Sortie Python non bufferisée et pas de .pyc dans l'image
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Dépendances d'abord pour profiter du cache Docker
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Code de l'application Flask et ses templates
COPY app.py ./
COPY templates ./templates

# Port d'écoute de l'application
EXPOSE 5000

# Serveur WSGI de production
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
