# Copyright 2020 Google LLC

ARG BUILDPLATFORM=linux/amd64

# =========================
# BUILDER
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

RUN pip install --no-cache-dir -r requirements.txt

# Install fixed security versions
RUN pip install --no-cache-dir --upgrade \
    "jaraco.context==6.1.0" \
    "msgpack==1.2.1" \
    "setuptools==78.1.1" \
    "wheel==0.46.2"

# Verify builder versions
RUN python -c "\
import importlib.metadata as m; \
assert m.version('jaraco.context') == '6.1.0'; \
assert m.version('msgpack') == '1.2.1'; \
assert m.version('setuptools') == '78.1.1'; \
assert m.version('wheel') == '0.46.2'; \
print('All Python package versions are fixed')"


# =========================
# RUNTIME
# =========================
FROM --platform=$BUILDPLATFORM python:3.14.7-alpine@sha256:05b2b8b732ecd268fee8727a369f936f022d1321b59befd13c30ede22769dcdc

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

RUN apk update \
    && apk upgrade \
    && apk add --no-cache libstdc++ \
    && rm -rf /var/cache/apk/*

# Copy Python runtime and application dependencies
COPY --from=builder /usr/local/lib/python3.14/ /usr/local/lib/python3.14/

# Remove build-only packaging tools and ALL of their metadata.
# This also removes vulnerable vendored copies such as:
# setuptools/_vendor/wheel-0.45.1.dist-info
RUN rm -rf \
    /usr/local/lib/python3.14/site-packages/setuptools \
    /usr/local/lib/python3.14/site-packages/setuptools-*.dist-info \
    /usr/local/lib/python3.14/site-packages/wheel \
    /usr/local/lib/python3.14/site-packages/wheel-*.dist-info \
    /usr/local/lib/python3.14/site-packages/jaraco \
    /usr/local/lib/python3.14/site-packages/jaraco.context-*.dist-info \
    /usr/local/lib/python3.14/site-packages/pip

# Remove any remaining vulnerable packaging metadata anywhere
# underneath site-packages.
RUN find /usr/local/lib/python3.14/site-packages \
    -type d \
    \( \
        -name 'setuptools*' \
        -o -name 'wheel*' \
        -o -name 'jaraco.context*' \
    \) \
    -exec rm -rf {} +

WORKDIR /recommendationservice

COPY . .

ENV PORT="8080"

EXPOSE 8080

ENTRYPOINT ["python", "recommendation_server.py"]
