FROM python:3.8-slim-buster

RUN apt-get update && apt-get install -y python3-dev build-essential

WORKDIR /app

COPY requirements.txt .
RUN pip3 install -r requirements.txt

COPY . .

# *Part 1: FastAPI + Docker
# EXPOSE 5000
# CMD uvicorn --host 0.0.0.0 --port 5000 app:app

# *Part 2: Heroku + Github actions
# CMD uvicorn --host 0.0.0.0 --port $PORT app:app

# *Part 3: Prometheus + Grafana
CMD uvicorn --host 0.0.0.0 app:app