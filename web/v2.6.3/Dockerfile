FROM ruby:2.7.2

ARG RENDER_EXTERNAL_HOSTNAME
ARG CUSTOM_DOMAIN
ARG DISCOURSE_WORKER_CONCURRENCY

ENV DISCOURSE_HOSTNAME=${CUSTOM_DOMAIN:-$RENDER_EXTERNAL_HOSTNAME}
ENV RAILS_ENV=production
ENV DISCOURSE_VERSION=v2.6.3
ENV DISCOURSE_ROOT=/var/www/discourse
ENV DISCOURSE_WORKER_CONCURRENCY=${DISCOURSE_WORKER_CONCURRENCY:-2}

WORKDIR $DISCOURSE_ROOT

# Pull the Discourse app code at the pinned version
RUN git clone https://github.com/discourse/discourse.git --branch $DISCOURSE_VERSION --single-branch $DISCOURSE_ROOT

# ----- System + build deps needed for Discourse 2.6.x -----
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      ca-certificates curl gnupg dirmngr \
      build-essential git python3 make g++ \
      imagemagick \
      brotli gifsicle jhead jpegoptim optipng pngquant \
      postgresql-client libpq-dev \
      zlib1g-dev libssl-dev libyaml-dev libreadline-dev \
      libxml2-dev libxslt1-dev libffi-dev libicu-dev file \
      pkg-config && \
    rm -rf /var/lib/apt/lists/*

# ----- Node 16 LTS + yarn (matches era of Discourse v2.6.x) -----
RUN curl -fsSL https://deb.nodesource.com/setup_16.x | bash - && \
    apt-get update && \
    apt-get install -y --no-install-recommends nodejs && \
    npm install -g yarn svgo && \
    rm -rf /var/lib/apt/lists/*

# Install JS deps (in app dir) for asset build
RUN yarn install --frozen-lockfile

# ----- Ruby gems (production) -----
RUN gem install bundler -v "~> 2.3"
RUN bundle config set path 'vendor/bundle' && \
    bundle config set deployment 'true' && \
    bundle config set without 'development test' && \
    bundle install --jobs 4 --retry 3

# Foreman to run Procfile web/worker
RUN gem install foreman

# ----- NGINX -----
COPY install-nginx.sh /tmp/install-nginx.sh
RUN chmod +x /tmp/install-nginx.sh && /tmp/install-nginx.sh && \
    cp config/nginx.sample.conf /etc/nginx/conf.d/discourse.conf && \
    sed -i "s/^  server_name enter\.your\.web\.hostname\.here;/  server_name $DISCOURSE_HOSTNAME;/" /etc/nginx/conf.d/discourse.conf && \
    mkdir -p /var/nginx/cache/

# ----- Redis -----
COPY install-redis.sh /tmp/install-redis.sh
COPY redis.conf .
RUN chmod +x /tmp/install-redis.sh && /tmp/install-redis.sh

# Force HTTPS (Render provides TLS)
RUN perl -i -p0e 's/  force_https:\n    default: false/  force_https:\n    default: true/' config/site_settings.yml

# Runtime dirs/log
RUN mkdir -p tmp/sockets/ tmp/pids/ && touch log/production.log

# App configs + entrypoint
COPY puma.rb sidekiq.yml ./config/
COPY entrypoint.sh Procfile ./
RUN sed -i "s/^  :concurrency: 2/  :concurrency: ${DISCOURSE_WORKER_CONCURRENCY}/" config/sidekiq.yml
RUN chmod +x entrypoint.sh

ENTRYPOINT [ "bash", "entrypoint.sh" ]
