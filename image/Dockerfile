FROM python:3-slim

# Override the images default base directory using docker build --build-arg SERVER_BASE_DIR=<base_dir>
ARG SERVER_BASE_DIR=/usr/local/bin
EXPOSE 8000/tcp
WORKDIR $SERVER_BASE_DIR

# Add the argument --directory <dir_path> to change the base directory at runtime
ENTRYPOINT ["python3", "-m", "http.server", "--bind", "0.0.0.0", "8000"]
