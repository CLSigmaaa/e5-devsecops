FROM python:3.9-alpine

# Set the working directory
WORKDIR /app

# Copy the requirements file
COPY requirements.txt .

# Install the dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the application files
COPY . .

# Expose the port the app runs on
EXPOSE 8080

# Define the command to run the application
CMD ["python", "run.py"]