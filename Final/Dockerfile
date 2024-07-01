# syntax = docker/dockerfile:1
FROM ruby:3.3.0-slim
LABEL maintainer="jstrausb86@gmail.com"

# Add basic packages
RUN apt-get update && apt-get install -y --no-install-recommends \
      xz-utils \
      postgresql-client \
      libgomp1 \
      tzdata \
      file \
      netcat-openbsd \
      imagemagick \
      libmagickwand-6.q16-6 \
      libvips \
      libglib2.0-0 \
      libglib2.0-dev \
      libgomp1 \
      libjpeg62-turbo \
      libpng16-16 \
      libwebp7 \
      libgif7 \
      librsvg2-2 \
    && rm -rf /var/lib/apt/lists/*

# Set LD_LIBRARY_PATH
ENV LD_LIBRARY_PATH=/lib:/usr/lib:/usr/local/lib:/lib/x86_64-linux-gnu:/usr/lib/x86_64-linux-gnu

# Set LIBGLIB_PATH
ENV LIBGLIB_PATH=/lib/x86_64-linux-gnu/libglib-2.0.so.0

# Verify glib installation
RUN ldconfig -p | grep glib && \
    ls -l /lib/x86_64-linux-gnu/libglib-2.0.so* && \
    ls -l /usr/lib/x86_64-linux-gnu/libglib-2.0.so*

# Add debugging information
RUN echo "LIBGLIB_PATH: $LIBGLIB_PATH" && \
    ls -l $LIBGLIB_PATH && \
    ldd $LIBGLIB_PATH

# Configure Rails
ENV RAILS_LOG_TO_STDOUT=true
ENV RAILS_SERVE_STATIC_FILES=true
ENV MALLOC_ARENA_MAX=2

WORKDIR /app

# Expose Puma port
EXPOSE 3000

# This image is for production env only
ENV RAILS_ENV=production

# Write GIT meta data from arguments to env vars
ONBUILD ARG COMMIT_SHA
ONBUILD ARG COMMIT_TIME
ONBUILD ARG COMMIT_VERSION
ONBUILD ARG COMMIT_BRANCH

ONBUILD ENV COMMIT_SHA ${COMMIT_SHA}
ONBUILD ENV COMMIT_TIME ${COMMIT_TIME}
ONBUILD ENV COMMIT_VERSION ${COMMIT_VERSION}
ONBUILD ENV COMMIT_BRANCH ${COMMIT_BRANCH}

# Add user
ONBUILD RUN addgroup --gid 1000 app && \
      adduser --uid 1000 --ingroup app --shell /bin/sh --disabled-password --no-create-home app

# Copy app with gems from former build stage
ONBUILD COPY --from=Builder --chown=app:app /usr/local/bundle/ /usr/local/bundle/
ONBUILD COPY --from=Builder --chown=app:app /app /app
