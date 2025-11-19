FROM python:3.10-slim-bullseye

WORKDIR /app

COPY requirements.txt .

RUN apt update -y && \
    apt install -y build-essential libpq-dev && \
    pip install --upgrade pip setuptools wheel && \
    pip install -r requirements.txt

COPY . .

EXPOSE 5153

CMD ["python", "app.py"]
