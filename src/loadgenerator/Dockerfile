# Copyright 2020 Google LLC

ARG BUILDPLATFORM=linux/amd64

# =========================
# Builder
# =========================
FROM --platform=$BUILDPLATFORM python:3.14.7-alpine@sha256:05b2b8b732ecd268fee8727a369f936f022d1321b59befd13c30ede22769dcdc AS builder

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

RUN apk update \
    && apk upgrade \
    && apk add --no-cache g++ linux-headers \
    && rm -rf /var/cache/apk/*

WORKDIR /build

COPY requirements.txt .

# Create isolated Python environment
RUN python -m venv /opt/venv

# Install application dependencies
RUN /opt/venv/bin/pip install --no-cache-dir -r requirements.txt

# Make sure the fixed msgpack version is installed
RUN /opt/venv/bin/pip install \
    --no-cache-dir \
    --upgrade \
    "msgpack==1.2.1"

# Remove packaging tools from the APPLICATION environment.
# IMPORTANT: Do NOT search/delete msgpack globally,
# because pip has its own vendored msgpack.
RUN /opt/venv/bin/pip uninstall -y setuptools wheel jaraco.context || true

# Remove any remaining packaging metadata from application site-packages.
RUN rm -rf \
    /opt/venv/lib/python3.14/site-packages/setuptools \
    /opt/venv/lib/python3.14/site-packages/setuptools-*.dist-info \
    /opt/venv/lib/python3.14/site-packages/wheel \
    /opt/venv/lib/python3.14/site-packages/wheel-*.dist-info \
    /opt/venv/lib/python3.14/site-packages/jaraco \
    /opt/venv/lib/python3.14/site-packages/jaraco.context-*.dist-info

# Verify application package
RUN /opt/venv/bin/python -c \
    "import importlib.metadata as m; print('msgpack=', m.version('msgpack')); assert m.version('msgpack') == '1.2.1'"

# =========================
# Runtime
# =========================
FROM python:3.14.7-alpine@sha256:05b2b8b732ecd268fee8727a369f936f022d1321b59befd13c30ede22769dcdc

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PATH="/opt/venv/bin:$PATH"

RUN apk update \
    && apk upgrade \
    && apk add --no-cache libstdc++ \
    && rm -rf /var/cache/apk/*

# Remove packaging tools inherited from the Python base image.
RUN rm -rf \
    /usr/local/lib/python3.14/site-packages/setuptools \
    /usr/local/lib/python3.14/site-packages/setuptools-*.dist-info \
    /usr/local/lib/python3.14/site-packages/wheel \
    /usr/local/lib/python3.14/site-packages/wheel-*.dist-info \
    /usr/local/lib/python3.14/site-packages/jaraco \
    /usr/local/lib/python3.14/site-packages/jaraco.context-*.dist-info

# Copy application environment
COPY --from=builder /opt/venv /opt/venv

# pip is only required during the build.
# Do NOT ship pip in the production image.
RUN rm -rf \
    /opt/venv/lib/python3.14/site-packages/pip \
    /opt/venv/lib/python3.14/site-packages/pip-*.dist-info \
    /opt/venv/bin/pip \
    /opt/venv/bin/pip3 \
    /usr/local/lib/python3.14/site-packages/pip \
    /usr/local/lib/python3.14/site-packages/pip-*.dist-info \
    /usr/local/bin/pip \
    /usr/local/bin/pip3

WORKDIR /loadgenerator

COPY . .

ENV PORT="8080"

EXPOSE 8080

ENTRYPOINT ["python", "loadgenerator.py"]
