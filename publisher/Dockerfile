# scottzach1/redis-publisher
FROM python:3.13.0a4-alpine

ENV PYTHONFAULTHANDLER=1 \
    PYTHONUNBUFFERED=1 \
    PYTHONHASHSEED=random \
    PIP_NO_CACHE_DIR=off \
    PIP_DISABLE_PIP_VERSION_CHECK=on \
    PIP_DEFAULT_TIMEOUT=100 \
    POETRY_VERSION=1.1.3

# System dependencies
# https://stackoverflow.com/a/70316507
RUN apk update && apk add python3-dev gcc libc-dev libffi-dev
RUN pip install "poetry==$POETRY_VERSION"

# Copy dependencies
WORKDIR /code
COPY poetry.lock pyproject.toml /code/

# Install dependencies
RUN poetry config virtualenvs.create false && \
    poetry install --no-ansi

# Copy codebase
COPY . /code

ENTRYPOINT ["python", "publish.py"]
