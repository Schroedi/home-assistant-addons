ARG BUILD_FROM
FROM $BUILD_FROM

#Dependencies
ARG RUNTIME_PACKAGES="\
  # General utils
  curl \
  # Docker specific
  gosu \
  # Timezones support
  tzdata \
  # fonts for text file thumbnail generation
  fonts-liberation \
  gettext \
  ghostscript \
  gnupg \
  icc-profiles-free \
  imagemagick \
  # Image processing
  liblept5 \
  liblcms2-2 \
  libtiff5 \
  libfreetype6 \
  libwebp6 \
  libopenjp2-7 \
  libimagequant0 \
  libraqm0 \
  libjpeg62-turbo \
  # PostgreSQL
  libpq5 \
  postgresql-client \
  # MySQL / MariaDB
  mariadb-client \
  # For Numpy
  libatlas3-base \
  # OCRmyPDF dependencies
  tesseract-ocr \
  tesseract-ocr-eng \
  tesseract-ocr-deu \
  tesseract-ocr-fra \
  tesseract-ocr-ita \
  tesseract-ocr-spa \
  unpaper \
  pngquant \
  # pikepdf / qpdf
  jbig2dec \
  libxml2 \
  libxslt1.1 \
  libgnutls30 \
  # Mime type detection
  file \
  libmagic1 \
  media-types \
  zlib1g \
  # Barcode splitter
  libzbar0 \
  poppler-utils \
  # RapidFuzz on armv7
  libatomic1 \
  # for Homeassistant
  build-essential \
  libjpeg62-turbo-dev \
  libmariadb-dev \
  libpq-dev \
  p7zip-full \
  python3-dev \
  python3-pip \
  python3-scipy \
  sudo \
  unzip \
  wget"

# Install basic runtime packages.
# These change very infrequently
RUN set -eux \
  echo "Installing system packages" \
    && apt-get update \
    && apt-get install --yes --quiet --no-install-recommends ${RUNTIME_PACKAGES} \
  && echo "Installing supervisor" \
    && python3 -m pip install --default-timeout=1000 --upgrade --no-cache-dir supervisor==4.2.5 \
    && mkdir /var/log/supervisord /var/run/supervisord \
  && echo "Installing NLTK data" \
    && python3 -m pip install --default-timeout=1000 --upgrade --no-cache-dir nltk \
    && python3 -W ignore::RuntimeWarning -m nltk.downloader -d "/usr/share/nltk_data" snowball_data \
    && python3 -W ignore::RuntimeWarning -m nltk.downloader -d "/usr/share/nltk_data" stopwords \
    && python3 -W ignore::RuntimeWarning -m nltk.downloader -d "/usr/share/nltk_data" punkt


RUN wget -O paperless.tar.xz https://github.com/paperless-ngx/paperless-ngx/releases/download/v1.13.0/paperless-ngx-v1.13.0.tar.xz \
    && 7z x paperless.tar.xz \
    && 7z x paperless.tar -o/usr/src/ \
    && cd /usr/src 

# Fix aarch64 builds
# see: https://github.com/pikepdf/pikepdf/issues/194#issuecomment-1020419586
RUN wget -O qpdf.tar.gz https://github.com/qpdf/qpdf/releases/download/release-qpdf-10.6.3/qpdf-10.6.3.tar.gz \
    && tar zxf qpdf.tar.gz \
    && mv qpdf-10.6.3 qpdf \
    && ls -lah \
    && cd /qpdf \
    && ./configure \
    && make

WORKDIR /usr/src/paperless-ngx/

# We need to make the directory before coping to it
RUN mkdir -p /usr/src/paperless/src
RUN cp requirements.txt /usr/src/paperless/requirements.txt

# copy app
RUN cp -R ./src /usr/src/paperless/

# Install requirements
RUN python3 -m pip install --upgrade pip supervisor setuptools \
    && env QPDF_SOURCE_TREE=/qpdf python3 -m pip install --no-cache-dir -r requirements.txt

#### Redis Setup
RUN apt-get install --yes --quiet --no-install-recommends redis-server redis-tools
RUN sed -i "s/bind .*/bind 127.0.0.1/g" /etc/redis/redis.conf

# Clean Up
RUN apt-get -y purge build-essential libqpdf-dev \
    && apt-get -y autoremove --purge \
    && rm -rf /var/lib/apt/lists/* 

# copy scripts
# FROM ORIGINAL DOCKER FILE: this fixes issues with imagemagick and PDF
RUN cp docker/imagemagick-policy.xml /etc/ImageMagick-6/policy.xml \
    && cp gunicorn.conf.py /usr/src/paperless \
    && cp docker/supervisord.conf /etc/supervisord.conf \
    && cp docker/wait-for-redis.py /sbin/wait-for-redis.py \
    && chmod 755 /sbin/wait-for-redis.py \
    && cp docker/env-from-file.sh /sbin/env-from-file.sh \
    && chmod 755 /sbin/env-from-file.sh \
    && cp docker/paperless_cmd.sh /usr/local/bin/paperless_cmd.sh \
    && chmod 755 /usr/local/bin/paperless_cmd.sh \
    && cp docker/flower-conditional.sh /usr/local/bin/flower-conditional.sh \
    && chmod 755 /usr/local/bin/flower-conditional.sh \
    && cp docker/docker-entrypoint.sh /sbin/docker-entrypoint.sh
COPY scripts/docker-entrypoint.sh /sbin/docker-entrypoint.sh
COPY scripts/docker-prepare.sh /sbin/docker-prepare.sh


WORKDIR /usr/src/paperless

# add users, setup scripts
RUN addgroup --gid 1000 paperless \
    && useradd --uid 1000 --gid paperless --home-dir /usr/src/paperless paperless \
    && chown -R paperless:paperless . \
    && chmod 755 /sbin/docker-entrypoint.sh

WORKDIR /usr/src/paperless/src/

ENV PAPERLESS_REDIS="redis://localhost:6379"

COPY scripts/wait-for-redis.sh scripts/wait-for-redis.sh
RUN sudo chmod +x scripts/wait-for-redis.sh

RUN sudo -HEu paperless python3 manage.py collectstatic --clear --no-input

RUN sudo -HEu paperless python3 manage.py compilemessages


VOLUME ["/usr/src/paperless/data", "/usr/src/paperless/media", "/usr/src/paperless/consume", "/usr/src/paperless/export"]

COPY scripts/setup_superuser.py scripts/setup_superuser.py
ENTRYPOINT ["/bin/bash", "/sbin/docker-entrypoint.sh"]
EXPOSE 8000


CMD ["/usr/local/bin/supervisord", "-c", "/etc/supervisord.conf"]

LABEL maintainer="Moshe <34072688+TheBestMoshe@users.noreply.github.com>"
