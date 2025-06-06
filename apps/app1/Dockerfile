# FROM python:3.9

# # set environment variables
# ENV PYTHONDONTWRITEBYTECODE 1
# ENV PYTHONUNBUFFERED 1

# COPY requirements.txt .
# # install python dependencies
# RUN pip install --upgrade pip
# RUN pip install --no-cache-dir -r requirements.txt

# COPY . .

# # Set UP
# RUN python manage.py collectstatic --no-input
# RUN python manage.py makemigrations
# RUN python manage.py migrate

# #__API_GENERATOR__
# #__API_GENERATOR__END

# # Start Server
# EXPOSE 5005
# CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]
FROM python:3.9-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

# Set working directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Collect static files and apply migrations
RUN python manage.py collectstatic --no-input && \
    python manage.py makemigrations && \
    python manage.py migrate

# Expose port and set command to run the application
EXPOSE 80
CMD ["gunicorn", "--config", "gunicorn-cfg.py", "core.wsgi"]