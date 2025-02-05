# syntax = docker/dockerfile:1
FROM ruby:3.3.0-slim
LABEL maintainer="jstrausb86@gmail.com"

# Install dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    libglib2.0-dev \
    libvips-dev \
    libmagickwand-dev \
    libpq-dev \
    libsqlite3-dev \
    default-libmysqlclient-dev \
    git \
    tzdata \
    curl \
    unzip \
    && rm -rf /var/lib/apt/lists/* \
    && curl -fsSL https://bun.sh/install | bash

# Update PATH for Bun
ENV BUN_INSTALL="/root/.bun"
ENV PATH="$BUN_INSTALL/bin:$PATH"

WORKDIR /app

#### Install standard Node modules
COPY package.json bun.lockb /app/
# If your app uses Cypress for end-to-end testing, installing the binary can be slow, so we
# explicitly exclude it from installing (because tests are not run here)
ENV CYPRESS_INSTALL_BINARY=0
RUN bun install
####

##### Install standard gems
COPY Gemfile Gemfile.lock /app/
RUN bundle config --local frozen 1 && \
    bundle install -j4 --retry 3
####

#### ONBUILD: Add triggers to the image, executed later while building a child image

# Copy only the files needed for installing gems
ONBUILD COPY Gemfile Gemfile.lock /app/
ONBUILD COPY vendor/ /app/vendor/
ONBUILD COPY .ruby-version /app/

# Install Ruby gems (for production only)
ONBUILD RUN --mount=type=secret,id=bundleconfig,dst=/root/.bundle/config \
    bundle config --local without 'development test' && \
    bundle install -j4 --retry 3 && \
    # Precompile gems with Bootsnap (and ignore errors)
    bundle exec bootsnap precompile --gemfile || true && \
    # Remove unneeded gems
    bundle clean --force && \
    # Remove unneeded files from installed gems (cache, *.o, *.c)
    rm -rf /usr/local/bundle/cache && \
    find /usr/local/bundle/gems/ -name "*.c" -delete && \
    find /usr/local/bundle/gems/ -name "*.o" -delete

# Copy the whole application folder into the image
ONBUILD COPY . /app

# Precompile application code with Bootsnap (and ignore errors)
ONBUILD RUN bundle exec bootsnap precompile app/ lib/ || true

# Precompile assets
#
# Notes:
#   1. What exactly "assets:precompile" does depends on your JavaScript bundler
#   2. For an app using encrypted credentials, Rails raises a `MissingKeyError`
#      if the master key is missing. Because on CI there is no master key,
#      we hide the credentials while compiling assets (by renaming them before and after)
#
ONBUILD RUN mv config/credentials.yml.enc config/credentials.yml.enc.bak 2>/dev/null || true
ONBUILD RUN mv config/credentials config/credentials.bak 2>/dev/null || true
ONBUILD RUN bun install

# Currently, it seems a hack is needed to get the Vite bundle activity from
# `bundle exec rails assets:precompile`to ignore an expectation for node
# Create a symbolic link for 'node' to point to 'bun'
ONBUILD RUN ln -s $(which bun) /usr/bin/node

# If sensitive data like env vars are needed during asset compilation:
# 1. Mount the secrets to the target directory
# 2. In your Rails app, set up an initializer script to extract the secrets from the
#    mounted directory, so they're available on ENV and provided to asset compilation
# 3. At image build, pass the secrets from a local file as a --secret build arg:
#   `docker build --secret id=env_secrets,src=.env`
ONBUILD RUN --mount=type=secret,id=env_secrets,target=/app/env_secrets \
    export RAILS_ENV=production \
    && bundle exec rails assets:precompile
ONBUILD RUN mv config/credentials.yml.enc.bak config/credentials.yml.enc 2>/dev/null || true
ONBUILD RUN mv config/credentials.bak config/credentials 2>/dev/null || true

# Remove folders not needed in resulting image
# This includes `app/javascript` which contains the JavaScript source code.
# Normally it is not needed in the resulting image, because it was compiled
# to `public/`. But if the app uses importmaps, the folder is still required
# for pinning and must not be removed.
ONBUILD RUN rm -rf node_modules bun.lockb vendor/bundle test spec app/packs
# .yarn .yarnrc.yml
ONBUILD RUN if [ ! -f config/importmap.rb ]; then rm -rf app/javascript; fi
