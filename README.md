- 下载代码

  ```
  git clone https://github.com/getredash/redash.git
  ```

- 配置docker file

  ```dockerfile
  FROM node:12 as frontend-builder
  
  RUN npm config set registry https://registry.npm.taobao.org
  WORKDIR /frontend
  COPY package.json package-lock.json /frontend/
  RUN npm ci
  
  COPY client /frontend/client
  COPY webpack.config.js /frontend/
  RUN npm run build
  
  FROM python:3.7-slim
  
  RUN pip config set global.index-url http://mirrors.aliyun.com/pypi/simple
  RUN pip config set install.trusted-host mirrors.aliyun.com
  
  EXPOSE 5000
  
  # Controls whether to install extra dependencies needed for all data sources.
  ARG skip_ds_deps
  
  RUN useradd --create-home redash
  
  # Ubuntu packages
  RUN apt-get update && \
    apt-get install -y \
      curl \
      gnupg \
      build-essential \
      pwgen \
      libffi-dev \
      sudo \
      git-core \
      wget \
      # Postgres client
      libpq-dev \
      # ODBC support:
      g++ unixodbc-dev \
      # for SAML
      xmlsec1 \
      # Additional packages required for data sources:
      libssl-dev \
      default-libmysqlclient-dev \
      freetds-dev \
      libsasl2-dev && \
    # MSSQL ODBC Driver:
    curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
    curl https://packages.microsoft.com/config/debian/10/prod.list > /etc/apt/sources.list.d/mssql-release.list && \
    apt-get update && \
    ACCEPT_EULA=Y apt-get install -y msodbcsql17 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
  
  WORKDIR /app
  
  # Disalbe PIP Cache and Version Check
  ENV PIP_DISABLE_PIP_VERSION_CHECK=1
  ENV PIP_NO_CACHE_DIR=1
  
  # We first copy only the requirements file, to avoid rebuilding on every file
  # change.
  COPY requirements.txt requirements_bundles.txt requirements_dev.txt requirements_all_ds.txt ./
  RUN pip install -r requirements.txt -r requirements_dev.txt
  
  RUN if [ "x$skip_ds_deps" = "x" ] ; then pip install -r requirements_all_ds.txt ; else echo "Skipping pip install -r requirements_all_ds.txt" ; fi
  
  
  COPY . /app
  COPY --from=frontend-builder /frontend/client/dist /app/client/dist
  RUN chown -R redash /app
  USER redash
  
  ENTRYPOINT ["/app/bin/docker-entrypoint"]
  CMD ["server"]
  ```

  - docker-compose文件

    ```dockerfile
    # This configuration file is for the **development** setup.
    version: '3.2'
    # For a production example please refer to getredash/setup repository on GitHub.
    services:
      server:
        build: .
        command: dev_server
        depends_on:
          - postgres
          - redis
        ports:
          - "5000:5000"
          - "5678:5678"
        volumes:
          - ".:/app"
        environment:
          PYTHONUNBUFFERED: 0
          REDASH_LOG_LEVEL: "INFO"
          REDASH_REDIS_URL: "redis://redis:6379/0"
          REDASH_DATABASE_URL: "postgresql://postgres@postgres/postgres"
          REDASH_RATELIMIT_ENABLED: "false"
          REDASH_MAIL_DEFAULT_SENDER: redash@example.com
          REDASH_MAIL_SERVER: email
      scheduler:
        build: .
        command: dev_scheduler
        volumes:
          - type: bind
            source: .
            target: /app
        depends_on:
          - server
        environment:
          REDASH_REDIS_URL: "redis://redis:6379/0"
          REDASH_MAIL_DEFAULT_SENDER: redash@example.com
          REDASH_MAIL_SERVER: email
      worker:
        build: .
        command: dev_worker
        volumes:
          - type: bind
            source: .
            target: /app
        depends_on:
          - server
        environment:
          PYTHONUNBUFFERED: 0
          REDASH_LOG_LEVEL: "INFO"
          REDASH_REDIS_URL: "redis://redis:6379/0"
          REDASH_DATABASE_URL: "postgresql://postgres@postgres/postgres"
          REDASH_MAIL_DEFAULT_SENDER: redash@example.com
          REDASH_MAIL_SERVER: email
      redis:
        image: redis:3-alpine
        restart: unless-stopped
      postgres:
        image: postgres:9.5-alpine
        # The following turns the DB into less durable, but gains significant performance improvements for the tests run (x3
        # improvement on my personal machine). We should consider moving this into a dedicated Docker Compose configuration for
        # tests.
        ports:
          - "15432:5432"
        command: "postgres -c fsync=off -c full_page_writes=off -c synchronous_commit=OFF"
        restart: unless-stopped
        environment:
          POSTGRES_HOST_AUTH_METHOD: "trust"
      email:
        image: djfarrelly/maildev
        ports:
          - "1080:80"
        restart: unless-stopped
    ```

    

  - 启动部署

    ```shell
    cd redash
    docker-compose run --rm server create_db
    npm install
    npm run build
    docker-compose up
    ```

    