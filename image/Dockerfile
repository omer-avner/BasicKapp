FROM python:3-slim
ARG SERVER_BASE_DIR=/usr/local/bin
EXPOSE 8000/tcp
WORKDIR $SERVER_BASE_DIR
ENTRYPOINT ["python3", "-m", "http.server", "--bind", "0.0.0.0", "8000"]
