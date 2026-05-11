FROM python:3.12-slim

WORKDIR /app

COPY app_map/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app_map/ ./app_map/
COPY app_map/statics/ ./statics/

EXPOSE 8000

CMD ["fastapi", "run", "app_map/main.py", "--host", "0.0.0.0", "--port", "8000"]
