ARG SERVER_BASE_DIR=/usr/local/bin
FROM python:3-slim
EXPOSE 8000/tcp
WORKDIR $SERVER_BASE_DIR
CMD ["python3", "-m", "http.server", "--bind", "0.0.0.0", "8000"]
