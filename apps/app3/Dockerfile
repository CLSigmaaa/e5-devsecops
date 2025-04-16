FROM python:3.9-alpine

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set working directory
WORKDIR /app

# Copy and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Collect static files and apply migrations
RUN python manage.py collectstatic --no-input \
    && python manage.py makemigrations \
    && python manage.py migrate

# Expose port and start server
EXPOSE 9000
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]